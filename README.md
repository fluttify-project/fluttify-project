## Fluttify是什么以及解决什么问题？
用一句话来说，Fluttify是一个把原生SDK(目前支持Android/iOS)编译成Flutter插件的编译器。

Fluttify解决了Flutter插件开发过程中需要懂原生开发的问题，为原生SDK生成Dart接口，使插件开发者不再需要深入原生语言的细节，直接调用原生接口的Dart Binding即可。

## 编译过程中，Fluttify做了什么事？
1. 为所有原生的公开(public)接口生成其对应的Dart接口。所谓的"接口"包括，public(实例方法，静态方法, 函数, 字段, 常量); 不包括private，protected(实例方法, 静态方法, 字段), 被混淆的(类, 实例方法, 静态方法, 字段)。
2. 为public的视图类(android.view.View/UIView的子类)生成对应的`PlatformView`, 供Dart侧使用。
3. 为1中产生的类增加`is/as`的方法，用于类型判断和造型。

## 输出插件工程的结构
编译器的输出是标准的插件工程，在`lib`文件夹下会有`lib/src/android`和`lib/src/ios`两个文件夹，这两个文件夹分别存放android sdk和ios sdk生成的对应dart类。

这两个文件夹下有会有5种类型的文件，分别是sdk内的类，platform view，常量，函数以及is/as扩展（类型检查及造型），最后会有`android.export.g.dart`和`ios.export.g.dart`导出所有的生成文件。

## 如何使用生出来的插件？
### 创建一个类的对象
一个Java类转为Dart类时，产生的Dart类类名为Java类的类名全称，比如有一个Java类`com.abc.A`，那么生成出来的Dart类为`com_abc_A`。ObjC由于没有命名空间，所以ObjC类转成的Dart类名字是一样的。

编译器会为*有公开构造器*的类生成Dart侧的`create__`方法，`create__`方法会调用Java/ObjC类的对应构造器，如果有多个构造器，那么就会生成多个`create__`方法，根据参数不同在`create__`方法名后面一次添加参数类型名称。比如有一个Java类：
```java
class com.abc.A {
    public A() {}
    public A(String arg) {}
}
```
那么生成的Dart类为：
```dart
class com_abc_A {
    Future<com_abc_A> create__() async { ... }
    Future<com_abc_A> create__String(String arg) async { ... }
}
```

### 调用方法
编译器的目标是让开发者原先怎么调用原生接口，那么就怎么调用编译器生成的Dart接口。所以理论上只需要对原生语句逐句转换即可。比如有这么一段逻辑，摘自[amap_map_fluttify](https://github.com/fluttify-project/amap_map_fluttify/blob/e91f20e6d83c4ededcf136014c8def320a0f7ce1/lib/src/facade/amap_controller.dart#L287)，此段逻辑是设置是否显示室内地图：
原生逻辑：
```java
com.amap.api.maps.AMap amap = mapView.getMap();
amap.showIndoorMap(show);
```
逐句转换为Dart后：
```dart
com_amap_api_maps_AMap map = await mapView.getMap();
await map.showIndoorMap(show);
```
其他的静态方法也好，函数也好，都是一样的逻辑。

### 内存管理
为了能够实现全局获取需要操作的目标对象，原生端创建的对象或从SDK的接口返回的对象，都会被放入一个全局的Map/NSDictionary中，其中key为对象的hash。如果不手动从全局Map/NSDictionary中删除不需要的对象的话，会造成内存泄露。这个Map/NSDictionary下文中称为**HEAP**。

只要是原生端返回给Dart侧的**非可直接传输**的类型（**可直接传输**的类型为String, int等，具体类型列表参考[官方文档](https://flutter.dev/docs/development/platform-integration/platform-channels)，这些类型下文称为**jsonable**类型）原生端会被放入HEAP中，Dart侧会把对应的Dart对象放入一个全局对象`kNativeObjectPool`中，所谓*对应的Dart对象*，其实只是一个持有原生对象hash值的Dart对象，每当通过这个Dart对象调用方法时，都会把这个hash传给原生，原生端通过这个hash去HEAP查找需要操作的原生对象，再在这个原生对象上进行方法调用。

Fluttify提供了一个基础库，即[foundation_fluttify](https://pub.flutter-io.cn/packages/foundation_fluttify)。这个插件提供了系统类的实现，和一些辅助方法。例如`platform`方法，提供了方便操作多端逻辑的方法。调用过程形如：
```dart
await platform(
  android: (pool /* pool参数用于存放需要在方法结束后释放的对象 */) async {
    final map = await androidController.getMap();
    await map.setTrafficEnabled(enable);
    
    // 由于map对象在方法结束后不再使用，所以放入pool对象中，方法结束后会统一释放
    // 如果方法结束后，仍然需要使用的对象，就不要放入pool中，不然后续调用该对象的时候会报空指针异常，因为已经不在原生的HEAP中
    pool.add(map);
  },
  ios: (pool) async {
    await iosController.set_showTraffic(enable);
  },
);
```