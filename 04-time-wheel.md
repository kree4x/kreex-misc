## 零零碎碎的稀奇古怪之时间轮：TimeWheel

> Created By [RV](mailto:rodney.vin@gmail.com), and licensed with Creative Commons "[CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/)"

### 遇到的问题

在没有原生Promise的时代，我们曾使用process.nextTick()和setTimeout()在NodeJS及浏览器中手工创建Promise Like的结构。

它可以工作。但是，坑就在那里。

曾经迷恋过使用Promise来修正Callback地狱问题。倒金字塔结构消失不见，Promise带来了代码形式上的规范和规整，很养眼。

仅仅是养眼，如此而已，但是带来的最大的问题是空耗。

在遇到不可思议的回调触发延迟后，使用分析工具看了下内存分析，然后被惊到了。

绝大部分的调用都是nextTick和setTimeout本身的空耗，滥用异步调度原语（尤其是 Timer）所带来的调度开销，往往远高于任务本身。

于是，一个强制的编码规范是：可以不async/promise的时候，禁止异步，禁止Promise，特别是在高流量的热路径上。

**问题的本质**：

如果有大量的定时任务，为每个任务创建一个Timer，这绝对是最差的架构设计，它会带来在Coding层面无法解决的性能问题。

### 更多的场景

需要使用到Timer的场景很多，KreeX必须解决的，是带超时控制的任务队列问题。例如：待发DataPack队列，接收到的待组装的DataFrame队列。

我们无法使用CappedList之类的基于容量的滑动窗口设计，因为我们不能拒绝一个待发任务，也不能拒绝接受入站的数据包，更不能在处理完成前，直接将他们丢弃。

我们需要的是基于时间的滑动窗口设计：每个DataPack、DataFrame都带有timeout属性，在其timeout时，直接将其丢弃，这在业务层面是完全正确的。

为每个DataPack、DataFrame创建一个Timer去做超时控制，这个思路很简单，却是一条死路。

**Kree4N**(KreeX For NodeJS)中，我们的极限分帧测试，一个byte一个DataFrame的发送。

以此种规模使用setTimeout，整个JS的事件循环就崩了。

时间轮算法，是解决这种“**海量定时任务的高效调度与管理**”问题的完美算法。

### 时间轮：TimeWheel

**时间轮的核心是**：

- 任务归簇：把同一个触发时间的任务，归纳为一个簇，按簇触发。
- 一个Timer：一个时间轮，一个Timer(setTimeout/setInterval)
- 划分刻度：时间轮有自己的精度，也就是最小刻度，setTimeout/setInterval时的那个时间值。

直观地将一个机械手表的表盘想象为一个时间轮就好：

- 最小刻度是1s
- 总共60刻度
- 秒针指向当前刻度

实际上，TimeWheel这个命名也来源于此。

- Tick：一个刻度是一个Tick
- Slot：一个Tick对应一个Slot，Slot存放一个任务簇
- Cursor：当前Tick是多少？

所以其实现很简单。

**创建一个TimeWheel**

指定一个tick是多少，一个Wheel总共允许多少个tick，这就够了。

```javascript
const timeWheel = new TimeWheelCache({
      autoStart: false, // 不自动开始计时
      tickInterval: 1000, // ms
      tickCount: 60 // 最多60个tick
})
```

**加入一个任务**

```javascript
timeWheel.set(key1, value1, 55000) // ttl = 55s
```

**监听超时**

```javascript
timeWheel.on('expired', (key, value) => {
  // do what you want
})
```

### **分层时间轮：**Hierarchical Timing Wheels

你会发现，上述的秒盘，60个slot是合理的。

如果是一天呢，60 * 60 * 24 = 86400，创建一个86400个slots，这技术上可行，但是工程上不太可以接受，😊。

那如果是一个月、一年内？

分层时间论，正是为了解决这个问题。

它的核心是多个时间轮聚合，然后，逐层自动降级。

以24小时的时间轮为例：

- 天轮：一个tick一小时， 24个Slot
- 时轮：一个tick一分钟，60个Slot
- 分轮：一个tick一秒钟，60个Slot

所以：

- 天轮中超时的数据，自动压入时轮，根据剩余 TTL 重新计算目标 Slot。
- 时轮中超时的数据，自动压入分论，根据剩余 TTL 重新计算目标 Slot。

你要追求极致，可以使用单个 Timer 驱动整个分层时间轮，这也是分层时间轮的标准形态。但是这没必要，因为这代表要独立于TimeWheel类，完全重写分层时间轮的算法。

将24小时的时间轮作为一个聚合对象，内部直接聚合3个TimeWheel实例，是实现起来最简单、架构最优美、维护性最好的方案。

最终使用3个Timer，外加24 + 60 + 60 = 144个Slot，解决问题。

### 24小时时间轮：Hour24TimeWheel

我们默认提供了Hour24TimeWheel的实现，也到此为止。

截至目前，KreeX 尚未出现超过 24 小时的定时任务场景。

```javascript
import { Hour24TimeWheelCache } from '@kree4js/commons-collection'
```



