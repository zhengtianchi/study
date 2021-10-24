# Go 并发读写 sync.map 的强大之处 



map 的两种目前在业界使用的最多的并发支持的模式:

- 原生 map + 互斥锁或读写锁 mutex。
- 标准库 sync.Map（Go1.9及以后）。



## sync.Map 优势

- 多个 goroutine 的并发使用是安全的，不需要额外的锁定或协调控制。
- 大多数代码应该使用原生的 map，而不是单独的锁定或协调控制，以获得更好的类型安全性和维护性。

同时 Map 类型，还针对以下场景进行了性能优化：

- 当一个给定的键的条目只被写入一次但被多次读取时。例如在仅会增长的缓存中，就会有这种业务场景。
- 当多个 goroutines 读取、写入和覆盖不相干的键集合的条目时。

这两种情况与 Go map 搭配单独的 Mutex 或 RWMutex 相比较，使用 Map 类型可以大大减少锁的争夺。



## sync.Map 剖析

### 数据结构

`sync.Map` 类型的底层数据结构如下：

```go
type Map struct {
 mu Mutex                                // 互斥锁，用于保护 read 和 dirty。
 read atomic.Value // readOnly
 dirty map[interface{}]*entry            // 操作 dirty 需要加锁来保证数据安全
 misses int
}

// Map.read 属性实际存储的是 readOnly。
type readOnly struct {
 m       map[interface{}]*entry
 amended bool
}

复制代码
```

- mu：互斥锁，用于保护 read 和 dirty。
- read：只读数据，支持并发读取（atomic.Value 类型）。如果涉及到更新操作，则只需要加锁来保证数据安全。
- read 实际存储的是 readOnly 结构体，内部也是一个原生 map，amended 属性用于标记 read 和 dirty 的数据是否一致。
- dirty：读写数据，是一个原生 map，也就是非线程安全。操作 dirty 需要加锁来保证数据安全。
- misses：统计有多少次读取 read 没有命中。每次 read 中读取失败后，misses 的计数值都会加 1。

在 read 和 dirty 中，都有涉及到的结构体：

```
type entry struct {
 p unsafe.Pointer // *interface{}
}

复制代码
```

其包含一个指针 p, 用于指向用户存储的元素（key）所指向的 value 值。

在此建议你必须搞懂 read、dirty、entry，再往下看，食用效果会更佳，后续会围绕着这几个概念流转。

### 查找过程

划重点，Map 类型本质上是有两个 “map”。一个叫 read、一个叫 dirty，长的也差不多：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3a34f8cbe34f428c93bddfa7d302a2bb~tplv-k3u1fbpfcp-watermark.awebp)

sync.Map 的 2 个 map

当我们从 sync.Map 类型中**读取数据**时，其会先查看 read 中是否包含所需的元素：

- 若有，则通过 atomic 原子操作读取数据并返回。
- 若无，则会判断 `read.readOnly` 中的 amended 属性，他会告诉程序 dirty 是否包含 `read.readOnly.m` 中没有的数据；因此若存在，也就是 amended 为 true，将会进一步到 dirty 中查找数据。

sync.Map 的读操作性能如此之高的原因，就在于存在 read 这一巧妙的设计，其作为一个缓存层，提供了快路径（fast path）的查找。

同时其结合 amended 属性，配套解决了每次读取都涉及锁的问题，实现了读这一个使用场景的高性能。

### 写入过程

我们直接关注 `sync.Map` 类型的 Store 方法，该方法的作用是新增或更新一个元素。

源码如下：

```
func (m *Map) Store(key, value interface{}) {
 read, _ := m.read.Load().(readOnly)
 if e, ok := read.m[key]; ok && e.tryStore(&value) {
  return
 }
  ...
}

复制代码
```

调用 `Load` 方法检查 `m.read` 中是否存在这个元素。若存在，且没有被标记为删除状态，则尝试存储。

若该元素不存在或已经被标记为删除状态，则继续走到下面流程：

```go
func (m *Map) Store(key, value interface{}) {
 ...
 m.mu.Lock()
 read, _ = m.read.Load().(readOnly)
 if e, ok := read.m[key]; ok {
  if e.unexpungeLocked() {
   m.dirty[key] = e
  }
  e.storeLocked(&value)
 } else if e, ok := m.dirty[key]; ok {
  e.storeLocked(&value)
 } else {
  if !read.amended {
   m.dirtyLocked()
   m.read.Store(readOnly{m: read.m, amended: true})
  }
  m.dirty[key] = newEntry(value)
 }
 m.mu.Unlock()
}

复制代码
```

由于已经走到了 dirty 的流程，因此开头就直接调用了 `Lock` 方法**上互斥锁**，保证数据安全，也是凸显**性能变差的第一幕**。

其分为以下三个处理分支：

- 若发现 read 中存在该元素，但已经被标记为已删除（expunged），则说明 dirty 不等于 nil（dirty 中肯定不存在该元素）。其将会执行如下操作。
- 将元素状态从已删除（expunged）更改为 nil。
- 将元素插入 dirty 中。
- 若发现 read 中不存在该元素，但 dirty 中存在该元素，则直接写入更新 entry 的指向。
- 若发现 read 和 dirty 都不存在该元素，则从 read 中复制未被标记删除的数据，并向 dirty 中插入该元素，赋予元素值 entry 的指向。

我们理一理，写入过程的整体流程就是：

- 查 read，read 上没有，或者已标记删除状态。
- 上互斥锁（Mutex）。
- 操作 dirty，根据各种数据情况和状态进行处理。

回到最初的话题，为什么他写入性能差那么多。究其原因：

- 写入一定要会经过 read，无论如何都比别人多一层，后续还要查数据情况和状态，性能开销相较更大。
- （第三个处理分支）当初始化或者 dirty 被提升后，会从 read 中复制全量的数据，若 read 中数据量大，则会影响性能。

可得知 `sync.Map` 类型不适合写多的场景，读多写少是比较好的。

若有大数据量的场景，则需要考虑 read 复制数据时的偶然性能抖动是否能够接受。

### 删除过程

这时候可能有小伙伴在想了。写入过程，理论上和删除不会差太远。怎么 `sync.Map` 类型的删除的性能似乎还行，这里面有什么猫腻？

源码如下：

```
func (m *Map) LoadAndDelete(key interface{}) (value interface{}, loaded bool) {
 read, _ := m.read.Load().(readOnly)
 e, ok := read.m[key]
 ...
  if ok {
  return e.delete()
 }
}

复制代码
```

删除是标准的开场，依然先到 read 检查该元素是否存在。

若存在，则调用 `delete` 标记为 expunged（删除状态），非常高效。可以明确在 read 中的元素，被删除，性能是非常好的。

若不存在，也就是走到 dirty 流程中：

```
func (m *Map) LoadAndDelete(key interface{}) (value interface{}, loaded bool) {
 ...
 if !ok && read.amended {
  m.mu.Lock()
  read, _ = m.read.Load().(readOnly)
  e, ok = read.m[key]
  if !ok && read.amended {
   e, ok = m.dirty[key]
   delete(m.dirty, key)
   m.missLocked()
  }
  m.mu.Unlock()
 }
 ...
 return nil, false
}

复制代码
```

若 read 中不存在该元素，dirty 不为空，read 与 dirty 不一致（利用 amended 判别），则表明要操作 dirty，上互斥锁。

再重复进行双重检查，若 read 仍然不存在该元素。则调用 delete 方法从 dirty 中标记该元素的删除。

需要注意，出现频率较高的 delete 方法：

```
func (e *entry) delete() (value interface{}, ok bool) {
 for {
  p := atomic.LoadPointer(&e.p)
  if p == nil || p == expunged {
   return nil, false
  }
  if atomic.CompareAndSwapPointer(&e.p, p, nil) {
   return *(*interface{})(p), true
  }
 }
}

复制代码
```

该方法都是将 entry.p 置为 nil，并且标记为 expunged（删除状态），而**不是真真正正的删除**。

注：不要误用 `sync.Map`，前段时间从字节大佬分享的案例来看，他们将一个连接作为 key 放了进去，于是和这个连接相关的，例如：buffer 的内存就永远无法释放了...