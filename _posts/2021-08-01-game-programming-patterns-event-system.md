---
layout: post
title:  "游戏编程模式-事件系统"
date:   2021-08-01 10:03:03 +0800
categories: jekyll update
---


## 什么是事件系统
为了降低游戏或其它软件中对象之间的耦合性，开发者会采用一个名为事件的对象将功能调用者与功能执行者隔离开来。调用者无需知道该功能是否被执行、被谁执行，执行者无需关心调用者的具体身份。调用者只需要关心事件是否正确生成。执行者只需要关心事件是否正确处理。

事件系统是发布-订阅模式的典型实现。

## 事件系统设计
C# event 在语言层面提供了完备的发布、订阅事件机制。

而在实际的游戏开发工作中，c# event 所提供的发布订阅功能难免过于基础，项目组更倾向于自己实现一套发布订阅事件模型，引入项目相关的特性，以更好地完成项目需求。一个完备的游戏事件发布订阅系统需要解决的问题主要有：
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