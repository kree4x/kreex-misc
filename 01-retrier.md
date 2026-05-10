## 零零碎碎的稀奇古怪之Retrier

> Created By [RV](mailto:rodney.vin@gmail.com), and licensed with Creative Commons "[CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/)"

### 失败了，总要再试试

失败，这个事情，它是很难避免的。

失败本身并不重要，怎么处理失败才最重要。

失败一次就立即放弃，这只在需要Fast Fail的场景下才合理。而通常来讲，失败以后，再试试，说不定下次就成功了。

在一个分布式系统中，我们可能追求运行态3个9，甚至5个9、6个9的可靠性。

但是，在设计和开发阶段，我们首先要考虑的是墨菲定律。

一个失败，只要它可能发生，它就一定会发生。

所以，在设计和开发阶段，我们的预期是倒置的：失败是常态的，正常才是不正常的。

所以，脑子里面时刻要绷紧一根弦，实现一个功能时，时时要问自己：

会失败么？需要Fast Fail，还是需要再试试？😁

**BTW:**

KreeX的动态Tracing，是基于Retrier的。

如果你开启了重试，当第一次服务调用失败，在最后一次重试服务调用时，会自动打开一次性的traceEnabled标签，将当次完整的通信链路及服务调用栈跟踪信息输出。

### 合适比什么都重要

可以做重试的库，很多。

号称轻量的，号称功能完备的，号称解耦的，号称为特定场景优化的……

十年前，需要一个库来**“重试”**的时候，翻来翻去很久，最终决定手搓一个。

十年后，又翻来翻去很久，最终无奈，还是不得不再搓一遍。

**核心的问题：**

主流的库核心定位是**“故障处理”**，而我们的核心定位是**“任务调度”**。

### 设计目标和约束

需要满足的要求和场景其实很简单：

- 支持同步、异步任务，支持async、promise
- 三种任务模式：普通重试、Always、Forever
- 链式调用，流式接口
- 可插拔、支持抖动的退避策略
  - 固定间隔
  - 线性退避
  - 指数退避
  - Shuttle
  - 自定义
  - 成功后，自动重置退避策略
- 重试器是**"对象"**，而不是~~**函数**~~，通过重试器对象提供控制接口
- 熔断控制
  - 超时：总任务、单次任务
  - 次数：达次关停
  - 重试任务内，根据业务逻辑调整、中断重试
- 运行时动态干预
  - 重试任务可反向控制“重试”行为
  - 可动态启停，随时可stop
  - 可随时踢醒，在下次重试时间未到时，可以随时强制立即执行
- 自动报告生命周期动态

如此简单，对吧。👈🏻

### **Retrier，名字很土**

相信，名字就是力量。

需要一个重试器对象，想了很久，没找到合适的、别的名字，所以把它命名为“Retrier”。

土。

### 链式调用

链式，或者说流式的API设计是很优雅的，我们使用它可以用近乎自然语言的方式描述重试策略。

```javascript
Retrier.times(5) // max retry: 5 times
  .min(100) // min interval, 100ms
  .max(300) // max interval, 300ms
  .fixedIncrease(20) // each call, increase interval by 20ms
	.task((currentRetries, latency) => {}) //
  .start()
```

### 退避策略

退避策略，Backoff Policy，其核心功能，是用来计算下一次重试动作的延迟时间。

当然为了避免惊群效应，一般来讲，会要求退避策略支持抖动(Jitter)这个参数。

业界主流的库，往往只支持指数退避一种策略，其核心逻辑是：“**绝对规律的数字，导致绝对的系统崩溃**”。

但是，那只是解决了战术层面的问题。

“请求”发生在什么时间，这是不可改变的。全局上，大量请求失败时，对应的应该是**“熔断”**。所有的退避、不论是什么算法，都是治标的。

想开了这一点，我们就不会纠结了。 天生我才必有用，每种算法都有适合它场景。怎么用才合适，是用户才能取舍的事情。

库内，內建了下述几种退避策略：

#### 固定退避：FixedIntervalPolicy

固定退避时，一次重试任务结束后，开始下一次重试的时间间隔是固定的，同时支持Jitter。

```javascript
Retrier.fixedBackoff(fixedInterval, jitter = 500) // 固定时间间隔，随机Jitter扰动
Retrier.fixedInterval(fixedInterval) // 固定时间间隔，Jitter=0
```

#### 线性退避：LinearBackoffPolicy

线性退避时，时间间隔每次增加一个固定值。一次重试任务结束后，开始下一次重试的时间间隔**+=increasement**，同时支持Jitter。

```javascript
Retrier.linearBackoff (increasement, jitter = 500)  // 每次增加increasement，随机Jitter扰动
Retrier.fixedIncrease (increasement) // 每次增加increasement，Jitter=0
```

#### 指数退避：ExponentialBackoffPolicy

指数退避时，实际是因数退避，时间间隔每次乘以一个因数。一次重试任务结束后，开始下一次重试的时间间隔***=Factor**，同时支持Jitter。

我把因数固定为了2，不允许指定。周知一下就好。

```javascript
Retrier.linearBackoff (jitter = 500)  // Factor =2, interval*=2，每次时间加倍，随机Jitter扰动
Retrier.factorIncrease (factor)// 因数增长，Jitter=0
```

### 回摆策略：ShuttlePolicy

这个，是我自己加的。别的地没见过。

用于在一个资源不可用时，多次检测其可用性。此种场景下，固定、线性、指数都不合适，每次检查的间隔逐渐增加，达峰后逐渐减小，从而在min和max之间摆动。

简单讲，它本质是固定增长策略。只是在重试时，到达Interval的min、max边界后，increasement(stepLength)会自动乘以-1。

```javascript
Retrier.shuttleInterval (stepLength, jitter = 500) // 每次变动的步长
```

### 超时控制

只要是**“任务”**，就必须给一个Deadline。设定任务，不给timeout控制，除非你明确知道，但是“**故意**”就是你的目的，否则后果就是“放羊”。

Retrier两个层面的超时时间控制：总任务、单次任务

#### 总任务超时

设定总超时间后，重试耗费的时间达到此限制后，不论是否达到最大重试次数，重试都会被标记为失败。

```javascript
Retrier.timeout(timeout) // 设定总超时时间
```

#### 单次超时

单次任务卡死，重试次数不再增加，达到总任务超时时间后，任务失败。这是一种滑稽的场景。

估算单次任务的耗时，为其设定一个合理的超时时间，才能保证重试如我们期望的方式不断地进行。

```javascript
Retrier.taskTimeout(timeout) // 设定单次任务超时时间
```

### Task: 设定要执行的任务

使用task()方法，可以设定需要重试的任务。

```javascript
Retrier.task((currentRetries, latency) => {}) // 设定重试任务
```

**任务函数**

currentRetries: 当前任务是第几次重试

latency: 当前任务启动时，距离Retrier.start()的潜伏期是多少ms

```typescript
(currentRetries:number, latency:number):any|Promise<any> 
```

**何时认为失败**

- 抛出异常
- 返回Rejected Promise

### 普通重试

使用task((currentRetries, latency) => {})方法，设定的是普通重试任务。

其普通于：

- 遵守times最大重试次数限制
- 遵守timeout总任务超时
- 失败了，等待下一次重试
- 成功了，整个Rerier任务完成，退出执行。

### Always：不论成功失败，再来一次，直至限制

使用always((currentRetries, latency) => {})方法，设定的是always重试任务

其行为：

- 遵守times最大重试次数限制
- 遵守timeout总任务超时
- 不论成功失败，一定等待下一次重试

### Forever：永远执行，天长地久

使用forever((currentRetries, latency) => {})方法，设定的是Forever重试任务

其行为：

- infinite，忽略times最大重试次数限制
- noTimeout，忽略timeout总任务超时
- 不论成功失败，永远执行下一次任务

### start-stop：随时可以启停

重试必须随时可以启停。

这是，要求重试必须是一个Rerier的最重要的原因。

Retrier对象，为我们提供了运行时动态控制Rerier的抓手和api门面。

**!!!注意!!!**

stop()并不能强制正在运行的单次Task立即终止。

stop()只会在当次正在执行的任务运行结束后，终止下一次的重试企图。

所以，taskTimeout很重要。它提供了一种预期管理，至少到了taskTimeout时间，它会被停下来。

### wakeup：随时踢醒装睡者

如果一次重试结束，在下一次重试开始前，Retrier就进入了休眠期。

wakeup()方法，提供了将Retrier从休眠中踢醒，立即开始下一次重试的能力。

```javascript
retrier.wakeup() // 将retrier从休眠中踢醒
```

这个方法很重要。

KreeX中，我们将Retrier用于任务队列的Producer/Consumer模式管理。

队列的问题麻烦。暴力轮询，浪费资源，长时等待，处理延迟高。

我们在Consumer内部维持了一个Retrier，使用较长的fixedInterval策略定期从任务队列中拉取任务。如果无任务、或者本次处理完毕会进入休眠。

Producer产生任务，向任务队列压栈后，会立即wakeup() Consumer，通知其开始处理。

如此，既节省资源又延迟极低。

### EventEmitter：监听生命周期

Retrier是一个EventEmitter，它会emit下述事件。

```typescript
export const Start = 'start' // retry started
export const Stop = 'stop' // retry stopped
export const Retry = 'retry' // one retry began
export const Success = 'success' // one task running succeeded
export const Failure = 'failure' // one task ran failed
export const Timeout = 'timeout' // total timeout
export const TaskTimeout = 'task-timeout' // one task timed out
export const Completed = 'complete' // all retries completed
export const MaxRetries = 'max-retries' // Reach the max retries
```

事件清单，列在这里了。

怎么使用，就是你的事情了。结合“危险行为清单”，还是能玩不少事情。😁

### 危险行为清单

在你的task()函数内，是**"可以"**动态改变Retrier的行为的。捂脸，🤦‍♀️。

这是最初的设计目标之一，但是实现及测试时，各种情况的交叉矩阵太多复杂。

所以，你要这么做，我没有阻止。

但是，会发生什么不可预期行为的话，欢迎提issue。

