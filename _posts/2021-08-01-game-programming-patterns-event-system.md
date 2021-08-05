---
layout: post
title:  "游戏编程模式-事件系统"
date:   2021-08-01 10:03:03 +0800
categories: jekyll update
---


## 什么是事件系统
为了降低游戏或其它软件中对象之间的耦合性，开发者会采用一个名为事件的对象将功能调用者与功能执行者隔离开来。调用者无需知道该功能是否被执行、被谁执行，执行者无需关心调用者的具体身份。调用者只需要关心事件是否正确生成。执行者只需要关心事件是否正确处理。

事件系统是发布-订阅模式的典型实现。

## 事件系统需求
C# 多播委托在语言层面提供了完备的发布、订阅事件机制。

而在实际的游戏开发工作中，c# 多播委托所提供的发布订阅功能难免过于基础，项目组更倾向于自己实现一套发布订阅事件模型，引入项目相关的特性，以更好地完成项目需求。一个完备的游戏事件发布订阅系统需要解决的问题主要有：
- 事件订阅、反订阅
  - 游戏对象销毁时确保相关反订阅执行
  - 尽可能缩减事件字典大小
- 事件发布、调用
  - 尽可能不引入GC、不向下转型
  - 主线程事件应同步发布，订阅者调用堆栈可以追溯到发布者
  - 支持多线程异步事件
- 事件系统可扩展性
  - 新增事件类型不需要对事件系统本身做事件类型定义之外的任何修改
  - 
- 事件系统易用性
  - 事件系统使用者无序关系其实现细节，只需要关注自己定义的时间类型并执行订阅、反订阅、发布

## 事件订阅
游戏对象，如场景物体、UI等，在销毁时需要反订阅事件。与WinForm不同，form类在其生命周期内可以始终持有窗口内各个控件对象，因此可以将控件的事件响应写成form类的方法，通过委托就足以完成事件处理。在游戏程序中，游戏对象的动态创建和销毁更为普遍，在对象销毁时，如果其订阅的事件未反订阅，那么事件发布时就会因为引用了已销毁的对象产生异常或其它非预期行为。

为进行事件反订阅，需要在事件里建立订阅对象与订阅方法的映射。Delegate Target属性提供了实例方法委托的实例，即所指向非静态方法的宿主对象，当游戏对象为事件订阅一个对象自身的非静态方法时，通过 Delegate Target 自然地建立了订阅对象与订阅方法的映射。但是游戏程序中也难免会遇到订阅静态方法，例如对象对事件的处理仅仅是打印日志，此时开发者只能显式提供订阅对象而无法依赖 Delegate Target 属性。相应地，事件系统也应该显式维护订阅对象与订阅方法的映射。

基于上述原因，本文所设计的事件系统包含一个 object 到 Action<T> 的映射，即订阅对象与订阅犯法的映射：
```csharp
private static readonly Dictionary<object, Action<T>> _handlers = new EventImpl.UserHandlerDict<T>();
```
其中 UserHandlerDict 是为了更好维护映射字典而实现的 Dictionary 派生类。

在进行事件订阅时，要求开发者提供订阅该事件的游戏对象和订阅方法。例如 class Foo 需要通过静态或实例方法 Bar(BarEvent e) 方法用来处理BarEvent事件，并且希望在 Foo 的 Init 方法中订阅 BarEvent。那么事件订阅需要提供Foo的this实例、Foo的Bar方法。本文设计的事件系统将提供如下写法：
```csharp
Event<BarEvent>.Register(this, Bar);
```
或者以lambda表达式订阅:
```csharp
Event<BarEvent>.Register(this, (BarEvent e)=>{...});
```
反订阅写法为：
```csharp
Event<BarEvent>.UnRegister(this);
```

## 事件发布
事件发布只需要广播一个事件对象，例如对于 BarEvent，只需要像这样设计接口：
```csharp
Event<BarEvent>.Dispatch(BarEvent barEvent);
```
但是这个接口存在问题。
- 使用者必须自行构造一个BarEvent对象并传参
- 使用者如果不缓存barEvent, 则每次事件发布都将产生GC
在相当多的情况下，事件发布与订阅者回调是同步执行的，即Dispatch返回之前会执行所有的订阅委托，不需要使用队列缓存事件对象，不需要考虑消息队列模型，每个发布过程都只存在一个消息实例。如此一来 Event<T> 可以设计为单例。
```csharp
public class Event<T> where T : new()
{
    static T instance = new T();  
}
```
与典型的单例模式写法不同，没有必要获取实例对象，它只是一个缓存，用来避免GC。

Dispatch方法的设计由直接传参改写为传入构造委托：
```csharp
public class Event<T> where T : new()
{
    static T instance = new T();
    public static void Dispatch(Action<T> init)
    {
        init(instance);
    }
}
```
```csharp
Event<BarEvent>.Dispatch(barEvent=>{
  barEvent.param = my_bar_event_param;
});
```
## 其它设计方案对比
在一些项目中，Event 对象拥有一个公共基类，并使用多级字典来维护事件注册，例如：
```csharp
Dictionary<int, Dictionary<object, Action<EventBase>>> eventDict
```
其中外层字典int key表示事件类型id，内层字典object key 和 Action<EventBase> value 分别是事件订阅者对象和订阅委托。
事件订阅时所提供的方法声明须使用统一形参EventBase， 并在方法体中奖 EventBase 向下转型为业务所需的事件类型，例如：
```csharp
private void OnBarEvent(EventBase evt)
{
    if(evt is BarEvent barEvent)
    {
        //process barEvent
    }
}

//接口调用：
{
    Event.Dispatch(EventType.BAR_EVENT, barEvent);
    Event.Register(EventType.BAR_EVENT, this, OnBarEvent);
    Event.UnRegister(EventType.BAR_EVENT, this);
}
```
可以看到这种写法对外提供的接口相当简洁。除了向下转型和字典访问开销外，是一个相当实用的实现方式。

在某些项目中，Event 能够反注册单个委托：
```csharp
// some class, a Unity MonoBehavior class for example
{
    private void Awake()
    {
        Event.Register(EventType.BAR_EVENT, GetDelegateOf_OnBarEvent);
    }
    private void OnDestroy()
    {
        Event.UnRegister(EventType.BAR_EVENT, GetDelegateOf_OnBarEvent);
    }
    private Action<EventBase> delegateOf_OnBarEvent;
    private Action<EventBase> GetDelegateOf_OnBarEvent()
    {
        return delegateOf_OnBarEvent??(delegateOf_OnBarEvent = OnBarEvent);
    }
    private void OnBarEvent(EventBase evt)
    {
        if(evt is BarEvent barEvent)
        {
            //process barEvent
        }
    } 
}
// 事件发布
{
    Event.Dispatch(EventType.BAR_EVENT, barEvent);
}
```
笔者当前参与的项目中正是这种方式。除了能够精确进行单个事件的订阅管理，它真的是丑。不得不让人怀疑是项目进行到一半才发现注册和反注册时直接传入方法名只是个语法糖，由于无法正确反注册匿名的委托对象，只好临时加了个丑到爆炸的显示委托对象声明。



## 事件系统实现
完整的事件系统实现参见 https://gitee.com/isncg/gpw/blob/stg/scripts/utils/Event.cs ， 以下为部分代码
```csharp
public class Event<T> where T : Event<T>, new()
{
    private static readonly Dictionary<object, Action<T>> _handlers = new EventImpl.UserHandlerDict<T>();
    static T instance = new T();
    static bool isBusy = false;// prevent recursive dispatch call
    public virtual void InitContent() { }
    public static void Dispatch(Action<T> init)
    {
        try
        {
            if (isBusy) return;
            isBusy = true;
            instance.InitContent();
            init(instance);
            foreach (var handler in _handlers.Values)
                handler?.Invoke(instance);
            isBusy = false;
        }
        catch (Exception e)
        {
            Log.E("[Event:Dispatch] exception when dispatch\n{0}", e.Message);
            isBusy = false;
        }
    }
    public static void Register(object user, Action<T> handler) { _handlers[user] = handler; }
    public static void UnRegister(object user) { _handlers.Remove(user); }
}
```