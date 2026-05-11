## 零零碎碎的稀奇古怪之EventEmitter：支持Owner分组管理

> Created By [RV](mailto:rodney.vin@gmail.com), and licensed with Creative Commons "[CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/)"

### 发布-订阅，还是观察者模式

NodeJS的EventEmitter经常被称为事件“发布-订阅”。

但实际上，在严格的设计模式分类中，它是一个顶着“发布-订阅”马甲的经典的观察者模式的实现。

发布-订阅模式，要求通过消息代理解耦发布者与订阅者，例如通过消息队列、消息总线……

而EventEmitter中：

Subject：被观察者，EventEmitter对象本身，负责管理事件和监听器。

Observer：观察者，通过on()， addListener()监听事件，注册回调函数

Notify：通知，当emit()被调用时，同步或异步执行Observer注册的回调函数

### EventEmitter很好用

好用到在本地使用它还不够，我还要以EventEmitter的接口和功能为模板和载体，实现DSE，Distributed Service EventEmitter。

在本地，我们通过EventEmitter来支持对一个对象的异步观测，对象状态变更时，触发各种event，而监听者如果对某种event感兴趣，就可以监听event然后被踢醒执行事件回调。

而将EventEmitter开放为一个透明RPC服务后，我们可以以方法调用的形式，实现事件驱动的程序架构。

通过DSE，我们实现了事件驱动的模式，取其精髓，弃其糟粕。解决了事件驱动编程在Coding时不直观、不符合人类直觉的问题。

### EventEmitter还不够好

标准的EventEmitter，少了一项功能：按Owner管理Listener。

标准的EventEmitter是按照事件名EventName类管理Listener的，如果你监听了一个event，当emit(event)时，所有监听该EventName的Listener都会被触发执行。

这是其一：不支持按照分组部分触发。

移除listener的时候，也是按照eventName和listener实例本身，去移除的。这个事情很麻烦，为移除自己注册的listener，我们经常不得不专门定义一个局部变量去保存listener的引用。

这是其二：不支持按照分组移除。

#### 扩充EventEmitter支持Owner

所以，简单的扩充下EventEmitter，支持按照Owner对listener进行分组注册、分组触发、分组移除，会带来巨大的好处。

另外，KreeX必须同时支持NodeJS Server端和Browser端，我们无法直接使用标准的NodeJS EventEmitter实现。这也是，我们不得不实现一个扩展版EventEmitter的原因。

```typescript
export default class EventEmitter {
  addListener(eventName: string | Symbol, listener: Function, owner?: any): this;
  once(eventName: string | Symbol, callback: Function, owner?: any): this;    
  emitOwner(eventName: string | Symbol, owner: any, ...args: any[]): boolean;
  offOwner(owner: any): this;      
}
```

#### KreeX按Owner管理Event及Listener

KreeX中在大规模使用Owner这个Feature简化代码逻辑，提升运行性能：

一个简单的例子。

WhoHas-IHave服务动态发现机制中，我们使用whoHasSignal.id作为Owner，从而在Transport接收到IHaveSignal时，可以精确的只触发指定的一个whoHasSignal的回调，而不是将信号Flood得到处都是。

```javascript
this._transport.observer.onIHaveIn((_inTransportContext, iHaveSignal) => {
      ……
}, whoHasSignal.id)
```

