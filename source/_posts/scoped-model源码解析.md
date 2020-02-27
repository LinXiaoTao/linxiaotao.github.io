---
title: scoped_model源码解析
date: 2019-05-06 22:13:54
categories: Flutter
tags:
---

### 参考

* [ephemeral-vs-app](https://flutter.dev/docs/development/data-and-backend/state-mgmt/ephemeral-vs-app)
* [inheritedwidget-inheritedmodel]([http://juju.one/inheritedwidget-inheritedmodel/](http://juju.one/inheritedwidget-inheritedmodel/)



[scoped_model](https://pub.dartlang.org/packages/scoped_model) 是 Google 推荐使用的应用状态管理库（Simple app state management），当然，除了它，还有其他选择，比如 Redux，Rx，hooks 等等。

### 临时状态和应用状态

我们说 scoped_model 用于**应用状态**管理，是因为除了**应用状态（app state）**以外，还有临时状态（ephemeral state）。

#### 临时状态

也可以称为 UI 状态或者本地状态，是一种可以包含在单个 widget 的状态。

比如以下定义：

* `PageView` 当前的页面
* 动画当前的进度
* `BottomNavigationBar` 当前选中项

使用临时状态，我们不需要使用状态管理，只需要一个 `StatefulWidget`。

#### 应用状态

需要在应用中多个组件之间共享，而且在不同的用户会话之间保持，这就是应用状态，有时候也称为共享状态。

比如以下定义：

* 用户偏好设置
* 登录信息
* 通知栏功能
* 购物车功能
* 已读未读功能



临时状态和应用状态之间没有非常明确的界限，比如你可能需要持久化保存 `BottomNavigationBar` 的选中项，那它就不是临时状态了，而是应用状态。如果你愿意，你甚至可以不用任何状态管理库，只使用 `State` 和 `setState()` 来管理应用所有的状态。

### scoped_model

scoped_model 由三个部分组成，`ScopedModel`、`ScopedModelDescendant` 和 `Model`。

#### Model

`Model` 是定义一个数据模型的基类，它继承了 `Listenable`，提供了 `notifyListeners()` 方法来通知组件需要更新。

``` dart
@protected                                                                  
void notifyListeners() {                                                    
  // We schedule a microtask to debounce multiple changes that can occur    
  // all at once.                     
  if (_microtaskVersion == _version) {
    // 去抖
    _microtaskVersion++; 
    // 安排一个 Microtask 任务
    scheduleMicrotask(() {                                                  
      _version++;                                                           
      _microtaskVersion = _version;                                         
                                                                            
      // Convert the Set to a List before executing each listener. This     
      // prevents errors that can arise if a listener removes itself during 
      // invocation!                                                        
      _listeners.toList().forEach((VoidCallback listener) => listener());   
    });                                                                     
  }                                                                         
}                                                                           
```

> 关于 Microtask：这涉及到 dart 的单线程模型，这里我们只需要知道，它的优先级比 event 要高，会先执行。
>
> 关于更多信息，可以查看 [event-loop](https://webdev.dartlang.org/articles/performance/event-loop#darts-event-loop-and-queues)

#### ScopedModel

在定义一个 `Model` 后，我们需要再定义一个 `ScopedModel` 作为 root widget，它有两个参数，一个是 `model` 即我们上面定义的 `Model` 实例，另外一个是 `child`，则是我们定义的 widget。`ScopedModel` 是一个 `StetelessWidget`，我们看下它的 `build()` 方法：

``` dart
@override                                                                     
Widget build(BuildContext context) {                                          
  return AnimatedBuilder(                                                     
    animation: model,                                                         
    builder: (context, _) => _InheritedModel<T>(model: model, child: child),  
  );                                                                          
}                                                                             
```

`AnimatedBuilder` 是一个用于构建动画的 widget，它的 `animation` 参数是一个 `Listenable` 类：

``` dart
abstract class Listenable {                                                          
  /// Abstract const constructor. This constructor enables subclasses to provide     
  /// const constructors so that they can be used in const expressions.              
  const Listenable();                                                                
                                                                                     
  /// Return a [Listenable] that triggers when any of the given [Listenable]s        
  /// themselves trigger.                                                            
  ///                                                                                
  /// The list must not be changed after this method has been called. Doing so       
  /// will lead to memory leaks or exceptions.                                       
  ///                                                                                
  /// The list may contain nulls; they are ignored.                                  
  factory Listenable.merge(List<Listenable> listenables) = _MergingListenable;       
                                                                                     
  /// Register a closure to be called when the object notifies its listeners.        
  void addListener(VoidCallback listener);                                           
                                                                                     
  /// Remove a previously registered closure from the list of closures that the      
  /// object notifies.                                                               
  void removeListener(VoidCallback listener);                                        
}                                                                                    
```

`AnimatedBuilder` 会调用 `addListener()` 方法添加一个监听者，然后调用 `setState()` 方法进行 rebuild。从上面可知，`Model` 继承 `Listenable` 类。这也是为什么在修改值后需要调用 `notifyListeners()` 的原因。

再看下 `builder` 参数，它实际上返回了一个 `_InheritedModel` 实例：

``` dart
class _InheritedModel<T extends Model> extends InheritedWidget {     
  final T model;                                                     
  final int version;                                                 
                                                                     
  _InheritedModel({Key key, Widget child, T model})                  
      : this.model = model,                                          
        this.version = model._version,                               
        super(key: key, child: child);                               
                                                                     
  @override                                                          
  bool updateShouldNotify(_InheritedModel<T> oldWidget) =>           
      (oldWidget.version != version);                                
}                                                                    
```

`InheritedWidget` 是 `scoped_model` 的核心。

##### InheritedWidget

`InheritedWidget` 可以在组件树中有效的传递和共享数据。将 `InheritedWidget` 作为 root widget，child widget 可以通过 `inheritFromWidgetOfExactType()` 方法返回距离它最近的 `InheritedWidget` 实例，同时也将它注册到 `InheritedWidget` 中，当 `InheritedWidget` 的数据发生变化时，child widget 也会随之 rebuild。

当 `InheritedWidget` rebuild 时，会调用 `updateShouldNotify()` 方法来决定是否重建 child widget。

继续看 `ScopedModel`，它使用 `version` 来判断是否需要通知 child widget 更新：

``` dart
@override                                                          
bool updateShouldNotify(_InheritedModel<T> oldWidget) =>           
      (oldWidget.version != version); 
```

当我们调用 `Model` 的 `notifyListeners()` 方法时，`version` 就会自增。

#### ScopedModelDescendant

`ScopedModelDescendant` 是一个工具类，用于获取指定类型的 `Model`，当 `Model` 更新时，会重新执行 `build()` 方法：

``` dart
@override                                                            
Widget build(BuildContext context) {                                 
  return builder(                                                    
    context,                                                         
    child,                                                           
    ScopedModel.of<T>(context, rebuildOnChange: rebuildOnChange),    
  );                                                                 
}


static T of<T extends Model>(                       
  BuildContext context, {                           
  bool rebuildOnChange = false,                     
}) {                                                
  final Type type = _type<_InheritedModel<T>>();    
  
  // 最终也是使用 inheritFromWidgetOfExactType 或 ancestorWidgetOfExactType
  Widget widget = rebuildOnChange                   
      ? context.inheritFromWidgetOfExactType(type)  
      : context.ancestorWidgetOfExactType(type);    
                                                    
  if (widget == null) {                             
    throw new ScopedModelError();                   
  } else {                                          
    return (widget as _InheritedModel<T>).model;    
  }                                                 
}                                                   
                                                    
static Type _type<T>() => T;                        
```

注意到，在调用 `ScopedModel.of()` 方法时，有个 `rebuildOnChange` 参数，表示当 `Model` 更新时，是否需要 rebuild。当设置为 `false` 时，会使用 `ancestorWidgetOfExactType()` 方法去获取最近的 `InheritedWidget`，和 `inheritFromWidgetOfExactType()` 方法的区别是，`inheritFromWidgetOfExactType` 在获取的同时会注册到 `InheritedWidget` 上。

### 总结

使用 `scoped_model`，我们首先定义一个 `Model`，这里面封装了对数据的操作，需要注意，数据改变后需要调用 `notifyListeners()` 方法。接着再将 `ScopedModel` 作为 root widget，传递一个 `Model` 实例，最后我们可以使用 `ScopedModelDescendant` 来响应数据的修改，也可以手动调用 `ScopedModel.of()` 方法来获取 `Model` 实例，调用这个方法，如果参数 `rebuildOnChange` 传递为 `true`，则同时会将当前 widget 注册到 `InheritedWidget` 上，当数据改变时，当前 widget 会重新构建。