# Go 面试完整知识体系

> **目标**: 中高级后端工程师面试 | 覆盖 95%+ 考点 | 底层原理 + 面试应答 + 手写代码 + 场景排查

---

# 第一章：语言基础

---

## 1.1 变量与类型系统

### 一、背景（为什么需要）

Go 的类型系统是理解所有后续知识（接口、反射、并发、内存）的基石。面试中类型系统的问题看似简单，但能轻易区分"用过 Go"和"理解 Go"的候选人。

### 二、核心概念（是什么）

**Go 的 25 个关键字：**
```
break    default     func   interface   select
case     defer       go     map         struct
chan     else        goto   package     switch
const    fallthrough if     range       type
continue for         import return      var
```

**值类型 vs 引用语义类型：**
- **值类型**（拷贝整个数据）：int, float, bool, string, array, struct
- **引用语义类型**（拷贝 header，共享底层）：slice, map, channel, interface, func, pointer
- **核心原则：Go 只有值传递，没有引用传递。** slice/map 传参时拷贝了 header，但 header 内含指针指向共享底层数据。

**零值设计哲学：** Go 中每个类型都有零值，设计良好的零值应当是"可用的"：
- slice 零值 nil → append 可用
- sync.Mutex 零值 → 直接可用，无需 new
- strings.Builder 零值 → 直接可用
- map 零值 nil → 读安全，写 panic
- channel 零值 nil → 收发永久阻塞

**new vs make：**
- `new(T)`：分配零值内存，返回 `*T`。只分配不初始化。
- `make(T, args)`：只用于 slice/map/channel，返回初始化好的 T（不是指针）。
- 为什么 map 不能 new？map 底层是 `*hmap`，new 只返回 nil 指针，写操作会 panic。

**string 底层结构：**
```go
type stringStruct struct {
    str unsafe.Pointer  // 底层字节数组指针
    len int             // 字节数，不是字符数！
}
// len("你好") == 6，不是 2
// 字符串不可变，s[0]='x' 编译错误
// string ↔ []byte 互转都会拷贝数据
```

**type 定义 vs type 别名：**
- `type A B` → 新类型，不兼容，可定义自己的方法
- `type A = B` → 别名，完全等价

### 三、底层原理（重点）

**interface 装箱与类型断言：**
```go
// 类型断言底层：检查 interface 内部的 _type 字段是否匹配目标类型
var v interface{} = "hello"
s := v.(string)      // runtime 检查 _type == stringType
s, ok := v.(string)  // 不匹配时 ok=false，不 panic
i := v.(int)         // _type 不匹配 → panic
```

**iota 原理：** 编译器在 const 块内维护的行号计数器，每个 const 块重置。

### 四、关键设计（为什么这样设计）

- **零值可用性**：减少构造函数和初始化的 boilerplate，鼓励"声明即用"
- **25 个关键字**：降低学习曲线，减少语法糖，代码风格统一
- **隐式接口实现**：解耦定义和使用方，符合 Go 的"接受接口，返回结构体"哲学
- **string 不可变**：线程安全，哈希稳定，内存管理简单

### 五、面试高频问题 ⭐

**Q1：new 和 make 的区别？**
> new 分配零值返回指针（任何类型），make 只用于 slice/map/channel 并完成初始化。map 用 new 会返回 nil 指针，写入 panic。

**Q2：Go 有引用传递吗？**
> 没有。Go 只有值传递。slice/map 传参拷贝了 header（24B），但 header 包含指向底层数据的指针，所以函数内修改底层数据外部可见。但 append 扩容后，外部 slice 的 header 不变。

**Q3：nil slice 和 empty slice 的区别？**
> 功能上几乎一致（都可 append, range, len），区别：nil slice 判 `== nil` 为 true，JSON 序列化为 `null`；empty slice JSON 序列化为 `[]`。

**Q4：string 和 []byte 互转一定会拷贝吗？**
> 标准转换会拷贝。`string(b)` 和 `[]byte(s)` 都会分配新内存并复制数据，保证内存安全隔离。零拷贝互转需要 unsafe 包（生产慎用）。

### 六、总结（速记版）

1. Go 只有值传递，slice/map 是"拷贝 header + 共享底层"
2. `new` 返回指针只分配不初始化，`make` 只用于三种引用类型并初始化
3. string 底层 `{*byte, len}`，不可变，len 返回字节数
4. 零值设计是 Go 哲学：slice/sync.Mutex/strings.Builder 零值可用
5. 接口值 ≠ nil 陷阱：`{type: *T, data: nil} ≠ nil`

---

## 1.2 数组与切片 ⭐🔥

### 一、背景（为什么需要）

切片是 Go 中使用最频繁的数据结构。面试中 slice 的底层理解直接决定了能否通过基础关。90% 的 slice bug 源自不理解 header 和底层数组的关系。

### 二、核心概念（是什么）

**数组 vs 切片：**
| | 数组 | 切片 |
|------|------|------|
| 长度 | 固定，类型的一部分 `[3]int` | 动态 |
| 传参 | 拷贝整个数组 | 拷贝 header（24B），共享底层 |
| 零值 | 每个元素零值 | nil（但 append 可用） |

**切片底层结构（24B, 64位系统）：**
```go
type SliceHeader struct {
    Data uintptr  // 底层数组指针
    Len  int      // 当前长度
    Cap  int      // 容量
}
```

### 三、底层原理（重点）

**扩容机制（Go 1.18+ / runtime.growslice）：**

```
func growslice(et *_type, old slice, cap int) slice {
    // 步骤 1：计算 newcap
    newcap := old.cap
    doublecap := newcap + newcap
    if cap > doublecap {            // 期望容量大于翻倍 → 直接用期望容量
        newcap = cap
    } else {
        if old.cap < 256 {
            newcap = doublecap      // < 256：翻倍
        } else {
            // ≥ 256：过渡公式，逐渐降低增长率
            for 0 < newcap && newcap < cap {
                newcap += (newcap + 3*256) / 4
            }
            if newcap <= 0 {
                newcap = cap        // 溢出兜底
            }
        }
    }

    // 步骤 2：roundupsize — 向内存分配器对齐
    // 根据元素类型大小，将所需的字节数向上对齐到 Go 的 size class
    // 这一步是"为什么扩容后的 cap 可能比你算的更大"的原因
    lenmem := uintptr(old.len) * et.Size_
    newlenmem := uintptr(cap) * et.Size_
    capmem := roundupsize(newlenmem)       // 对齐到最近的 span class
    newcap = int(capmem / et.Size_)        // 反算真正的 cap

    // 步骤 3：分配新数组 + 拷贝
    mem = mallocgc(capmem, et, true)       // 用内存分配器分配
    memmove(mem, old.array, lenmem)         // 将旧数据整体拷贝到新数组
    // 注：memmove 是汇编实现，按字长拷贝（非逐个元素），非常快

    // 步骤 4：返回新 slice header
    return slice{mem, old.len, newcap}
}
```

**roundupsize 的影响（为什么扩容后的 cap 可能比你预期的大）：**

```
场景：[]int32，oldcap=100, 期望 cap=200
  newcap 公式 → 200（翻倍）
  需要内存：200 * 4 = 800 字节
  roundupsize(800) → 对齐到 size class → 实际分配 1024 字节的 span
  新 cap = 1024/4 = 256  ← 比期望的 200 更大！

为什么这样设计？
  Go 的内存分配器是按 size class 分配固定大小内存块的（见 3.1 节）。
  roundupsize 将请求的字节数对齐到 Go 的 67 种 size class 之一，
  既然已经分配了更大的内存块，不如把容量直接给用户用。
```

**Go 1.18 扩容公式的数学含义：**

```
公式：newcap += (newcap + 3*256) / 4

分析：
- 当 newcap = 256（刚触发）：newcap = 256 + (256+768)/4 = 256+256 = 512 → 翻倍
- 当 newcap = 1024：newcap = 1024 + (1024+768)/4 = 1024+448 = 1472 → ~1.43x
- 当 newcap = 4096：newcap = 4096 + (4096+768)/4 = 4096+1216 = 5312 → ~1.30x
- 当 newcap → ∞：增长率 → 1.25x

设计意图：非常平滑的递减曲线，不会像 "<1024 翻倍, ≥1024 1.25x" 那样有突变点。
```

**为什么从恒定 2x 改成阶梯式？** 减少大 slice 的内存浪费。1024 元素的 slice 翻倍需要 2048，但可能只需要 1025 个。256 作分界点是 trade-off：小 slice 快速减少 append 次数（翻倍仍划算），大 slice 控制内存浪费。

**截取与共享陷阱：**
```go
// 陷阱1：截取大数组导致内存泄漏
func getFirst100(bigData []byte) []byte {
    return bigData[:100]  // 返回的 slice 底层数组仍是 1GB！
}
// ✅ 修复：拷贝
func getFirst100Safe(bigData []byte) []byte {
    result := make([]byte, 100)
    copy(result, bigData[:100])
    return result
}

// 陷阱2：截取后 append 可能修改原数据
a := []int{1, 2, 3, 4, 5}
b := a[1:3]       // [2,3], len=2, cap=4
b = append(b, 6)  // b=[2,3,6], a=[1,2,3,6,5]  ← a 被改了!
b = append(b, 7)  // b=[2,3,6,7], a=[1,2,3,6,7]
b = append(b, 8)  // 触发扩容 → b 指向新数组，a 不再受影响
```

### 四、关键设计（为什么这样设计）

- **header 分离**：传参时只拷贝 24B header，参数传递 O(1)
- **动态扩容**：append 一次可能分配多余空间，换取后续 append 的 O(1) 均摊
- **底层共享**：截取操作 O(1)（不拷贝数据），性能优化，但需要开发者管理副作用
- **nil slice 可用**：`var s []int; s = append(s, 1)` 正常工作，减少 nil 判断

### 五、面试高频问题 ⭐

**Q1：slice 扩容后，原 slice 会被影响吗？**
> 扩容后新 slice 指向新底层数组，与旧数组脱离。但未扩容时，修改新 slice 会影响共享同一底层的所有 slice。

**Q2：如何安全截取一个大 slice？**
> `copy(dst, src[:n])` 显式拷贝需要的部分，让大数组可以被 GC。

**Q3：slice 作为函数参数，append 后外部会变吗？**
> 外部变量 header 不变（len, cap 不变）。但 append 未扩容时底层数组被修改，外部能"看到"数据变化（如果能访问到对应索引）。扩容后完全无关。

**Q4：`growslice` 中 roundupsize 是什么？扩容后 cap 为什么有时比我算的大？** ⚠️
> `roundupsize` 将所需字节数向上对齐到 Go 内存分配器的 67 种 size class 之一。例如 `[]int32` 扩容需要 800B，但 Go 分配了一个 1024B 的 span，所以你实际拿到 1024/4=256 的 cap。这是"白送"的容量——内存已分配，不用白不用。

**Q5：slice 的底层数组什么时候会被 GC 回收？**
> 当没有任何 slice header 的 Data 指针指向该底层数组时。⚠️ 截取大数组的片段（`bigSlice[:10]`）会保持整个大数组存活——因为新 slice 的 Data 指向大数组的起始位置（或中间位置），GC 无法部分回收。修复：`make`+`copy` 显式拷贝。

### 六、总结（速记版）

1. slice = 24B header `{Data, Len, Cap}` + 底层数组
2. cap < 256 翻倍扩容，≥ 256 渐进递减公式 → 1.25x
3. roundupsize 对齐到 size class → 实际 cap 可能比公式算的更大
4. 截取不拷贝 → 共享底层 → 大内存泄漏风险
5. append 前 `len < cap` 则共享底层，`len == cap` 扩容后脱离
6. 预防内存泄漏：截取后 `copy` 或 `make`+`copy`

---

## 1.3 Map ⭐🔥

### 一、背景（为什么需要）

Map 是最高频的数据结构之一。面试必问其底层实现，因为涉及哈希表设计、并发安全、内存管理等多维度知识。

### 二、核心概念（是什么）

Go map 是无序的哈希表，不是并发安全的。底层使用拉链法（溢出桶）解决冲突。

### 三、底层原理（重点）

**hmap 结构：**
```go
type hmap struct {
    count     int    // 元素个数
    B         uint8  // 桶数量的对数（2^B 个桶）
    hash0     uint32 // 随机哈希种子（防止 Hash DoS 攻击）
    buckets    unsafe.Pointer // 桶数组 (2^B 个 bmap)
    oldbuckets unsafe.Pointer // 扩容时的旧桶
    nevacuate  uintptr // 扩容进度
    noverflow  uint16 // 溢出桶数量
}
```

**bmap（桶）结构（编译期确定）：**
```
每个桶存放 8 个键值对：
- tophash [8]uint8    // hash 高 8 位，快速比较
- keys    [8]keytype  // 8 个 key（紧凑存储）
- values  [8]valuetype // 8 个 value（紧凑存储）
- overflow uintptr     // 溢出桶指针（链表）
```

**为什么 key/value 分开存储？** 内存对齐优化。`key/key/.../val/val/...` 比 `kv/kv/...` 减少 padding 浪费。

**渐进式扩容（evacuate）—— 面试核心考点：**

**扩容触发条件：**

| 条件 | 触发 | 原因 |
|------|------|------|
| 负载因子 > 6.5 | 翻倍扩容 | 桶太满，冲突严重 |
| 溢出桶 > 2^(B & 15) | 等量扩容 | 大量删除导致溢出桶稀疏（负载因子不一定高） |

**翻倍扩容（growWork）：**

```
B=2 时，有 2^2 = 4 个桶（bucket 0~3）

扩容后 B=3，2^3 = 8 个桶（bucket 0~7）

旧桶 i 的 key 被分流到两个新桶：
  - 新桶 i：hash 第 B 位（从 0 开始）为 0 的 key
  - 新桶 i + 2^B：hash 第 B 位为 1 的 key

示例（B=2 → B=3）：
  旧桶 0 → 新桶 0（低位=0） + 新桶 4（低位=1）
  旧桶 1 → 新桶 1（低位=0） + 新桶 5（低位=1）
  旧桶 2 → 新桶 2（低位=0） + 新桶 6（低位=1）
  旧桶 3 → 新桶 3（低位=0） + 新桶 7（低位=1）
```

**一次 evacuate 的逐步骤详解：**

```
函数：runtime.evacuate(t *maptype, h *hmap, oldbucket uintptr)

输入：要搬迁的旧桶编号

步骤 1：计算旧桶位置
  b = &oldbuckets[oldbucket]  // 指向旧桶

步骤 2：遍历旧桶及其溢出桶链表
  for ; b != nil; b = b.overflow(t) {
      for i := 0; i < 8; i++ {   // 每个桶 8 个 slot
          if b.tophash[i] == emptyOne || b.tophash[i] == emptyRest {
              continue            // 跳过已删除的空位
          }
          k := b.key(i)           // 取出 key
          v := b.value(i)         // 取出 value

步骤 3：重新计算 hash，分流到目标桶
          if h.flags & iterator {  // 翻倍扩容
              // 用低 B+1 位决定去新桶 x 还是新桶 y
              // hash & (1 << B) != 0 → 去新桶 (oldbucket + 2^B)
              // hash & (1 << B) == 0 → 去新桶 (oldbucket)
          } else {                 // 等量扩容
              // 桶数不变，直接搬到同名新桶
          }

步骤 4：写入目标桶
          dst.b.tophash[inserti] = keyHashTop(hash, t)  // 设置 tophash
          typedmemmove(t.key, dst.k, k)                  // 拷贝 key
          typedmemmove(t.elem, dst.v, v)                 // 拷贝 value
          // 如果目标桶满了 → 分配溢出桶 → 链到后面

步骤 5：标记旧桶为已搬迁
      }
  }

步骤 6：更新搬迁进度
  if h.nevacuate == oldbucket {
      h.nevacuate = oldbucket + 1  // 推进进度
      // 如果所有旧桶搬迁完成 → 释放 oldbuckets
  }
```

**等量扩容为什么能解决问题？**

```
场景：大量 delete 操作后
  旧桶中：8 个 slot 只有 2 个有效数据，6 个是 emptyOne
  溢出桶可能有 5 个，但实际总数据量很少

等量扩容过程：
  1. 分配新的等大小桶数组
  2. 遍历每个旧桶，把有效的数据重新紧凑排列到新桶
  3. 溢出桶被释放（因为数据被紧密排列，不再需要溢出）
  
结果：桶数不变，但数据更紧凑，减少了溢出桶链表长度
```

**为什么遍历无序？** Go 故意每次随机起点（`fastrand()`），防止程序依赖遍历顺序。具体做法：每次 `range` 时生成随机数 r → 从第 r 个桶开始遍历 → 桶内从第 r % 8 个 slot 开始。**扩容过程中**遍历还会打乱原本按 bucket 编号的顺序（正在搬迁的 key 可能在旧桶或新桶中出现）。

**map 的迭代器结构：**
```go
type hiter struct {
    key         unsafe.Pointer // 当前 key
    elem        unsafe.Pointer // 当前 value
    t           *maptype
    h           *hmap
    buckets     unsafe.Pointer // 迭代开始时 buckets 的快照
    bptr        *bmap         // 当前遍历到的桶
    startBucket uintptr        // 起始桶编号（随机）
    offset      uint8          // 桶内起始 slot（随机）
    wrapped     bool           // 是否绕回
    B           uint8
    i           uint8          // 桶内当前 slot
    bucket      uintptr        // 当前桶编号
    checkBucket uintptr        // 扩容时检查的桶编号
}
```

### 四、关键设计（为什么这样设计）

- **随机哈希种子（hash0）**：防 Hash DoS，每个 map 实例随机
- **tophash 快速过滤**：比较 hash 高 8 位，避免昂贵的内存比较
- **溢出桶链表**：处理好坏不等的 hash 分布
- **渐进式搬迁**：避免大 map 一次性搬迁的长时间延迟
- **非并发安全**：为了性能不内置锁，并发场景由开发者选择方案

### 五、面试高频问题 ⭐

**Q1：map 并发安全吗？怎么解决？**
> 不是并发安全的。并发读写触发 `fatal error: concurrent map read and map write`。方案：sync.RWMutex（通用）、sync.Map（读多写少）、分段锁/sharded map（高并发写）。

**Q2：sync.Map 适用场景？**
> 读多写少、key 集合稳定。每次写入新 key 需要在 dirty map 加锁操作，频繁新 key 写入退化为 Mutex+map。不适合需要 Len() 的场景。

**Q3：nil map 可以读吗？可以写吗？**
> 读安全（返回零值），range 安全（0 次迭代），delete 安全（no-op）。写 panic。

**Q4：map 的 key 有什么限制？**
> key 必须可比较（==）。slice/map/function 不能作为 key。`==` 必须能正确工作。

**Q5：删除 key 后内存会释放吗？**
> 不会立即释放底层桶。`delete` 只是将 tophash 标记为 `emptyOne` 并清零 key/value（让 GC 回收引用的对象），但底层桶内存不收缩。大量删除后需要等容扩容（等量重排）才能释放溢出桶。如需真正释放内存，只能将整个 map 置 nil 后重新 make。

**Q6：可以对 map 元素取地址吗？**
> **不可以。** `&m[key]` 编译报错。原因是 map 扩容时元素地址会变化，若允许取地址会出现**悬空指针**。如需可寻址的 value，用 `map[K]*V`（value 存指针）。

**Q7：map 可以边遍历边删除吗？**
> **单 goroutine 内可以**，Go 运行时明确支持 `range map` 时 `delete`，被删 key 在本次遍历中可能不出现但不会 crash。**多 goroutine 并发**（一个遍历一个删除）属于并发读写，触发 fatal error，需要加锁。

**Q8：map 扩容时，新插入的 key 会放到旧桶还是新桶？遍历呢？** 🔥
> - **写操作**：扩容期间，新的 key **只写入新桶**，不会写入旧桶
> - **读操作**：先查旧桶，再查新桶（确保不漏数据）
> - **遍历**：使用迭代器创建时的 `buckets` 快照，同时可能需要在旧桶和新桶之间切换（通过 `checkBucket` 字段追踪搬迁进度）
> - 这个设计保证扩容期间数据一致性，同时推进搬迁进度

**Q9：翻倍扩容时，旧桶的 key 如何决定去哪个新桶？** ⚠️
> 用 hash 的第 B 位（B 是扩容前 bucket 数的对数）。如果该位为 0 → 去新桶 `oldbucket`；该位为 1 → 去新桶 `oldbucket + 2^B`。例如 B=2 时旧桶 0 的数据分流到新桶 0（第 2 位为 0）和新桶 4（第 2 位为 1）。这样扩容后每个旧桶的数据均匀分流到两个新桶，负载均衡。

**Q10：map 的迭代器是如何处理扩容的？为什么扩容时遍历也可能"乱序"？**
> 迭代器创建时记录当前 `buckets` 指针 + 随机的 `startBucket` 和 `offset`。遍历过程中如果某个旧桶还未搬迁 → 直接遍历该旧桶。如果该旧桶在遍历过程中被搬迁 → 迭代器通过 `checkBucket` 在旧桶和新桶间切换，保证不会漏掉或重复输出 key。扩容导致的"新/旧桶切换"本身就是一种无序源。

### 六、总结（速记版）

1. hmap: `{B, hash0, buckets, oldbuckets}` + bmap: `{tophash, keys, values, overflow}`
2. 扩容分两种：负载因子 > 6.5 翻倍，溢出桶太多等量重排
3. 扩容是渐进式的（每次读写搬迁 1-2 桶）
4. 非并发安全 → sync.RWMutex / sync.Map / 分段锁
5. nil map 可读/可 delete/可 range，不可写

---

## 1.4 Channel ⭐🔥🔥

### 一、背景（为什么需要）

Channel 是 Go CSP 并发模型的核心："不要通过共享内存来通信，而要通过通信来共享内存。" 理解 channel 底层是区分初中级和高级候选人的关键标志。

### 二、核心概念（是什么）

Channel 是一个**环形队列 + 等待队列 + mutex** 组成的并发安全数据结构。

| | 无缓冲 `make(chan T)` | 有缓冲 `make(chan T, n)` |
|------|------|------|
| 发送等待 | 必须等接收者就绪 | 缓冲未满则直接写入 |
| 同步语义 | 严格同步（happens-before） | 异步（生产者可超前） |
| 用途 | 信号通知、同步 | 生产者-消费者解耦 |

### 三、底层原理（重点）

**hchan 结构：**
```go
type hchan struct {
    qcount   uint           // 当前队列元素数
    dataqsiz uint           // 环形队列大小
    buf      unsafe.Pointer // 环形队列指针
    elemsize uint16
    closed   uint32
    sendx    uint           // 发送索引
    recvx    uint           // 接收索引
    recvq    waitq          // 等待接收的 goroutine 队列
    sendq    waitq          // 等待发送的 goroutine 队列
    lock     mutex
}
```

**发送流程 `ch <- val`：**
```
1. 加锁
2. ch 已关闭 → panic("send on closed channel")
3. recvq 非空 → 直接从等待接收者取数据（零拷贝，不经过 buf），唤醒该 G
4. buf 未满 → 写入 buf[sendx]，sendx++, qcount++
5. buf 已满 → 当前 G 包装为 sudog 加入 sendq，解锁并 gopark 挂起
6. 解锁
```

**接收流程 `<-ch`：**
```
1. 加锁
2. sendq 非空（buf 为空时）→ 有缓冲：取 buf 头部给接收者，将发送者数据写入 buf 尾部
   无缓冲：直接从发送者拷贝数据（零拷贝），唤醒发送者
3. buf 非空 → 从 buf[recvx] 读取，recvx++, qcount--
4. buf 为空 且 ch 已关闭 → 返回零值 + false
5. buf 为空 且 ch 未关闭 → 当前 G 加入 recvq，挂起
6. 解锁
```

**有缓冲 channel 的 sendq 优先机制（面试加分点）：**
当接收者读取时发现 sendq 有等待发送者 → 先从 buf 头部弹出一个数据返回给接收者，再将 sendq 头部发送者的数据写入 buf 尾部。这保证了 FIFO 顺序，又避免了发送者饥饿。

**select 实现机制：**
```
1. 将所有 case 随机排列（pollorder，fastrandn 生成，保证公平）
2. 将所有 channel 按地址排序（lockorder，统一加锁顺序，避免死锁）
3. 按 lockorder 依次加锁所有 channel
4. 按 pollorder 检查每个 case 是否就绪
5. 有关键字 default → 无就绪 case 则执行 default
6. 全不就绪 → 挂起当前 G 到所有 channel 的等待队列，解锁，gopark
```

**select 为什么随机？** 防止前面 case 饥饿。固定顺序检查会导致前面的 case 总被优先执行。

### 四、关键设计（为什么这样设计）

- **环形队列**：O(1) 读写，循环利用内存
- **sendq/recvq**：阻塞时 goroutine 排队，唤醒时按 FIFO 顺序保证公平
- **零拷贝路径**：有等待者时直接传递数据，不经过 buf
- **关闭后行为**：读完缓冲后一直返回零值（不阻塞），方便 `for range` 优雅退出
- **内部 mutex**：channel 不是"无锁"的，性能场景下 mutex 可能比 channel 更快

### 五、面试高频问题 ⭐

**Q1：channel 关闭后还能读到什么？**
> 先读完所有缓冲数据，之后一直返回零值（ok=false），不会阻塞。不能向已关闭 channel 发送（panic）。不能重复关闭（panic）。只能在发送方关闭。

**Q2：nil channel 的行为？**
> 发送和接收永久阻塞（不 panic）。close(nil) 会 panic。这个特性在 select 中有用——可以把某个 case 的 channel 设为 nil 来"禁用"该分支。

**Q3：有缓冲 channel 接收时，如果 sendq 有等待者怎么处理？**
> 先从 buf 头部取出一个数据给接收者，然后把 sendq 头部等待发送者的数据写入 buf 尾部，唤醒发送者。保证 FIFO 且发送者不饿。

**Q4：select 的 case 为什么是随机的？怎么实现？**
> 防止饥饿。通过 `fastrandn` 生成 pollorder——每次 `select` 执行时，所有 case 的检查顺序都是随机的。

**Q5：什么场景 channel 会内存泄漏？**
> goroutine 在 channel 上永久阻塞（没人收/没人发/没人 close）。还有 select 中用 `time.After` 创建的 timer 在 case 未触发时不会被即时 GC。

**Q6：select + time.After 有什么坑？** ⚠️
> `time.After` 每次调用都创建一个新 timer。如果 select 在循环中且非超时 case 频繁触发，每次循环都会创建新 timer（旧 timer 到期后才被 GC），短时间大量创建会占满内存。
```go
// ❌ 循环中的 select + time.After = timer 泄漏
for {
    select {
    case <-ch:
        // ch 频繁可读 → 每次循环创建新 timer
    case <-time.After(5 * time.Second): // 泄漏！
    }
}

// ✅ 正确：复用 timer
timer := time.NewTimer(5 * time.Second)
for {
    timer.Reset(5 * time.Second) // 重用！
    select {
    case <-ch:
    case <-timer.C:
    }
}
```

**Q7：channel 是分配在栈上还是堆上？** ⚠️
> `make(chan T)` 创建的 **hchan 结构体本身在堆上**。channel 变量（指针）可以在栈上，但指向的 hchan 对象在堆上。原因是 channel 需要在多个 goroutine 之间共享，生命周期不能绑定到某个函数栈帧。

### 六、总结（速记版）

1. hchan = 环形队列 buf + 发送等待队列 sendq + 接收等待队列 recvq + mutex
2. 发送时 recvq 非空 → 零拷贝直接传递，跳过 buf
3. select 通过 pollorder 随机化 + lockorder 防死锁
4. 关闭后：先读缓冲 → 最后一直返回零值；不能发送和重复关闭
5. CSP 哲学：通过 channel 通信来共享数据，而不是通过共享内存来通信

---

## 1.5 函数、defer、panic/recover ⭐🔥

### 一、背景（为什么需要）

defer 是 Go 最独特的特性之一，panic/recover 是异常处理机制。这三者被放在一起讨论是因为它们在底层共享 defer 链表。

### 二、核心概念（是什么）

**方法接收者：**
- 值接收者：操作副本，不能修改原对象
- 指针接收者：操作原对象，可修改状态
- 编译器自动转换：`t.PtrMethod()` 等价 `(&t).PtrMethod()`

**方法集（Method Set）：**
- 类型 `T` 的方法集 = 值接收者方法
- 类型 `*T` 的方法集 = 值方法 + 指针方法
- 这是接口实现的判定基础：`T` 可能不能满足包含指针方法的接口

**defer 规则（必须全记）：**
1. 执行顺序：LIFO（后进先出）
2. 参数在 defer 语句执行时就求值（不是调用时）
3. 闭包读到的是变量的最终值
4. 命名返回值可以被 defer 修改

**闭包陷阱：**
```go
for i := 0; i < 3; i++ {
    go func() { fmt.Println(i) }()  // ❌ 全部打印 3（捕获引用）
}
for i := 0; i < 3; i++ {
    i := i                           // 创建新变量遮蔽
    go func() { fmt.Println(i) }()  // ✅
}
// 注：Go 1.22+ for range 循环变量每次迭代已自动为新变量
```

### 三、底层原理（重点）

**defer 实现演进：**
- **Go 1.12**：每个 defer 分配 `_defer` 结构体到堆，挂在 goroutine 的 `_defer` 链表
- **Go 1.13**：引入 `deferprocStack`，允许栈上分配 `_defer`（提升 ~30%）
- **Go 1.14+**：**open-coded defer** —— 如果 defer ≤ 8 个且无循环，编译器直接将 defer 内联到函数返回路径，零分配

```go
// _defer 结构（简化）
type _defer struct {
    started bool
    sp      uintptr   // 栈指针
    pc      uintptr   // 程序计数器
    fn      func()
    link    *_defer   // 链表
}
```

**panic/recover 机制：**
```
panic 发生时：
1. runtime 从当前 G 的 _defer 链表依次弹出并执行
2. 如果 defer 中调用了 recover() → _panic.recovered = true → 停止传播
3. 如果跑到 _defer 链表尽头仍未 recover → 打印 panic 信息 + 栈 trace → 程序退出
```

**recover 只在直接 defer 函数中有效：**
```go
defer func() {
    recover()          // ✅ 有效
}()
defer func() {
    func() { recover() }()  // ❌ 无效！不在直接 defer 函数中
}()
```

### 四、关键设计（为什么这样设计）

- **defer LIFO**：后获取的资源先释放（类似栈），先 defer 的锁后被释放
- **参数立即求值**：避免后续变量修改带来不确定行为，设计简单明确
- **open-coded defer**：让 defer 在热路径上几乎零开销
- **recover 只在 defer 中有效**：限制恢复范围，避免滥用
- **panic 用于不可恢复错误**：error 用于可预期错误，保持语义清晰

### 五、面试高频问题 ⭐

**Q1：defer 的执行顺序？参数求值时机？**
> LIFO 顺序执行。参数在 defer 语句执行时求值并固定，闭包变量读到的是最终值。命名返回值可以被 defer 修改。

**Q2：recover 的正确用法？**
> 必须在 defer 直接调用的函数中。嵌套函数中的 recover 无效。返回值需要用命名返回值承载。

**Q3：为什么 defer 是 LIFO 而不是 FIFO？**
> 对应资源获取和释放的对称性。后获取的资源（如锁、文件）应先释放，类似函数调用栈。

### 六、总结（速记版）

1. defer：LIFO 执行，参数立即求值，闭包读最终值
2. Go 1.14+ open-coded defer：≤8 个 defer 无循环 → 零分配
3. recover 只在直接 defer 函数中有效
4. 方法集：T 只有值方法，*T 有值方法+指针方法
5. 闭包捕获变量引用，循环中需用局部变量遮蔽

---

## 1.6 Interface ⭐🔥🔥

### 一、背景（为什么需要）

接口是 Go 实现多态的核心机制。理解接口底层结构（iface/eface）和 nil 接口陷阱是 Go 面试的"入门票"——答不上来基本被判为初级。

### 二、核心概念（是什么）

接口是一组方法的抽象。Go 采用**隐式实现**：类型无需显式声明 `implements`，只要实现了接口的所有方法就自动满足。

```go
// 空接口
type eface struct {
    _type *_type          // 动态类型信息
    data  unsafe.Pointer  // 指向具体值的指针
}

// 有方法接口
type iface struct {
    tab  *itab            // 方法表 + 类型信息
    data unsafe.Pointer   // 指向具体值的指针
}

// itab 结构
type itab struct {
    inter *_interfacetype  // 接口的静态类型
    _type *_type           // 具体值的动态类型
    hash  uint32           // _type.hash 拷贝，用于 switch/assert 加速
    fun   [1]uintptr       // 方法表（编译期确定实际大小）
}
```

**接口值 ≠ nil 的核心陷阱：**
```
接口变量为 nil ⇔ (_type == nil && data == nil)
{type: *MyError, data: nil} ≠ nil  ← 类型信息非 nil！
```

### 三、底层原理（重点）

**装箱（Boxing）过程：**
```
var w io.Writer
w = os.Stdout
// 1. 在堆上分配 iface 空间
// 2. 填充 itab — 接口类型 io.Writer + 具体类型 *os.File
// 3. data 指向 os.Stdout 指向的 *os.File

// itab 是编译期生成的全局只读表，不重复创建
```

**itab 方法派发流程（接口方法调用的完整路径）：**
```
调用 w.Write(buf) 时：
1. 从 iface.tab 取出 itab
2. 从 itab.fun[0] 取出 Write 方法的函数指针
   （偏移量由接口定义中的方法顺序决定，编译期确定）
3. 将 iface.data 作为第一个参数（receiver）传入
4. 通过函数指针间接调用 → 最终到达具体类型的方法

汇编层面：
  MOVQ  (w.tab+offset), AX   // 取 itab.fun[n] 函数指针
  MOVQ  w.data, BX            // 取具体值指针
  CALL  AX                    // 间接调用
```

**为什么无法内联？** 接口调用是**动态派发**——编译期不知道具体类型，只能通过运行时的 itab 查表跳转。编译器内联优化要求编译期确定调用目标，接口调用不满足这个条件。

**装箱代价：**
- 堆分配（绝大多数情况会逃逸）
- 方法调用多一次间接寻址（通过 itab.fun 跳转）
- 无法内联优化

### 四、关键设计（为什么这样设计）

- **隐式实现**：降低耦合，调用方定义接口（DIP 原则）
- **iface/eface 分离**：空接口不需要方法表，节省空间
- **itab 全局缓存**：不重复创建接口表，编译期生成
- **动态派发**：运行时通过方法表调用，支持多态
- **"接受接口，返回结构体"**：接口应该由调用方定义，实现方不知道接口存在

### 五、面试高频问题 ⭐

**Q1：interface{} 底层结构？nil 接口等于 nil 吗？** 🔥🔥
> 空接口 = `eface{_type, data}`，有方法接口 = `iface{tab, data}`。接口为 nil 的条件：`_type == nil && data == nil`。`var p *T = nil; var i interface{} = p` → i 不为 nil！因为 `_type = *T` 不为 nil。

**Q2：如何避免 nil 接口陷阱？**
> 函数返回 error 时，错误场景返回具体错误对象，无错误直接 `return nil`，不要 `return (*MyError)(nil)`。

**Q3：类型断言和类型转换的区别？**
> 类型转换在可兼容类型之间（int→float64），编译期检查。类型断言用于接口类型向下还原（interface → 具体类型），运行时检查，不匹配可 panic。

**Q4：接口的代价是什么？**
> 装箱时堆分配；方法调用间接寻址（通过 itab 方法表）；无法内联。性能敏感场景避免不必要的接口抽象。

**Q5：两个接口可以比较吗？什么时候会 panic？** ⚠️
> 可以比较，但有限制。两个接口值用 `==` 比较：动态类型相同且动态值相等 → true。**如果动态类型不可比较（如 slice, map, func），运行时会 panic**。安全比较使用 `reflect.DeepEqual`。

### 六、总结（速记版）

1. 接口 = `(类型信息, 值指针)`，两者都为 nil 时接口才为 nil
2. 有方法接口 iface={itab, data}，空接口 eface={_type, data}
3. itab 编译期生成，全局缓存，不重复创建
4. 装箱 = 堆分配 + 方法表填充，有性能代价
5. "接受接口，返回结构体" — 接口由调用方定义

---

## 1.7 Struct 内存对齐 ⭐🔥

### 一、背景（为什么需要）

CPU 按字长读取内存（64 位系统一次读 8 字节）。如果数据未对齐到自然边界，CPU 需要多次读取并拼接，性能下降。这是大厂高频考点。

### 二、核心概念（是什么）

Go 编译器会自动插入 padding（填充字节），确保每个字段对齐到其类型大小的边界。

```go
// 内存布局对比（64位系统）
type Bad struct {
    a bool   // 1B + 7B padding
    b int64  // 8B
    c bool   // 1B + 7B padding
}
// sizeof(Bad) = 24B  ← 浪费！

type Good struct {
    a bool   // 1B
    c bool   // 1B + 6B padding
    b int64  // 8B
}
// sizeof(Good) = 16B  ← 节省 33%！
```

### 三、底层原理（重点）

**对齐规则：**
1. 字段的起始地址必须能被 `min(字段大小, 系统字长)` 整除
2. 整个 struct 的大小必须是**最大字段大小**的倍数（方便数组连续存储）

```go
unsafe.Sizeof(Bad{})   // 24
unsafe.Alignof(Bad{})  // 8（按 int64 对齐）
unsafe.Offsetof(Bad{}.b) // 8（a 占 1B + 7B padding）
```

### 四、关键设计（为什么这样设计）

- **编译器自动对齐**：开发者不需要手动管理，但可以通过调整字段顺序优化
- **网络协议/文件格式**：如果 struct 直接映射到二进制格式，需要用 `//go:notinheap` 等特殊指令
- **空 struct{} 大小为 0**：`struct{}` 零大小，常用作 map value 节省空间（`map[string]struct{}`）

### 五、面试高频问题 ⭐

**Q1：一个 struct 怎么排列字段能省内存？**
> 将字段按大小从大到小排列（大字段先放），减少 padding。或用 `unsafe.Sizeof` 验证。

**Q2：为什么 `map[string]struct{}` 比 `map[string]bool` 省内存？**
> `struct{}` 大小 = 0，不占空间。`bool` 占 1B。大量 key 时差异显著。

**Q3：`unsafe.Sizeof` 和 `unsafe.Alignof` 的区别？**
> Sizeof 返回类型总大小（含 padding），Alignof 返回对齐边界。

### 六、总结（速记版）

1. 内存对齐 = 编译器按类型大小插入 padding，保证 CPU 一次读取
2. 字段从大到小排列 → 减少 padding → 节省内存
3. `unsafe.Sizeof` / `Alignof` / `Offsetof` 精细分析内存布局
4. 空 struct{} 大小为 0，适合做 map value 或 signal channel


---

## 1.8 语言细节高频问题 ⭐

### 1.8.1 struct tag 的作用与原理

**一句话总结：** struct tag 是附加在字段上的**元数据字符串**，通过反射在运行时读取，编译器本身不处理 tag 内容。

```go
type User struct {
    Name string `json:"name,omitempty"`   // JSON 序列化
    Age  int    `json:"age"`
    Pass string `json:"-"`                // 忽略该字段
    ID   int64  `gorm:"column:user_id;primaryKey"` // ORM 映射
    Email string `validate:"required,email"` // 参数校验
    Nick  string `form:"nick"`            // Gin 表单绑定
}
// 运行时读取：
tag := reflect.TypeOf(User{}).Field(0).Tag.Get("json") // "name,omitempty"
```

**常见用途：** JSON/XML 序列化、GORM/sqlx 数据库映射、validator 参数校验、Gin 表单绑定、protobuf 生成。

### 1.8.2 %v / %+v / %#v 的区别

```go
type Point struct{ X, Y int }
p := Point{1, 2}

fmt.Printf("%v\n",  p)   // {1 2}           — 默认格式，仅字段值
fmt.Printf("%+v\n", p)   // {X:1 Y:2}       — 带字段名
fmt.Printf("%#v\n", p)   // main.Point{X:1, Y:2} — Go 语法表示，含包名+类型
// %#v 输出是合法的 Go 语法，可直接用于调试和测试断言
```

### 1.8.3 init() 函数执行时机

**一句话总结：** `init()` 在 `main()` 之前由 runtime 自动调用，用于包级别初始化。

**执行顺序：**
1. 先执行被导入包的 `init()`（深度优先，按导入依赖顺序）
2. 同包内按源文件**名字典序**执行
3. 同文件内按 `init()` **出现顺序**执行
4. 包级变量初始化在 `init()` **之前**

**特点：** 一个包/文件可有多个 `init()`；不能被显式调用，无参数无返回值；常用于注册驱动（`import _ "driver"`）、初始化全局配置。

### 1.8.4 多返回值的底层实现

**一句话总结：** 多返回值通过**调用方栈帧**实现——调用方在栈上预留返回值空间，被调函数直接将返回值写入调用方栈帧。

```
调用方: 在栈帧上预留返回值空间
  │
  ▼ 调用 myFunc()
被调方: 将返回值写入调用方栈帧的预留位置
  │
  ▼ return
调用方: 直接从自己的栈帧读取返回值
```

**为什么命名返回值可以被 defer 修改？** 命名返回值本质上是**调用方栈帧上的变量**，defer 执行时 return 已完成赋值但函数未返回，defer 仍可访问栈帧并修改。

### 1.8.5 空白标识符 `_` 的用途

```go
// 1. 忽略返回值
val, _ := strconv.Atoi("123")

// 2. 忽略 range 的 index/value
for _, v := range slice {}

// 3. 仅触发 import 的 init()
import _ "net/http/pprof"

// 4. 编译期接口实现检查
var _ io.Writer = (*MyWriter)(nil) // 编译时报错如果没实现 io.Writer
```

### 1.8.6 rune 类型详解

**一句话总结：** `rune` 是 `int32` 的别名，代表一个 **Unicode 码点**。

```go
s := "你好"
fmt.Println(len(s))              // 6（UTF-8 字节数，不是字符数！）
fmt.Println(len([]rune(s)))      // 2（真正的字符数）

// range string 自动按 rune 迭代
for i, r := range s {
    fmt.Printf("%d: %c\n", i, r) // 0: 你, 3: 好（i 是字节索引）
}

// 按字符截取/反转 string → 先转 []rune
runes := []rune(s)
// 操作 runes...
result := string(runes)
```

### 1.8.7 for range 的地址问题（Go 1.22 前后）⭐

**一句话总结：** Go 1.22 之前，`for range` 循环变量是**复用**的（同一地址）；Go 1.22 起，每次迭代**创建新变量**。

```go
// Go 1.21 及之前 — 经典陷阱
for _, v := range slice {
    go func() { fmt.Println(&v) }() // 所有 goroutine 打印同一地址！
}
// 修复（Go 1.21 前）：v := v（局部变量遮蔽）或函数参数传值

// Go 1.22+ — 已修复
for _, v := range slice {
    go func() { fmt.Println(&v) }() // 每次迭代不同地址
}
```

**注意：** `&v` 始终是迭代变量副本的地址，不是原始 slice 元素的地址。要取原始元素地址用 `&slice[i]`。


---

## 1.9 Go 内存模型（Happens-Before）⭐🔥

### 一、背景（为什么需要）

Go 内存模型定义了**在什么条件下，一个 goroutine 对变量的写操作能被另一个 goroutine 观察到**。这是并发编程正确性的理论基础，也是大厂面试的必考题。

### 二、核心概念（是什么）

**核心原则：**
> 如果一个 goroutine 对变量 v 的写操作 w 想要被另一个 goroutine 的读操作 r 观察到，必须满足：**w happens-before r**（或者 w 和 r 之间没有数据竞争）。

**Happens-Before 定义：**
- 如果事件 e1 happens-before 事件 e2，则 e2 一定能观察到 e1 的结果
- 如果 e1 不 happens-before e2，且 e2 不 happens-before e1，则它们是**并发的**（concurrent）

**Go 中保证 happens-before 的同步机制：**

| 同步操作 | Happens-Before 保证 |
|---------|-------------------|
| `go func()` 启动 | go 语句 happens-before goroutine 执行 |
| channel send | send happens-before 对应的 receive 完成 |
| channel close | close happens-before 读到零值 |
| unbuffered channel receive | receive happens-before send 完成 |
| `sync.Mutex.Unlock()` | Unlock happens-before 后续 Lock 返回 |
| `sync.RWMutex.Unlock/RUnlock()` | Unlock/RUnlock happens-before 后续 Lock/RLock |
| `sync.Once.Do(f)` | f 完成 happens-before 其他 Do 返回 |
| `sync.WaitGroup.Wait()` | 最后一个 Done happens-before Wait 返回 |
| `atomic.Load/Store` | 单个原子操作是顺序一致的 |

### 三、底层原理（重点）

**为什么需要内存模型？**
现代 CPU 和编译器会进行**指令重排**（编译期重排+CPU乱序执行），导致代码的实际执行顺序和书写顺序不一致。Go 内存模型规定了同步操作的顺序保证，编译器在这些同步点插入**内存屏障**（memory barrier/fence）阻止重排。

**channel 的 happens-before 保证（最经典的例子）：**
```go
var a string

func f() {
    a = "hello, world"  // (1)
    <-done              // (2) unblock
}

func main() {
    go f()
    done <- true        // (3)
    println(a)          // (4) ✅ 保证打印 "hello, world"
}
// happens-before 链: (1) → (2) [goroutine内] → (3) [send happens-before receive]
//                     → (4) [goroutine内]
// 所以 (1) happens-before (4)，保证可见性
```

**Mutex 的 happens-before 保证：**
```go
var a string
var mu sync.Mutex

func f() {
    a = "hello"     // (1)
    mu.Unlock()     // (2)
}

func main() {
    mu.Lock()       // (3)
    go f()
    mu.Lock()       // (4) — 等 f() 完成 Unlock
    println(a)      // (5) ✅ 保证打印 "hello"
}
// (1) → (2) → (3) 之后的 (4) → (5)
// 因为 Unlock happens-before 后续 Lock 返回
```

**⚠️ 无同步 = 不保证可见性（最容易犯的错）：**
```go
var a string
var done bool

func setup() {
    a = "hello"
    done = true   // ❌ 没有同步！另一个 goroutine 可能看到 done=true 但 a 仍是空
}

func main() {
    go setup()
    for !done {}   // ❌ 忙等，不建立 happens-before 关系
    println(a)     // 可能打印 "" !!
}
// 修复：用 channel / atomic / sync 包建立 happens-before
```

**atomic 包的正确理解：**
```go
// atomic 保证单个变量操作的原子性和可见性，但不保证多个 atomic 变量之间的顺序
var a int32
var done int32

func setup() {
    a = 1
    atomic.StoreInt32(&done, 1) // ✅ done 的写对其他读可见
}

func main() {
    go setup()
    for atomic.LoadInt32(&done) == 0 {}
    println(a) // ❌ 仍可能打印 0！因为 Store 和 Load 只保证 done 本身的可见性，
    // 不保证 a 的写入对读方可见
}
// 正确做法：把 a 的赋值也改成 atomic，或使用 Mutex/channel
```

### 四、关键设计（为什么这样设计）

- **不是顺序一致性模型**：如果 Go 是完全的顺序一致性，就需要在每次内存访问前后插入内存屏障，性能极差。只在显式同步点插入屏障。
- **"Do not communicate by sharing memory; instead, share memory by communicating."** — Go 鼓励用 channel 通信，因为 channel 自动建立清晰的 happens-before 关系。
- **编译器/CPU 优化 + 程序员可控**：开发者不需要关心底层屏障，只需正确使用同步原语。

### 五、面试高频问题 ⭐

**Q1：什么是 Go 内存模型？为什么它很重要？**
> Go 内存模型定义了 goroutine 之间变量可见性的规则——什么条件下一个 goroutine 的写能被另一个 goroutine 看到。不遵循内存模型的代码在 `-race` 下会报 data race，且行为不可预期。

**Q2：用 `done` bool 变量做 goroutine 间信号通知有什么问题？**
> 纯粹的变量读写在 goroutine 之间不建立 happens-before 关系。一个 goroutine 写 `done = true`，另一个 goroutine 读到 `true`，不保证写 goroutine 之前对共享变量的写入可见。正确做法：用 channel、atomic、Mutex。

**Q3：atomic 真的能替代 Mutex 吗？**
> 不能完全替代。atomic 保证单变量的原子可见性，但不保证多个变量之间的顺序一致性（无临界区保护）。多个相关变量需要一起修改时，必须使用 Mutex。

**Q4：unbuffered channel 和 buffered channel 的 happens-before 保证有什么不同？** ⚠️
> - **unbuffered channel**：receive happens-before send 完成（发送者在接收者读取完成后才返回）→ 更强的同步语义
> - **buffered channel 第 k 个 receive**：happens-before 第 (k+C) 个 send 完成（C 是缓冲容量）
> - 本质区别：unbuffered channel 提供**双向同步**，buffered channel 只提供**单向同步**（发送者可以领先接收者 C 步）

### 六、总结（速记版）

1. Go 内存模型 = goroutine 间变量可见性的规则
2. Happens-Before 是保证可见性的唯一方式
3. 同步原语（channel/sync/atomic）自动建立 happens-before
4. 无同步的变量读写 = data race = 未定义行为
5. `go test -race` 检测 data race，但不是所有 race 都能被检测到
6. channel 通信是 Go 推荐的同步方式（自动 happens-before）

---

# 第二章：并发模型

---

## 2.1 Goroutine ⭐🔥🔥

### 一、背景（为什么需要）

并发是 Go 的"杀手级特性"。goroutine 是 Go 并发的基石——它不是线程，不是传统协程，是 Go 独创的轻量级并发单元。

### 二、核心概念（是什么）

**goroutine = 用户态轻量级线程 + 动态栈 + GMP 调度**

| | 线程 | goroutine |
|------|------|------|
| 管理者 | OS 内核 | Go runtime |
| 栈大小 | 固定 1-8MB | 初始 2KB，按需扩缩 |
| 切换代价 | ~1-10μs (内核态) | ~200ns (用户态) |
| 创建上限 | 数千 | 数十万~百万 |

**9 种状态：**
```
_Gidle → _Grunnable → _Grunning → _Gdead
                  ↓          ↓
              _Gwaiting  _Gsyscall
```
- `_Grunnable`: 在 P 的 runq 中排队
- `_Grunning`: 在 M 上执行
- `_Gwaiting`: 被 channel/网络/timer 阻塞
- `_Gsyscall`: 执行系统调用中
- `_Gdead`: 退出，结构体放入 gFree 缓存池复用

### 三、底层原理（重点）

**栈管理（核心知识点）：**

**连续栈 vs 分段栈（栈复制 vs 栈分裂）：**

Go 的 goroutine 栈经历了一次重大演进：

| | 分段栈（Go < 1.4） | 连续栈（Go 1.4+） |
|------|------|------|
| 实现 | 按需分配不连续的内存块（链表） | 一整块连续内存，扩容时复制 |
| 问题 | "hot split" 问题 | 复制开销，栈地址变化 |
| 状态 | 已废弃 | 当前方案 |

**分段栈的 "hot split" 问题：**
```
在函数调用频繁的循环中，栈在分裂点附近反复增长和收缩：
  for { f() }  // f() 导致栈略超当前段 → 分配新段（split）
                // f() 返回 → 新段释放
                // 下次调用又 split → 频繁分配/释放 → 性能极差
```

**连续栈的栈复制过程：**
```
1. 函数调用时发现栈不够 → runtime.morestack()
2. 分配一块新内存（2x 当前大小）
3. 将旧栈全部数据拷贝到新栈
4. 调整所有指向旧栈的指针（精确 GC 知道哪些位置是指针）
5. 释放旧栈
6. 新栈中继续执行

GC 时的栈收缩：
- 检查 goroutine 实际栈使用量
- 如果 < 当前栈大小的 1/4 → 收缩到 1/2
- 回收未使用内存
```

**栈扩容与缩容参数：**
- 初始：2KB（Go 1.4+）
- 扩容：每次翻倍，最大 1GB
- 缩容：GC 检查，使用 < 1/4 时缩到 1/2
- `runtime.morestack()` 分配新栈 → 拷贝数据 → 更新指针
- GC 时检查：实际使用 < 1/4 容量 → 收缩到 1/2

**goroutine 创建过程 `go myFunc()`：**
```
1. 编译器编译为 runtime.newproc(size, fn)
2. 从 P.gFree 获取 g 结构体（没有则 malg 分配）
3. 分配栈空间（初始 2KB）
4. 初始化 g: startpc = fn, sched.sp = 栈顶
5. 设置状态 _Grunnable
6. runqput 放入 P 本地队列（优先放 runnext）
7. 本地队列满（>256）→ 放一半到全局队列
```

**goroutine 泄漏：**
```go
// 场景1：channel 永久阻塞
go func() { ch <- 1 }()  // 没人收 → 永远阻塞

// 场景2：无限循环无退出
go func() { for { /* 没有 ctx.Done() */ } }()

// 排查：pprof goroutine profile
// import _ "net/http/pprof"
// go tool pprof http://localhost:6060/debug/pprof/goroutine
```

### 四、关键设计（为什么这样设计）

- **动态栈**：小 goroutine 不占用过多内存，大的 goroutine 按需扩展
- **gFree 缓存池**：g 结构体复用，减少 GC 和内存分配
- **g0 系统栈**：每个 M 有 g0，运行在系统栈上处理调度和 runtime 逻辑，与用户 goroutine 隔离
- **复用 g 结构体**：退出的 G 不释放内存，放入 gFree 池等待复用

### 五、面试高频问题 ⭐

**Q1：goroutine 与线程的根本区别？**
> goroutine 是用户态调度，线程是内核态调度；栈动态扩缩 vs 固定栈；切换成本纳秒级 vs 微秒级；可创建百万级 vs 数千级。

**Q2：goroutine 泄漏怎么排查？**
> pprof goroutine profile → 查看阻塞的 goroutine 栈 trace。每个 goroutine 都应有退出路径：ctx.Done() 检查、channel 超时、defer close。

**Q3：100 万个 goroutine 会发生什么？**
> 技术上可行（初始栈占 ~2GB）。需要注意：栈会扩张导致内存增长；GC 扫描根对象耗时增加；调度器压力。实际应用建议用 worker pool 控制并发度。

### 六、总结（速记版）

1. goroutine = 用户态轻量线程 + 初始 2KB 动态栈 + GMP M:N 调度
2. 状态：idle → runnable → running → waiting/syscall → dead
3. `go func()` → newproc → 取 gFree → 初始化 g → runqput
4. 泄漏 = goroutine 永久阻塞无法退出，pprof goroutine 排查
5. 栈按需翻倍增长，GC 时收缩（使用 < 1/4 容量时）

---

## 2.2 GMP 调度模型 🔥🔥🔥

### 一、背景（为什么需要）

GMP 是 Go 面试的"珠穆朗玛峰"。回答好 GMP 问题，基本可以证明你是深入阅读过源码的高级工程师。

**调度器演进：**
- **G-M 模型（早期）**：全局队列 + 全局大锁 → 严重锁竞争
- **G-M-P 模型**：引入 P 提供局部性 → 本地队列 + 本地 mcache
- **Go 1.14+**：信号抢占式调度

### 二、核心概念（是什么）

```
                    Go Runtime
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │    P0    │  │    P1    │  │    P2    │
  │ runq     │  │ runq     │  │ runq     │
  │ [G G G]  │  │ [G G G]  │  │ [G G]    │
  │ mcache   │  │ mcache   │  │ mcache   │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │              │              │
  ┌────▼─────┐  ┌────▼─────┐  ┌────▼─────┐
  │    M0    │  │    M1    │  │    M2    │
  │(OS线程)  │  │(OS线程)  │  │(OS线程)  │
  └──────────┘  └──────────┘  └──────────┘

  全局 G 队列 (Global Run Queue)
  网络轮询器 (Netpoller)
```

- **G (Goroutine)**：{栈, pc, 状态, 绑定的 M/P}。g0 是每个 M 的系统栈 goroutine。
- **M (Machine)**：OS 线程抽象。包含 g0、curg（当前运行的 G）、绑定的 P。最多 10000+。
- **P (Processor)**：调度上下文，**G 和 M 的中介**。包含本地 runq（256 个 G）、mcache。数量 = GOMAXPROCS。

### 三、底层原理（重点）

**调度循环 `schedule()`：**
```
1. 每 61 次调度检查全局队列（防止全局队列饥饿）
2. 从 P.runq 取 G（优先取 runnext，提高局部性）
3. runq 为空 → 从全局队列取
4. 全局为空 → 从 netpoller 取
5. 还没有 → 工作窃取（runqsteal）
6. 实在没找到 → M 进入自旋或休眠
```

**工作窃取（Work Stealing）：**
```
1. 随机起点（fastrand）→ 最多尝试 4 次
2. 从随机 P 的 runq 偷一半 goroutine
3. 为什么偷一半？分摊窃取成本，一次偷够
4. 为什么随机起点？避免多个 P 同时抢同一个 P（羊群效应）
```

**系统调用 Handoff 机制：**
```
1. G 进入 syscall → entersyscall()，P 状态 _Psyscall
2. sysmon 检测 P 在 syscall 超 10ms
3. sysmon 寻找空闲 M 或创建新 M
4. 新 M 接管 P（handoffp），继续执行 P.runq 中的 G
5. 原 syscall 返回后，原 G 尝试重新获取旧 P
6. 若 P 已被接管 → G 放入全局队列 → M 休眠
```

**抢占式调度（Go 1.14+）：**
- Go 1.13 前：只有协作式抢占（函数入口检查），死循环饿死同 P 的 G
- Go 1.14+：信号抢占 —— sysmon 检测 G 运行 > 10ms → 发 SIGURG → 信号处理器抢占 G
- 抢占点：函数调用、循环 back edge、安全检查点

**Netpoller 协作：**
```
1. G 调用 conn.Read() → fd 非阻塞 → 数据不可读
2. 注册到 netpoller (epoll_ctl ADD)
3. G 挂起 (gopark)，M 执行其他 G
4. netpoller 后台 epoll_wait
5. 数据就绪 → G 标记 runnable → 放回 P.runq
6. 调度循环重新执行 G
```

**Netpoller 的意义：** 用户代码是同步阻塞写法（`conn.Read()`），底层是异步事件驱动（epoll）。G 阻塞但 M 不被阻塞——这是 Go 高并发的核心秘密。

### 四、关键设计（为什么这样设计）

**为什么需要 P？**
1. 提供局部性（runq + mcache 绑定 P → 无锁）
2. 减少全局锁竞争（大多数调度发生在 P 本地）
3. 控制并发度（GOMAXPROCS = P 数量 = 同时执行的 G 数量）

**为什么工作窃取偷一半？** 一次窃取分摊成本，窃取太多可能导致源 P 饥饿。

**为什么每 61 次检查全局队列？** 防止全局队列中的 G 饿死，又不频繁访问全局队列引起锁竞争。

**为什么抢占式调度用信号机制？** 协作式对死循环无能为力。信号机制能在任何时候打断用户代码，保证公平。

### 五、面试高频问题 ⭐

**Q1：GMP 分别是什么？P 为什么需要？** 🔥🔥
> G=goroutine（任务），M=OS线程（执行者），P=调度上下文（中介）。P 提供局部性：每个 P 有独立的 runq 和 mcache，减少锁竞争。GOMAXPROCS 决定 P 数量。

**Q2：什么是工作窃取？怎么实现？** 🔥
> P 的 runq 为空时，从随机选择的另一个 P 的 runq 中偷走一半 G。使用随机起点避免羊群效应，偷一半分摊成本。

**Q3：系统调用时 P 和 M 会怎样？** 🔥
> M 上的 G 进入 syscall → M 与 P 分离 → sysmon 检测超 10ms → 新 M 接管 P → 原 M 在 syscall 返回后试图回收 P，失败则将 G 放全局队列。

**Q4：Go 的调度是抢占式的吗？1.14 前后区别？**
> 1.14 前是协作式（只在函数调用点让出），死循环会饿死同 P 的其他 G。1.14+ 是基于信号的异步抢占：sysmon 检测 G 运行 > 10ms → 发 SIGURG 信号抢占。

**Q5：netpoller 在调度中起什么作用？**
> 将同步 IO 转换为异步事件驱动。G 在 IO 上"阻塞"时，实际被挂起而 M 继续执行其他 G。IO 就绪后通过 netpoller 将 G 放回 runq。用户代码写同步，底层跑异步。

**Q6：g0 是什么？g0 和用户栈是如何切换的？** 🔥
> g0 是每个 M 绑定的特殊 goroutine，运行在**系统栈**（OS 线程栈），不受 goroutine 2KB 栈限制，不执行用户代码，专门执行调度逻辑（`schedule()`）。
> 
> **切换过程（汇编级）：**
> - 用户 G → g0：调用 `mcall(fn)` → 保存当前 G 的 PC/SP 等寄存器到 `g.sched` 字段 → 切换栈指针到 g0 的栈 → 在 g0 栈上执行调度逻辑
> - g0 → 用户 G：调用 `gogo(&g.sched)` → 从目标 G 的 `sched` 字段恢复 PC/SP 寄存器 → 切换栈指针到目标 G 的栈 → 跳转到目标 G 上次暂停位置继续执行
> - **全过程是纯用户态切换**（不涉及内核），开销约几十纳秒

**Q7：怎么用 GODEBUG 排查调度问题？** 🔥
> `GODEBUG=schedtrace=1000` 每 1000ms 打印一次调度器状态（P 数量、空闲 M、自旋线程数、全局队列长度等）。`GODEBUG=scheddetail=1` 则打印每个 P 的详细信息。
```bash
GODEBUG=schedtrace=1000 ./myapp
# 输出示例：
# SCHED 0ms: gomaxprocs=8 idleprocs=5 threads=12 spinning=0 ...
# idleprocs 高 → CPU 利用率低
# 全局队列持续增长 → 负载不均衡
# spinning=0 且 idleprocs>0 → 可能有 G 泄漏
```

**Q8：GOMAXPROCS 应该设为多少？为什么不能盲目用 runtime.NumCPU()？** ⚠️
> 默认 `GOMAXPROCS = runtime.NumCPU()`。大多数场景这个值就是最优的。但需要注意：
> - **IO 密集型**：可以设大一些（CPU 等待 IO 时 M 被阻塞，多出来的 P 可以调度其他 G）
> - **CPU 密集型**：等于 CPU 核数即可，更大只会增加调度开销
> - **容器环境**：Go 1.x 早期版本读的是宿主机的 CPU 核数而非容器的 cgroup 限制 → 需要手动设置 `GOMAXPROCS`。**Go 1.23+ 已自动识别 cgroup 限制。**
> - 生产建议：容器中显式设置 `GOMAXPROCS` 或使用 `go.uber.org/automaxprocs`

**Q9：P 的 runq 和全局 runq 分别存什么？为什么 P 的 runq 大小是 256？** ⚠️
> P 的 runq 是**环形队列**（实际上是一个固定大小 256 的数组 + head/tail 指针），存储该 P 本地可运行的 G。全局队列是链表，存储从其他 P 溢出或 syscall 返回后无 P 可回的 G。
> 256 是工程权衡：太小 → 频繁往全局队列放数据（需要加锁）；太大 → 调度局部性差，工作窃取一次偷太多。每个 G 大约占用几十到几百字节，256 个 G 约几十 KB，是 L1/L2 缓存友好的大小。

### 六、总结（速记版）

1. GMP = G(任务) + M(OS线程) + P(调度上下文+本地队列)
2. P 的设计目的：局部性 → 无锁本地队列，减少全局锁竞争
3. 调度循环：本地 runq → 全局队列 → netpoller → 工作窃取
4. 系统调用 handoff：P 与 M 分离，新 M 接管
5. Go 1.14+ 信号抢占：SIGURG 打断死循环
6. netpoller 让同步代码享受异步 IO 性能
7. `GODEBUG=schedtrace=1000` 追踪调度器瓶颈

---

## 2.3 Sync 包 ⭐🔥

### 一、背景（为什么需要）

sync 包是 Go 并发编程的基石。面试不仅考察"怎么用"，更考察"为什么这样实现"。sync.Mutex 的正常/饥饿模式、sync.Map 的双缓存设计，都是经典的工程 trade-off。

### 二、核心概念（是什么）

```go
sync.Mutex    // 互斥锁，有正常模式和饥饿模式
sync.RWMutex  // 读写锁，写优先
sync.WaitGroup // 等待组
sync.Once     // 确保函数只执行一次
sync.Cond     // 条件变量，广播/信号
sync.Pool     // 临时对象池，GC 会清空
sync.Map      // 并发安全 map，读写分离
```

### 三、底层原理（重点）

**sync.Mutex 实现：**
```go
type Mutex struct {
    state int32  // bit0=Locked, bit1=Woken, bit2=Starving, bit3-31=WaiterCount
    sema  uint32 // 信号量
}
```

**两种模式：**

| | 正常模式 | 饥饿模式 |
|------|------|------|
| 加锁 | 自旋 + CAS 抢占 | 直接排队到信号量 FIFO |
| 触发 | 默认 | 等待 > 1ms |
| 退出 | — | 队列空或等待 < 1ms |

**加锁过程：**
1. 快速路径：CAS(state, 0→Locked)，成功 → 获得锁
2. 慢路径：自旋（4 条件：多核 + P 空 + 次数 < 4 + 有其他 P 运行）
3. 多次 CAS 失败 → 信号量排队（semacquire）

**自旋条件（4 个必须同时满足）：**
- GOMAXPROCS > 1（多核）
- P 本地队列空
- 自旋次数 < 4
- 至少有一个其他 P 在运行

**sync.RWMutex：**
```go
type RWMutex struct {
    w           Mutex   // 写者互斥
    writerSem   uint32  // 写者等待信号量
    readerSem   uint32  // 读者等待信号量
    readerCount int32   // 等待的读者数（<0 时有写者等待）
    readerWait  int32   // 写者需要等待完成的读者数
}
// 设计：写优先。readerCount 变负 → 新读者阻塞
```

**sync.Once 实现：**
```go
func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 0 { // 快速路径
        o.doSlow(f)
    }
}
func (o *Once) doSlow(f func()) {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {                     // 双重检查
        defer atomic.StoreUint32(&o.done, 1) // 在 f() 前 defer！
        f()                             // 即使 f() panic，done 也会被标记
    }
}
```

**关键设计点：** `defer atomic.StoreUint32(&o.done, 1)` 在 `f()` **之前**写入 defer 栈。即使 f() panic，done 也会标记完成，不会导致后续 Do 永久阻塞。

**sync.WaitGroup 底层实现：**
```go
type WaitGroup struct {
    noCopy noCopy          // 防拷贝（go vet 检测）
    state  atomic.Uint64   // 高 32 位 = 计数器，低 32 位 = 等待者数量
    sema   uint32          // 信号量，用于阻塞/唤醒等待者
}
```
- `Add(n)`：原子操作修改计数器（高 32 位 +n）
- `Done()`：等价于 `Add(-1)`
- `Wait()`：等待者数量（低 32 位 +1），然后 `runtime_Semacquire` 挂起
- `Add` 归零时：通过 `runtime_Semrelease` **一次性唤醒所有**等待的 goroutine
- **注意事项**：`Add` 必须在 `Wait` 之前调用；WaitGroup 是值类型但不可拷贝（`noCopy` 检测）

**原子操作 CPU 指令级别：**
> Go `sync/atomic` 底层依赖 CPU 原生原子指令，如 x86 的 `LOCK` 前缀指令 + `CMPXCHG`（CAS）。`LOCK` 锁定总线/缓存行，保证操作的**不可分割性**。Go 1.19+ 引入了 `atomic.Int64`、`atomic.Pointer[T]` 等泛型原子类型，更易用。原子操作适合计数器、标志位等简单场景；复杂临界区用 Mutex。

**sync.Pool 原理：**
```go
// 每个 P 有一个 poolLocal（无锁私有 + 无锁共享队列）
type poolLocal struct {
    private any       // 仅当前 P 访问
    shared  poolChain // 无锁双端队列
}
// GC 清理：victim 机制，缓存最多持续一个 GC 周期
```

**sync.Map 原理（双缓存 + 原子指针）：**
```go
type Map struct {
    mu     sync.Mutex
    read   atomic.Pointer[readOnly] // 只读 map，无锁访问
    dirty  map[any]*entry           // 完整数据，加锁访问
    misses int                      // read 未命中累计
}
```
- **Load**：先无锁查 read → miss 后加锁查 dirty
- **Store**：read 中 key 存在且未标记删除 → CAS 更新（无锁）；否则加锁写 dirty
- **misses ≥ len(dirty)** → dirty 提升为 read（原子指针替换）
- **Delete**：不是物理删除，标记 entry.p = nil

**sync.Map entry 的三种状态（nil vs expunged）：**
```
正常值  →  Delete  →  nil（软删除）
 nil    → dirty提升 → expunged（已从 dirty 清除）
 nil    →  重新写入 →  直接更新 entry（无锁，因为 dirty 中还有此 key）
expunged → 重新写入 →  加锁重新加入 dirty（dirty 中已无此 key）
```
- **nil（软删除）**：key 被 Delete，但 dirty map 中可能还有此 entry。若用户重新 Store 此 key，可**无锁 CAS 更新**（entry 指针仍在 dirty 中共享）
- **expunged（已清除）**：dirty 提升为 read 时，nil 状态的 entry 不会被拷贝到新 dirty，并被标记为 expunged。此时重新 Store 必须**加锁**将其重新加入 dirty
- 两种状态的设计目的：延迟删除 → 减少加锁次数，提高并发性能

### 四、关键设计（为什么这样设计）

**Mutex 为什么有饥饿模式？** 防止等待久的 goroutine 被持续抢断（新来的 goroutine 在 CAS 自旋中有优势）。

**Once 为什么用 defer 标记 done？** 防止 f() panic 导致后续调用永久阻塞。

**sync.Pool 为什么 GC 会清空？** 防止池无限增长。用 victim 机制缓存一个 GC 周期。

**sync.Map 为什么不适合大量新 key？** 每次新 key 写入都要加锁查 dirty 和 read，miss 增加，退化为 Mutex+map。

**sync.Cond（条件变量）：** ⭐

Cond 用于**等待或通知**某个条件发生。与 Mutex 不同，Cond 可以让 goroutine 在某个条件不满足时挂起（Wait），条件满足时被唤醒（Signal/Broadcast）。

```go
var mu sync.Mutex
cond := sync.NewCond(&mu)
ready := false

// 等待方
go func() {
    cond.L.Lock()
    for !ready {           // ⚠️ 必须用 for 而不是 if！
        cond.Wait()        // Wait 内部：解锁 → 挂起 → 被唤醒后重新加锁
    }
    // ready == true, 继续处理
    cond.L.Unlock()
}()

// 通知方
cond.L.Lock()
ready = true
cond.L.Unlock()
cond.Broadcast()  // 或 cond.Signal() 唤醒单个
```

**面试要点：**
- `Wait()` 会原子地**解锁并挂起** goroutine，被唤醒后重新加锁
- **必须用 `for` 包裹 `Wait()`**（不是 `if`）— 防止虚假唤醒（spurious wakeup）
- `Signal()` 唤醒一个等待的 goroutine，`Broadcast()` 唤醒所有
- 适用场景：生产者-消费者队列、连接池等待、批量处理触发

### 五、面试高频问题 ⭐

**Q1：sync.Mutex 的正常模式和饥饿模式？**
> 正常模式：自旋 + CAS 抢占（不保证 FIFO）。饥饿模式：进入条件等待 > 1ms，直接排队到信号量（FIFO，保证公平）。退出条件：队列空或等待 < 1ms。

**Q2：sync.Once 怎么保证即使 f() panic 也只执行一次？**
> `defer atomic.StoreUint32(&o.done, 1)` 在 f() 调用前 defer，即使 f() panic 也会触发 defer 链，done 会被标记完成。

**Q3：sync.Pool 适合做什么？不适合做什么？**
> ✅ 临时对象复用，减少 GC 压力（如 buffer pool）。❌ 持久化缓存（GC 会清空），含状态的连接池。

**Q4：sync.Map 的读和写分别走什么路径？**
> 读：先无锁查 read → 命中直接返回 → miss 后加锁查 dirty。写：read 中 key 存在且未删除 → CAS 更新（无锁）；否则加锁写 dirty。

### 六、总结（速记版）

1. Mutex = CAS + 信号量，正常模式(自旋抢占) + 饥饿模式(FIFO排队)
2. Once = 双重检查锁 + defer 标记 done（防 f() panic 导致永不被标记）
3. Pool = P 本地缓存 + victim 机制（GC 最多缓存一个周期）
4. Map = read(无锁原子指针) + dirty(加锁) 读写分离
5. RWMutex 写优先：readerCount < 0 时新读者阻塞

---

## 2.4 Context ⭐🔥

### 一、背景（为什么需要）

在微服务和并发系统中，需要在 goroutine 之间传播取消信号、超时控制、请求范围的元数据。Context 解决了"如何优雅地取消一个 goroutine 树"的问题。

### 二、核心概念（是什么）

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}       // 取消信号
    Err() error                  // 取消原因
    Value(key any) any           // 请求范围的值
}

// 四种创建方法
context.Background()    // 根 context
context.TODO()          // 占位符
context.WithCancel(parent)       // 可取消
context.WithTimeout(parent, d)   // 超时
context.WithDeadline(parent, t)  // 截止时间
context.WithValue(parent, k, v)  // 携带值
```

### 三、底层原理（重点）

**cancelCtx 结构：**
```go
type cancelCtx struct {
    Context                      // 父 context
    mu       sync.Mutex
    done     atomic.Value        // chan struct{}，惰性创建
    children map[canceler]struct{} // 子 context 集合
    err      error
}
```

**取消传播流程：**
```
1. cancel() 调用
2. 关闭 done channel（close(chan struct{})）
3. 遍历 children，递归调用 child.cancel()
4. 从 parent.children 中移除自己
5. 所有 <-ctx.Done() 的监听者立即返回（close 的 channel 可无限读）
```

**关键实现细节：**
- `Done()` 返回的 channel 是**惰性创建**的（第一次调用时 make）
- 关闭 channel 后，所有监听者同时被通知
- cancel 函数**必须被调用**，否则 goroutine 泄漏（即使超时 timer 触发 cancel 也需要确保 defer cancel()）
- WithTimeout 内部用 `time.AfterFunc` 创建 timer，到期自动 cancel

### 四、关键设计（为什么这样设计）

- **树状结构**：取消父 → 传播所有子，实现级联取消
- **不可变传递**：With* 方法创建新 ctx，不修改原有 ctx
- **惰性创建 Done channel**：不使用取消功能的 ctx 不需要分配 channel
- **Close channel 做通知**：广播式通知，无竞争，graceful

### 五、面试高频问题 ⭐

**Q1：Context 的取消是怎么传播的？**
> 树状结构向下递归。父 cancel 关闭 Done channel → 遍历 children → 递归 cancel → 从 parent 中移除。

**Q2：Context.Value 应该放什么？不应该放什么？**
> ✅ 请求范围的元数据：traceID、userID、caller、deadline。❌ 业务数据、可选参数、大对象、依赖（db 连接等）。

**Q3：为什么 Context.Value 的 key 要用自定义类型？**
> 防止不同包使用相同的字符串 key 冲突。`type contextKey struct{}` 或 `type contextKey string` 确保不同包即使值相同也不冲突。

**Q3.5：context.Value 的查找过程是怎么样的？** ⚠️
> 沿着 context 链**向上逐层查找**（不是向下，不是全局）：
> 1. 检查当前 context 的 key 是否匹配 → 匹配返回 value
> 2. 不匹配 → 递归调用 parent.Value(key)
> 3. 到根 context（Background/TODO）→ 返回 nil
> 时间复杂度 O(n)，n 是 context 链深度。**不要用 context 传递大量数据或频繁查找的值**——应只传递请求范围元数据（traceID 等）。

**Q4：cancel 函数可以不调用吗？**
> 不可以。即使 WithTimeout 到期自动 cancel，也应该 `defer cancel()` — 如果提前完成，主动 cancel 可以释放 timer 资源，减少泄漏。

### 六、总结（速记版）

1. Context = 树状结构 + close channel 广播取消 + 惰性创建 Done()
2. 四种派生：WithCancel / WithTimeout / WithDeadline / WithValue
3. 取消传播：递归遍历 children，每个 child 关闭自己的 done channel
4. Value key 用自定义类型防冲突，只放请求范围元数据
5. cancel 函数必须调用（defer），否则 goroutine 泄漏

---

# 第三章：内存管理

---

## 3.1 内存分配器 ⭐🔥

### 一、背景（为什么需要）

Go 是 GC 语言，理解内存分配器是理解 GC 和性能优化的基础。Go 借鉴 TCMalloc 设计，用多级缓存减少锁竞争。

### 二、核心概念（是什么）

```
分配层级：
  P.mcache ──(无锁)──► mcentral ──(锁)──► mheap ──(mmap)──► OS

对象分级：
  Tiny  (1-16B)       → mcache.tiny 直接分配
  Small (17B-32KB)    → 66 种 span class (size class)
  Large (>32KB)       → 直接从 mheap 分配
```

- **mcache**：每个 P 独有，无锁。包含每种 span class 的缓存 + tiny 分配器。
- **mcentral**：每个 span class 一个，全局共享。维护 nonempty/empty 两个链表。
- **mheap**：全局唯一。管理所有堆内存，通过 heapArena 管理 64MB 的虚拟内存块。
- **span**：内存管理的基本单位，包含多个连续页。

### 三、底层原理（重点）

**mspan — 内存管理的基本单位：**

span 是 Go 内存分配器中最关键的概念。一个 span 管理 **N 个连续的内存页（page，8KB/页）**，这 N 个页被切分成 `nelems` 个相同大小的对象。

```go
// runtime/mheap.go
type mspan struct {
    next       *mspan     // 链表中的下一个 span
    prev       *mspan
    startAddr  uintptr    // span 管理的起始地址
    npages     uintptr    // 这个 span 占用了多少页
    nelems     uintptr    // 这个 span 包含多少个对象

    // 对象分配状态追踪（核心）
    allocCount  uint16    // 已分配的对象数
    freeindex   uintptr   // 下一个空闲对象的索引（近似空闲扫描起点）
    allocBits   *gcBits   // 位图：标记每个 object 是否已分配
    gcmarkBits  *gcBits   // 位图：GC 标记时使用（标记阶段和 allocBits 互换）

    spanclass   spanClass // span 的 size class
    state       mSpanState // span 状态：mSpanFree / mSpanManual / mSpanInUse
}
```

**span 三种状态：**
- `mSpanInUse`：正在被分配器使用
- `mSpanFree`：空闲，可被重新分配
- `mSpanManual`：被 runtime 内部使用（如栈）

**object 分配与释放（freelist → allocBits 演进）：**

```
Go 早期版本（< 1.12）：使用显式 freelist 链表
  每个未被分配的对象内存前 8B 存放下一个空闲对象的地址 → 单链表
  缺点：GC 标记时需遍历 freelist，与分配路径有竞争

Go 1.12+：使用 allocBits 位图 + freeindex 加速
  allocBits：一个 bit 对应一个对象，1 = 已分配，0 = 空闲
  freeindex：标记"从这里往后找空闲对象"，避免每次从头扫描
  分配时：从 freeindex 开始扫描 allocBits 找第一个 0 → 标记为 1 → 更新 freeindex
  释放时：对应 bit 从 1 → 0 → 如果释放位置 < freeindex，更新 freeindex
```

**为什么从 freelist 改成 allocBits？** freelist 需要修改对象内存来存储链表指针（破坏性写入），allocBits 是独立的位图，不侵入对象内存。GC 标记和分配路径操作不同的位图（gcmarkBits vs allocBits），标记完成时交换两个位图即可——无需遍历。

**size class 与 span class 对照表（关键片段）：**

```
class  bytes/obj  bytes/span  objects/span
  1       8         8192        1024
  2      16         8192         512
  3      24         8192         341
  4      32         8192         256
  5      48         8192         170
  ...
 20     1152        8192           7
  ...
 35     6912       73728          10
 ...
 66    32768       32768           1     ← 最大 small object
```

**`mallocgc()` — 完整分配路径：**

Go 中所有堆内存分配最终都进入 `runtime.mallocgc(size, typ, needzero)`：

```
mallocgc(size, typ, flags):

┌─ size == 0 ──→ 返回 runtime.zerobase 的地址（固定地址，不可修改）
│
├─ size > 32KB (maxSmallSize) ──→ Large 对象
│   1. size = alignUp(size, pageSize)
│   2. npages = size / pageSize
│   3. span = mheap.alloc(npages)          // 直接从堆分配页
│   4. span.allocCount = 1
│   5. 返回 span.startAddr
│
├─ size <= 32KB && typ != nil && typ.tiny → Tiny 分配器路径
│   1. offset = mcache.tiny offset
│   2. 如果当前 tiny 槽位剩余空间够 → 直接使用，tiny offset += size
│   3. 不够 → 从 mcache 获取对应 tinySpanClass 的 span
│   4. 分配新 tiny 对象（16B）
│   5. 返回地址，重置 tiny offset
│
└─ size <= 32KB 且非 tiny ──→ Small 对象
    1. sizeclass = size_to_class(size)      // 查表找到 size class
    2. span = mcache.alloc[sizeclass]       // 从 P 本地 mcache 获取
    3. span 为空 → mcache.refill(sizeclass) // 向 mcentral 申请
    4. mcentral 无空闲 → mheap.alloc(npages)
    5. object = span.freeindex 找到空闲位
    6. span.allocBits.set(object)           // 标记已分配
    7. span.allocCount++
    8. 返回 span.startAddr + object * size
```

**Tiny 分配器的合并分配机制：**

这是 Go 内存分配器最精巧的设计之一。对于 ≤ 16B 的小对象，Go 不会为每个对象单独分配内存，而是将多个小对象**合并放到同一个 16B 的内存槽**中：

```go
// mcache 中的 tiny 字段
type mcache struct {
    tiny       uintptr   // 当前 tiny 分配槽的起始地址
    tinyoffset uintptr   // 当前 tiny 槽已使用的偏移量
    // ...
}

// 场景：连续分配 3 个小对象
// 假设 mcache.tiny 当前指向一个新获取的 16B 槽

// new(bool)      — 1B → tinyoffset 0 → 偏移 +1
// new(byte)      — 1B → tinyoffset 1 → 偏移 +1
// new(int32)     — 4B → tinyoffset 2, 需对齐到 4 → tinyoffset=4, +4
// tinyoffset = 8，该 16B 槽还剩 8B
// 三个对象共享同一条 16B 内存，只需要一次 mallocgc 分配！

// 条件限制：
// - 只有 typ != nil 且 typ.tiny 标记为 true 的类型
// - 只适用于无指针的对象（避免 GC 扫描复杂化）
// - 单个对象 ≤ 16B
```

**page allocator（页分配器，Go 1.16+）：**

mheap 之下是页分配器，负责管理向 OS 申请的内存页。Go 1.16 重构了这一层：

```
mheap.pages (pageAlloc 结构):
  - 使用 radix tree 管理所有已映射的虚拟内存
  - 每个 page (8KB) 的状态用位图追踪
  - 分配时按 best-fit 或 first-fit 搜索连续空闲页
  - 使用位图 + 按层次汇总的方式加速较大 gaps 的查找（类似 buddy allocator 的思路）

OS 内存申请路径：
  mmap 或 VirtualAlloc → heapArena (64MB 块)
  → mheap.pages 按页管理
  → mspan 包装页为 span
  → mcentral/mcache 将 span 切分为 object
```

### 四、关键设计（为什么这样设计）

- **P 绑定 mcache**：大多数分配不需要加锁（类似 GMP 的 P 本地队列思路）
- **多级缓存架构**：越往上越无锁、越快、容量越小
- **size class 分级**：固定大小桶减少内存碎片，类似 TCMalloc。67 个不同 size 覆盖了 0~32KB 的所有请求
- **span 作为中间层**：在 page 和 object 之间建立映射。非侵入式 allocBits 位图（不修改对象内存本身）
- **Tiny 分配器**：将多个 ≤16B 无指针对象合并到同一内存槽，大幅减少微型对象的内存浪费
- **freeindex 加速**：避免每次分配都扫描整个 allocBits，从上次分配位置开始找
- **radix tree 页管理**：O(log n) 查找连续空闲页，支持 best-fit 和 first-fit

### 五、面试高频问题 ⭐

**Q1：Go 内存分配器有哪几层？**
> mcache（P 本地，无锁）→ mcentral（全局，锁）→ mheap（全局，锁）→ pageAlloc（radix tree）→ OS（mmap）。

**Q2：为什么每个 P 要绑定 mcache？**
> 减少锁竞争。大多数小对象分配走 mcache 无锁路径，大幅提升并发分配性能。

**Q3：span 是什么？allocBits 和 freelist 有什么区别？** ⚠️
> span 是内存管理的基本单位——连续 N 页内存被切分成等大的对象槽。旧版用 freelist 链表（在对象内存中存 next 指针），Go 1.12+ 改用 allocBits 位图（独立于对象内存）。位图的好处：不侵入对象内存（GC 友好）、分配和标记操作不同位图、标记完成时交换位图即可——无需遍历。

**Q4：`new(T)` → 最终拿到内存的完整路径是什么？** 🔥
> `new(T)` → 编译器会改写为 `runtime.newobject(typ)` → `mallocgc(size, typ, flags)` → 根据 size 走 Tiny/Small/Large 三条路径。Small 对象的完整路径：查 size class → mcache.alloc[class] → 如果 P 本地 span 有空闲 → allocBits 标记 → 返回地址。如果 mcache 无可用 span → mcache.refill() → mcentral.cacheSpan() → mheap.alloc(npages) → pageAlloc → 最终 mmap。

**Q5：Tiny 分配器是什么？什么对象会走 Tiny 路径？**
> 对于 ≤16B 的无指针小对象（如 bool、byte、小的 int），Go 不会单独分配，而是将它们合并到同一个 16B 内存槽中。多个 tiny 对象共享一条 mallocgc 分配的内存。大幅减少微型对象的内存开销。

**Q6：为什么 Go 不用 jemalloc/tcmalloc 而是自己实现？**
> Go 的内存分配器确实**借鉴**了 TCMalloc 的多级缓存 + size class 思想，但 Go 的分配器与 GMP 调度器深度集成（mcache 绑定 P、GC 写屏障协作、栈分配与堆分配的统一），这些是通用内存分配器做不到的。自己实现还能做汇编级的内联优化。

### 六、总结（速记版）

1. 三层架构：mcache(无锁) → mcentral(锁) → mheap(锁) → pageAlloc(radix tree) → OS
2. 对象分级：Tiny(1-16B 合并分配) / Small(17B-32KB, 67种class) / Large(>32KB 直接 page)
3. span = 连续 N 页切分为等大 object，allocBits 位图追踪分配状态
4. `mallocgc()` 单入口三条路径：Tiny(合并) / Small(mcache→mcentral→mheap) / Large(直接 mheap)
5. freeindex 加速分配：从上次分配位开始扫描，避免遍历整个位图
6. 设计核心：P 绑定 mcache → 大多数分配无锁 + 非侵入式位图 → GC 友好

---

## 3.2 逃逸分析 ⭐🔥🔥

### 一、背景（为什么需要）

逃逸分析决定了变量分配在栈还是堆。栈分配随函数返回自动回收（零开销），堆分配需要 GC 扫描回收。理解逃逸分析是写高性能 Go 代码的基础。

### 二、核心概念（是什么）

逃逸分析是**编译器在编译期**决定每个变量分配位置的过程。

**典型逃逸场景：**
1. 返回局部变量指针 → 逃逸
2. 变量被 `interface{}` 装箱 → 可能逃逸
3. 闭包引用了外部变量 → 可能逃逸
4. 大对象（大小超过栈帧限制）→ 逃逸
5. 变量在 slice/map 中，且 map/slice 可能被外部引用 → 逃逸

```bash
go build -gcflags="-m" main.go     # 查看逃逸分析
go build -gcflags="-m -m" main.go  # 更详细
```

### 三、底层原理（重点）

逃逸分析发生在编译期的 SSA（静态单赋值）优化阶段。编译器分析每个变量的作用域和引用情况：
- 如果变量的生命周期不超过所在函数 → 栈分配
- 如果变量可能在函数返回后仍被引用 → 堆分配（逃逸）

**编译器判断是保守的**：宁可多逃逸，不可少逃逸（否则出现悬垂指针）。

### 四、关键设计（为什么这样设计）

- **为什么需要逃逸分析？** 让 GC 只管理需要长期存活的对象，短命对象在栈上自动回收，减少 GC 压力
- **为什么不交给开发者决定？** Go 的设计哲学是让 runtime 和编译器自动决策，降低开发者心智负担
- **保守策略**：宁多逃逸不少逃逸，保证内存安全

### 五、面试高频问题 ⭐

**Q1：什么是逃逸分析？**
> 编译器在编译期决定变量分配在栈还是堆的过程。返回局部变量指针、interface 装箱、闭包引用会导致逃逸。

**Q2：如何减少逃逸？**
> 返回值类型用值不用指针；预分配 slice 容量减少 append 扩容；sync.Pool 复用对象；避免不必要的 interface 装箱。

**Q3：逃逸一定不好吗？**
> 不是。逃逸 = 堆分配 = GC 管理，增加 GC 压力但提供灵活性。完全避免逃逸不需要 GC，但 Go 本身不支持手动内存管理。关键是"热路径减少逃逸"。

### 六、总结（速记版）

1. 逃逸 = 编译器决定栈 vs 堆分配，编译期完成
2. 返回局部指针 / interface 装箱 / 闭包引用 → 逃逸
3. 栈分配零开销随函数返回回收，堆分配需要 GC
4. 编译器保守策略：宁多逃逸不少逃逸
5. 热路径减少逃逸 = 减少 GC 压力

---

## 3.3 GC 垃圾回收 ⭐🔥🔥

### 一、背景（为什么需要）

GC 是 Go 的"黑魔法"——绝大多数开发者不需要关心，但在面试中必须完全理解。Go GC 号称亚毫秒级 STW，这是怎么做到的？这是面试的终极问题之一。

### 二、核心概念（是什么）

**Go GC 演进：**
| 版本 | 算法 | STW |
|------|------|------|
| Go 1.3 | Mark-Sweep (纯 STW) | 数百 ms |
| Go 1.5 | 三色标记 + 并发清扫 | ~100ms |
| Go 1.8 | 并发标记+清扫, 混合写屏障 | <1ms |
| Go 1.19+ | 持续优化 | <0.5ms |

### 三、底层原理（重点）

**三色标记算法：**
- **白色**：尚未扫描 = 潜在垃圾
- **灰色**：自身已扫描，引用对象未扫描
- **黑色**：自身及引用都已扫描

```
1. GC 开始 → STW → 开启写屏障
2. 根对象扫描（STW）：栈/全局变量/寄存器中的对象 → 标记为灰色
3. 并发标记：灰→黑，将其引用的白→灰
4. 没有灰色对象 → 终止标记（STW）→ 关闭写屏障
5. 并发清扫：回收白色对象
```

**写屏障（核心中的核心）：**

并发标记期间用户代码仍在运行 → 黑色对象可能引用白色对象 → 白色被误回收。

**三种写屏障：**
- **插入屏障（Dijkstra）**：新建引用时，将被引用对象变灰
- **删除屏障（Yuasa）**：删除引用时，将被引用对象变灰
- **混合写屏障（Go 1.8+）**：结合两者

```go
// Go 混合写屏障（简化）
func writePointer(slot *unsafe.Pointer, ptr unsafe.Pointer) {
    shade(*slot)     // 旧指针目标变灰（删除屏障）
    shade(ptr)       // 新指针目标变灰（插入屏障）
    *slot = ptr      // 执行赋值
}
```

**为什么栈上不用写屏障？** 栈操作极频繁，加屏障严重影响性能。Go 在两次 STW 时扫描所有 goroutine 栈。

**两次 STW：**
1. **Mark Setup**：开启写屏障 + 扫描根对象 — 微秒级
2. **Mark Termination**：确保标记完成 + 关闭写屏障 — 微秒级

两次 STW 之间是并发标记，与堆大小无关。STW 时间只与根对象数量相关（goroutine 栈 + 全局变量等）。

**GC 触发时机：**
1. 内存分配触发：堆内存 > 上次 GC 后存活 × (1 + GOGC/100)
2. 定时触发：超过 2 分钟无 GC → 强制执行
3. 主动触发：`runtime.GC()`

**Mark Assist（辅助标记）：** 如果 goroutine 分配速度 > GC 标记回收速度 → 该 goroutine 被强制帮助 GC 标记（背压机制），保证 GC 不落后。

### 四、关键设计（为什么这样设计）

- **为什么用三色标记？** 支持并发标记——用户代码和 GC 同时运行
- **为什么需要写屏障？** 保证并发标记期间三色不变式（黑不引白）
- **为什么用混合写屏障？** 插入屏障需要 STW 重新扫描栈，删除屏障精度低。混合屏障两者优势结合
- **为什么栈不用写屏障？** 性能原因。改在 STW 时扫描所有栈
- **Mark Assist 为什么存在？** 防止分配速度超过标记速度导致堆无限增长

### 五、面试高频问题 ⭐

**Q1：Go GC 是如何做到亚毫秒级 STW 的？** 🔥🔥
> 1) 并发标记（标记阶段与用户代码同时运行）；2) 混合写屏障（Go 1.8+，栈不需屏障，两次 STW 仅在初始和终止时）；3) 两次 STW 仅与根对象数量有关，与堆大小无关；4) 并发清扫（回收白色对象非 STW）。

**Q2：三色标记中为什么需要写屏障？**
> 并发标记期间，用户代码可能在黑色对象中新增对白色对象的引用（或删除灰色→白色的引用），导致白色对象漏标记 → 被误回收。写屏障拦截指针写操作保证三色不变式。

**Q3：GC 触发的条件有哪些？**
> 1) 内存分配达到阈值 GOGC；2) 超过 2 分钟强制 GC；3) 手动 runtime.GC()。

**Q4：GC 的根对象是什么？** ⚠️（常答错）
> GC 根对象（GC Roots）是**不需要通过其他对象引用就能直接访问**的对象集合：
> 1. **全局变量**（data/bss 段中的指针）
> 2. **每个 goroutine 栈上的变量**（所有活跃 goroutine 栈帧中的指针）
> 3. **寄存器中的指针**
> 4. **运行时内部数据结构**（channel、defer 链表等持有的指针）
> 易错：不是只有 main 函数创建的对象，所有 goroutine 的栈变量都是根对象。

**Q5：常见的 GC 算法有哪些？各有什么优劣？** ⭐
> | 算法 | 原理 | 优点 | 缺点 |
> |------|------|------|------|
> | 引用计数 | 维护引用计数，归零回收 | 实时性好 | 循环引用，每次赋值有开销 |
> | 标记-清除 | 标记可达对象，清除其余 | 简单 | 内存碎片 |
> | 标记-整理 | 标记后移动存活对象 | 无碎片 | 移动对象开销大 |
> | 复制算法 | 分两半，存活复制到另一半 | 无碎片 | 内存利用率 50% |
> | 分代收集 | 按年龄分代，新对象频繁GC | 高效 | 需要写屏障维护跨代引用 |
> | 三色标记 | 并发标记，支持用户代码并发 | 低 STW | 需要写屏障 |

**Q6：GC 的核心指标有哪些？**
> 1. **STW 延迟**（P99 pause）：直接影响服务响应，Go 目标 < 1ms
> 2. **GC CPU 占比**（GC CPU fraction）：GC 消耗的 CPU，目标 < 25%
> 3. **堆内存使用**（HeapInuse / HeapAlloc）：当前堆使用量
> 4. **GC 频率**（NumGC/s）：过高说明分配压力大，需优化
> 5. **对象分配速率**（Alloc/s）：每秒分配量，决定 GC 触发频率
> 通过 `GODEBUG=gctrace=1`、pprof heap、`runtime.ReadMemStats` 观察。

**Q7：GOGC 怎么调优？默认值是多少？设为 off 会怎样？** ⚠️
> 默认 GOGC=100，意味着堆内存增长到上次 GC 后存活内存的 2 倍（1+100/100）时触发下一次 GC。
> - **增大 GOGC**（如 200, 400）→ 更少的 GC 次数，更高的峰值内存，CPU 降低
> - **减小 GOGC**（如 25, 50）→ 更多 GC 次数，更低峰值内存，CPU 升高（适合内存受限的容器环境）
> - **GOGC=off** → 完全关闭自动 GC，只能手动 `runtime.GC()` 触发。堆会一直增长到 OOM。只在极特殊的延迟敏感场景使用（如高频交易），且必须配合手动 GC
> - **GOMEMLIMIT（Go 1.19+）** → 软内存上限。设置后 Go 会在接近上限时主动触发 GC，防止 OOM。比 GOGC 更直观（直接设内存上限而非比例）
> ```bash
> GOMEMLIMIT=1GiB ./myapp  # 堆内存接近 1GiB 时主动 GC
> ```

**Q8：Go 为什么不用分代 GC？和 Java 的分代 GC 相比优劣？** ⚠️
> Go 最初选择了非分代的三色标记并发 GC，原因：
> 1. **逃逸分析使大量短命对象分配在栈上**（不是堆）——减少了新生代的必要性
> 2. **分代 GC 需要写屏障追踪跨代引用**（老对象→新对象），增加了常驻的 CPU 开销
> 3. **Go 的 GC 设计哲学**：宁愿让 STW 极低，接受稍微高一点的 CPU 开销。分代 GC 的 Minor GC 虽然快但不为零
> 4. Go 的目标不是极致吞吐，而是**稳定的低延迟**
> 
> 但 Go 社区一直在探索分代/region-based GC。目前（1.22+）仍以并发标记-清除为核心。
> 对比：Java G1/ZGC 采用分代+并发，GC 吞吐更高但实现更复杂。

### 六、总结（速记版）

1. GC = 三色标记 + 并发标记 + 两次微秒级 STW + 并发清扫
2. 写屏障 = 拦截指针写，保证"黑不引白"
3. Go 1.8+ 混合写屏障 = 插入屏障 + 删除屏障，栈用 STW 扫描
4. STW 时间与堆大小无关，只与根对象数量有关
5. GC 触发：GOGC 阈值 / 2分钟定时 / runtime.GC()
6. Mark Assist：分配太快时强制帮 GC 标记（背压机制）

---

# 第四章：标准库核心

---

## 4.1 net/http ⭐🔥

### 一、背景（为什么需要）

HTTP 服务是后端开发最常见场景，面试必问 net/http 的工作机制和性能调优。

### 二、核心概念（是什么）

**服务端流程：**
```
http.HandleFunc("/", handler)
http.ListenAndServe(":8080", nil)

底层：
1. net.Listen("tcp", ":8080")       // 创建 TCP listener
2. for { conn := ln.Accept() }      // 循环 accept
3. go c.serve(connCtx)              // 每连接一个 goroutine
4. 读取 HTTP 请求 → ServeMux 路由匹配 → 调用 handler
5. handler 返回值写入 conn
```

### 三、底层原理（重点）

**ServeMux 路由：** 前缀匹配（非精确匹配）。`/` 匹配所有路径，`/images/` 匹配 `/images/` 开头的所有路径。

**http.Client 连接池（高频面试点）：**
```go
type Transport struct {
    MaxIdleConns        int  // 总空闲连接数（默认 100）
    MaxIdleConnsPerHost int  // 每 host 空闲连接（默认 2!!!）
    MaxConnsPerHost     int  // 每 host 最大连接（0=无限）
    IdleConnTimeout     time.Duration // 空闲超时（默认 90s）
}
// ⚠️ MaxIdleConnsPerHost 默认只有 2！
// 高并发场景不调大会频繁创建新连接（TLS 握手代价巨大）
```

### 四、关键设计（为什么这样设计）

- **goroutine-per-connection**：同步代码 + 自动并发，代码自然
- **连接池复用**：减少 TCP/TLS 握手，提升性能
- **前缀路由**：简单但容易出坑（需要留意 `/` 的兜底行为）

### 五、面试高频问题 ⭐

**Q1：MaxIdleConnsPerHost 默认值是多少？为什么重要？**
> 默认 2。高并发场景如果不调大，会导致每次请求都建立新连接（TCP + TLS 握手），延迟暴增。

**Q2：HTTP 服务怎么优雅关闭？**
> `srv.Shutdown(ctx)` — 不接收新连接，等待现有连接处理完。配合 `signal.Notify` 监听 SIGTERM 信号。`srv.Close()` 是暴力关闭。

**Q3：http.Client 应该每次请求都 new 吗？**
> 不应该。应该创建全局单例复用连接池。每次 new Client 就丢失了所有已建立的连接。

**Q4：HTTP 中间件是怎么实现的？`http.Handler` 的嵌套模式？** ⚠️
> 中间件本质是 `func(http.Handler) http.Handler` 的函数类型。通过闭包嵌套实现链式调用：
```go
// 中间件类型
type Middleware func(http.Handler) http.Handler

// 日志中间件
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Printf("%s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}

// 链式组合
handler = LoggingMiddleware(AuthMiddleware(RateLimitMiddleware(finalHandler)))
```
> `http.Handler` 接口只有一个方法 `ServeHTTP(w, r)` — 这种极简设计使得中间件嵌套非常自然。每个中间件是一个"装饰器"，修改请求/响应或控制是否调用下一个 handler。

**Q5：`http.Hijack` 是什么？什么场景使用？**
> `Hijack()` 允许 handler 接管 TCP 连接，从 HTTP 协议切换到原始 TCP。用于 WebSocket 升级（gorilla/websocket）、HTTP CONNECT 隧道、SSE 等场景。调用后，`http.Server` 不再管理此连接。
```go
type Hijacker interface { Hijack() (net.Conn, *bufio.ReadWriter, error) }
// gorilla/websocket 内部就是通过 Hijack 接管了 TCP 连接来跑 WebSocket 协议
```

**Q6：`http.Server` 的几个关键超时参数？** ⚠️
> - `ReadTimeout`：读取整个请求（含 body）的超时（从连接 accept 开始计时）
> - `ReadHeaderTimeout`：只读请求头的超时（推荐至少设置此项，防 Slowloris 攻击）
> - `WriteTimeout`：写响应的超时（从读完请求头开始计时）
> - `IdleTimeout`：Keep-Alive 连接的空闲超时（Go 1.8+）
> - **推荐生产配置**：`ReadHeaderTimeout: 5s, ReadTimeout: 30s, WriteTimeout: 60s, IdleTimeout: 120s`

### 六、总结（速记版）

1. Listen → Accept → goroutine per connection → handler
2. MaxIdleConnsPerHost 默认值 2，生产必须调大
3. 全局 Client 复用连接池，不要每次请求 new
4. 优雅关闭：Shutdown(ctx) + 超时兜底
5. 中间件本质：`func(http.Handler) http.Handler` 闭包链
6. Hijack：HTTP 升级到原始 TCP，WebSocket 的底座
7. 生产必设 ReadHeaderTimeout（防 Slowloris 慢速攻击）

---

## 4.2 reflect ⭐

### 一、背景（为什么需要）

反射是 JSON 序列化、ORM、DI 框架等的基础。面试中反射问题考察对 Go 类型系统和运行时的理解深度。

### 二、核心概念（是什么）

**反射三定律：**
1. 从 `interface{}` 可获取 `reflect.Value`
2. 从 `reflect.Value` 可获取 `interface{}`
3. 要修改值，`reflect.Value` 必须是 `CanSet()`（通过指针的 `Elem()` 获取）

### 三、底层原理（重点）

**反射为什么慢（10-100x）？**
1. 每次操作需要类型检查和边界检查
2. `reflect.Value` 是结构体，需要堆分配
3. 方法调用是间接调用（通过函数表），无法内联
4. 类型信息（`_type`）需要在运行时查找

### 四、面试高频问题 ⭐

**Q1：反射为什么慢？如何优化？**
> 类型检查、堆分配、间接调用、无法内联。优化：缓存反射结果（如 `Type.Field` 结果）、避免热路径使用反射、用代码生成代替。

**Q2：怎么通过反射修改值？**
> 必须通过指针：`reflect.ValueOf(&x).Elem().SetInt(100)`。直接 `ValueOf(x)` 获取的是不可设置的副本。

**Q3：如何比较两个对象是否完全相同？** ⭐
> 1. `==` 运算符 — 仅适用于可比较类型
> 2. `reflect.DeepEqual(a, b)` — 深度比较，支持 slice/map/struct 等所有类型，递归比较每个元素/字段。注意：**nil slice 和空 slice 不相等**
> 3. `bytes.Equal` — 比较两个 `[]byte`
> 4. `cmp.Equal`（`github.com/google/go-cmp`）— 比 DeepEqual 更强大，支持自定义比较选项，推荐在测试中使用

**Q4：reflect.Value 的 Elem() 和 Interface() 分别做什么？**
> - `Interface()`：将 `reflect.Value` 还原为 `interface{}`（反射三定律第二条）。v 必须是一个有效的、可导出的值
> - `Elem()`：解引用——如果 v 是指针类型，返回指针指向的元素；如果 v 是 interface 类型，返回 interface 内部的具体值。要修改值时，必须通过 `Elem()` 获取 `CanSet()` 的 Value

**Q5：reflect.Type 和 reflect.Value 有什么区别？**
> `reflect.Type` 描述**类型信息**（类型名、Kind、方法数、字段信息、对齐信息）——基本等于 `eface._type` 的包装。`reflect.Value` 描述**值信息**（实际数据、是否有效、能否设置）——包装了 `{_type, data}` 的完整接口值。`Type` 可以来自类型本身（`reflect.TypeOf(T{})`），`Value` 需要具体值。

**Q6：如何通过反射调用一个方法？**
```go
v := reflect.ValueOf(obj)
m := v.MethodByName("DoSomething")
// MethodByName 返回一个绑定了 receiver 的 func Value
args := []reflect.Value{reflect.ValueOf("arg")}
results := m.Call(args) // 返回 []reflect.Value
```

### 五、总结（速记版）

1. 三定律：interface→Value, Value→interface, 可设置需通过指针
2. 慢的原因：类型检查 + 堆分配 + 间接调用 + 无法内联
3. 必须通过 `Elem()` 获取可设置的 Value
4. `Type` = 类型元信息，`Value` = 类型+具体值
5. `MethodByName().Call()` 实现反射方法调用

---

# 第五章：网络编程核心

## 5.1 Go IO 模型与 Netpoller 🔥

### 一、背景（为什么需要）

Go 的网络编程看似是同步阻塞写法，背后却是异步 IO。理解 Netpoller 是理解 Go 高并发网络能力的核心。

### 二、核心概念（是什么）

**Go IO 模型 = 用户代码同步阻塞 + 底层异步事件驱动**

- 开发者写：`conn.Read(buf)` — 同步阻塞
- 底层实现：非阻塞 fd + epoll + goroutine 切换

### 三、底层原理（重点）

**完整 IO 流程：**
```
1. G 调用 conn.Read() → fd 设置为非阻塞 → read(fd) 返回 EAGAIN
2. 将 fd 注册到 netpoller (epoll_ctl ADD)
3. G 挂起 (gopark)，M 继续执行其他 G
4. netpoller 后台 epoll_wait 等待事件
5. fd 可读 → netpoller 将 G 标记为 runnable → 放回 P.runq
6. G 被调度重新执行，继续 Read 操作
```

**为什么 Go 网络编程"简单但高性能"？**
→ 开发者写同步代码（易理解），底层自动转换为异步事件驱动（高性能）。G 在 IO 上"阻塞"时不占用 M（M 去执行别的 G），实现了 C10K/C100K 甚至更高的并发。

### 四、关键设计（为什么这样设计）

- **同步 API + 异步底层**：开发体验最简单，性能最优
- **goroutine-per-connection**：每个连接自然的独立上下文，不需要回调地狱
- **netpoller 与调度器深度集成**：IO 就绪的 G 直接回到 P 的本地队列，调度成本极低

### 五、面试高频问题 ⭐

**Q1：Go 的 IO 模型是什么？**
> 用户代码同步阻塞 + 底层异步事件驱动（epoll/kqueue）。G 在 IO 上"阻塞"时不占用 M，M 继续执行其他 G。

**Q2：netpoller 是怎么和调度器协作的？**
> G 在 IO 上阻塞 → 注册到 netpoller → G 挂起。netpoller 检测 IO 就绪 → G 回到 runnable → P 调度执行。整个过程对开发者透明。

### 六、总结（速记版）

1. 同步 API + 异步底层（epoll/kqueue）→ 简单且高性能
2. G 阻塞 IO ≠ M 阻塞，M 继续执行其他 G
3. netpoller = fd + epoll_wait + G 状态切换
4. goroutine-per-connection 实现 C100K 以上并发

---

# 第六章：工程实践

---

## 6.1 Go Modules 与 MVS ⭐

### 一、背景（为什么需要）

Go Modules 是 1.11+ 引入的依赖管理系统。理解 MVS 算法是理解 Go 依赖管理与 npm/pip 根本不同的关键。

### 二、核心概念（是什么）

**MVS（最小版本选择算法）**：不选最新版本，选**满足所有约束的最小版本**。

```
A → B@v1.2, C → B@v1.3
→ MVS 选 B@v1.3（满足 1.2 和 1.3 的最小版本）

A → B@v1.2, C → B@v2.0
→ B v2 module path 变了（/v2），MVS 视为两个不同模块，两个都保留
```

**关键指令：** replace（替换路径/版本）、exclude（排除版本）、retract（撤回版本）

### 三、面试高频问题 ⭐

**Q1：MVS 和 npm 的版本选择有什么区别？**
> MVS 选最小满足版本，npm 默认选最新兼容版本。MVS 保证确定性：同一项目每次构建产出相同依赖图。npm 需要 lock 文件辅助。

**Q2：replace 指令的用途？**
> 本地开发调试（replace → 本地路径）、使用 fork 版本、临时修复上游 bug。

### 四、总结（速记版）

1. MVS = 最小满足版本，确定性构建
2. go.mod 声明依赖，go.sum 校验哈希防篡改
3. replace 用于本地开发 / fork 替换

---

## 6.2 错误处理 ⭐

### 一、背景（为什么需要）

Go 的 error 是一个接口，不是异常。Go 1.13 引入错误包装链（`%w` / `errors.Is` / `errors.As`），改变了错误处理模式。

### 二、核心概念（是什么）

```go
// 错误包装
return fmt.Errorf("getUser %d: %w", id, err)  // %w 保留错误链

// 沿链判断
errors.Is(err, sql.ErrNoRows)  // true 如果链中任何 err == sql.ErrNoRows

// 沿链提取
var nf *NotFoundError
errors.As(err, &nf)            // true 如果链中有 *NotFoundError

// ⚠️ 用 %v 会断开错误链！
```

### 三、底层原理（重点）

`fmt.Errorf("%w", err)` 创建 `wrapError{msg, err}`，内部保存了对原 error 的引用。`errors.Is` 沿链 unwrap 比较，`errors.As` 沿链 unwrap 检查类型。

### 四、关键设计（为什么这样设计）

- **errors are values**：没有堆栈跟踪，轻量，性能好
- **错误包装**：保留上下文信息的同时不丢失原始错误
- **显式错误处理**：每个调用点都要处理，没有隐式传播

### 五、面试高频问题 ⭐

**Q1：%w 和 %v 的区别？**
> `%w` 包装错误形成错误链（errors.Is/As 可沿链查找），`%v` 只格式化字符串，断开了错误链。

**Q2：errors.Is 和 == 有什么区别？**
> `==` 只比较最外层错误，`errors.Is` 沿错误链递归比较每一层。

### 六、总结（速记版）

1. `%w` 包装保留错误链，`%v` 断开错误链
2. `errors.Is` 沿链判断（替代 `==`），`errors.As` 沿链提取
3. 错误是值，处理是显式的，性能好但需要手动处理

---

## 6.3 测试与基准测试 ⭐⭐

### 一、核心概念

**Table-Driven Tests（表驱动测试）：** Go 测试的标准范式。

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name string
        a, b, want int
    }{
        {"positive", 1, 2, 3},
        {"negative", -1, -2, -3},
        {"zero", 0, 0, 0},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {  // 子测试
            got := Add(tt.a, tt.b)
            if got != tt.want {
                t.Errorf("Add(%d,%d) = %d, want %d", tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

**测试规范要点：**
- 测试文件命名：`xxx_test.go`
- 函数签名：`func TestXxx(t *testing.T)`
- `t.Run()` 创建子测试（支持单独运行 `go test -run TestAdd/positive`）
- `t.Parallel()` 标记可并行测试
- `TestMain(m *testing.M)` 全局 setup/teardown

**Benchmark 正确写法：**
```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {  // b.N 由框架自动调整
        Add(1, 2)
    }
}
// go test -bench=. -benchmem
// -benchmem 显示内存分配情况
// -count=5 多次运行取平均
```

**Benchmark 常见陷阱 ⚠️：**
- 编译器优化可能消除无用代码 → 结果偏小
- 使用 `b.ResetTimer()` 排除初始化时间
- 使用 `b.StopTimer()` / `b.StartTimer()` 排除中间处理

### 二、面试高频问题

**Q1：Table-Driven Tests 的优势？**
> 添加新测试用例只需在表中加一行；每个用例有名字方便定位；`t.Run` 支持子测试独立运行和并行。

**Q2：Benchmark 中 b.N 是什么？**
> 框架自动调整的迭代次数，目标是让测试函数运行足够长时间（默认 1s）以获得稳定测量。


## 6.4 Race Detector（竞态检测）⭐⭐⭐

### 一、背景（为什么需要）

数据竞争（data race）是并发编程最难排查的 bug。Go 内置了基于 **Thread Sanitizer** 的竞态检测器，是面试和生产的必备工具。

### 二、核心概念（是什么）

```bash
go test -race ./...
go build -race ./...
go run -race main.go
```

**检测器会在运行时：**
- 追踪每次内存访问的 goroutine ID + 时间戳
- 检测两个 goroutine 是否同时访问同一内存地址，且至少一个是**写操作**
- 发现竞争时打印详细栈信息

### 三、底层原理（重点）

Race Detector 基于 happens-before 关系：
- 如果两次内存访问之间没有 happens-before 关系，且至少一个是写 → 报告竞争
- 通过编译期**插桩**（instrumentation）在每个内存访问前后插入检查代码
- 维护每个内存地址的访问历史（"vector clock"）

**性能代价：** 内存增加 5-10x，CPU 增加 2-20x。**只能在测试/ staging 环境使用，不能在生产中开启。**

### 四、面试高频问题 ⭐

**Q1：Race Detector 的工作原理？**
> 编译期插桩，在每个内存访问点记录 goroutine + 时间戳。检测到两个 goroutine 并发访问同一地址且至少一写时报告竞争。

**Q2：Race Detector 能在生产环境用吗？**
> 不能。内存 5-10x、CPU 2-20x 的开销，仅限测试/staging 环境。

**Q3：Race Detector 能检测出所有数据竞争吗？**
> 不能。只检测测试执行路径上的竞争。未执行到的代码路径不会检测。因此需要高覆盖率的测试配合。

```go
// 典型检测结果：
// WARNING: DATA RACE
// Write at 0x00c000... by goroutine 7:
//   main.main.func1() at main.go:15
// Previous read at 0x00c000... by goroutine 6:
//   main.main.func2() at main.go:20
```


## 6.5 Functional Options 模式 ⭐🔥

### 一、背景（为什么需要）

Go 不支持函数参数默认值，也不支持函数重载。当一个结构体有很多可选配置时，传统的做法（直接传参 / Config struct）都有缺点。Functional Options 是 Go 社区公认的最佳实践，面试高频。

### 二、核心概念

```go
// 类型定义：Option 是一个函数类型，接收指针修改配置
type Option func(*Server)

// 每一个 Option 是一个"修饰器函数"
func WithTimeout(d time.Duration) Option {
    return func(s *Server) {
        s.timeout = d
    }
}

func WithPort(port int) Option {
    return func(s *Server) {
        s.port = port
    }
}

// 构造函数接收可变参数 Option
func NewServer(addr string, opts ...Option) *Server {
    s := &Server{
        addr:    addr,
        port:    8080,            // 默认值
        timeout: 30 * time.Second, // 默认值
    }
    for _, opt := range opts {
        opt(s) // 依次应用每个 Option
    }
    return s
}

// 调用：清晰、可扩展
s := NewServer("localhost",
    WithTimeout(10*time.Second),
    WithPort(9090),
)
```

### 三、设计要点

- **Option 是 `func(*T)` 类型**：接收指针用于原地修改
- **默认值在构造函数中设置**：Option 只覆盖需要的部分
- **每个 With 函数返回一个 Option 闭包**：闭包捕获配置参数
- **向后兼容**：新增配置项只需新增 With 函数，已有调用不受影响

### 四、面试高频问题

**Q1：Functional Options 解决了什么问题？**
> Go 没有参数默认值和函数重载。方案对比：
> - 直接传多参数：参数多时不可读，无法跳过中间参数
> - Config struct：零值语义不清晰（0 是"不设置"还是"设置为 0"？）
> - Functional Options：默认值清晰 + 可选配置灵活 + 向后兼容

**Q2：Functional Options 和 Builder 模式有什么区别？**
> Builder 模式侧重"分步构建"（复杂构建过程），Functional Options 侧重"可选配置"（如 GRPC DialOption、Uber Zap Logger）。Go 中 Functional Options 更常见，它更简洁、符合 Go 风格。

**Q3：什么场景不适合 Functional Options？**
> 必选参数不应该用 Option（应该在构造函数中直接要求）。如果构造过程有复杂的依赖/顺序/校验逻辑，Builder 模式更合适。

### 五、总结

1. Option = `func(*T)`，每个 WithXxx 返回 Option 闭包
2. 默认值在构造函数中设置，Option 覆盖默认值
3. 是 GRPC、Uber Zap、AWS SDK 等主流库的标准配置模式
4. Go 无默认参数 → Functional Options 是 Go 社区最优解


## 6.6 go generate 与代码生成 ⭐

### 一、核心概念

`go generate` 是 Go 内置的代码生成机制，用于在编译前自动生成代码（mock、枚举、protobuf 等）。

```go
//go:generate stringer -type=Status        // 生成 String() 方法
//go:generate mockgen -source=iface.go -destination=mock.go
//go:generate protoc --go_out=. proto/*.proto
```

```bash
go generate ./...  # 执行所有 //go:generate 指令
```

**常见应用：** stringer（枚举→String）、mockgen（接口→Mock）、protobuf、Wire（DI）

### 二、关键设计

- **不是 `go build` 的一部分**：必须手动或 CI 中运行
- **指令顺序无关**：`go generate` 不保证生成顺序
- **输出文件通常有 `_gen.go` 后缀**：表明是生成代码，手工不修改


## 6.7 pprof 实战：火焰图解读与排查案例 ⭐⭐

### 一、核心命令

```bash
# 开启 pprof 端点
import _ "net/http/pprof"  // 自动注册到 DefaultServeMux
// 或 http.ListenAndServe(":6060", nil)

go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30  # CPU
go tool pprof http://localhost:6060/debug/pprof/heap                 # 内存
go tool pprof http://localhost:6060/debug/pprof/goroutine            # goroutine
go tool pprof http://localhost:6060/debug/pprof/mutex                # 锁竞争
go tool pprof http://localhost:6060/debug/pprof/block                # 阻塞
```

### 二、火焰图解读（关键技能）

```
pprof 交互模式常用命令：
  top10         # CPU/内存占用前 10 的函数
  list funcName # 查看函数每一行的开销
  web           # 生成调用图（需要 graphviz）
  peek funcName # 查看函数的调用者和被调用者
  pdf           # 生成 PDF 火焰图

火焰图中：
  - X 轴宽度 = CPU 占比
  - Y 轴高度 = 调用栈深度
  - 最底部 = 程序入口（main / runtime）
  - 宽且矮的"平顶山" = CPU 热点
```

### 三、排查案例

**案例 1：goroutine 泄漏**
```
goroutine profile:
  → 10000 goroutine 都阻塞在同一个 channel 操作上
  → 检查代码：生产者 goroutine 已退出，channel 未 close，消费者永久阻塞
  → 修复：生产者在 defer 中 close(channel)
```

**案例 2：内存泄漏**
```
heap profile:
  → inuse_space 持续增长
  → 某个 slice 的 cap 远大于 len（截取大数组未拷贝）
  → 修复：copy(dst, src[:n]) 显式拷贝
```

**案例 3：CPU 热点**
```
CPU profile:
  → runtime.mallocgc 占用 60%（频繁分配）
  → 修复：sync.Pool 复用对象 / 预分配容量 / 减少 []byte↔string 互转
```

---

## 6.8 线上问题排查实战 ⭐🔥🔥

> 面试高频场景题："线上 CPU 100% 怎么排查？""内存一直涨怎么定位？""接口突然变慢怎么分析？"

### 场景一：CPU 100% / CPU 飙升

**现象：** 服务 CPU 持续 100%，请求延迟暴增，但进程未 OOM。

**排查路径（5 步法）：**

```
步骤 1：确认可疑进程
  top -p $(pgrep myapp)  → 确认哪个进程 CPU 高

步骤 2：pprof CPU profile（30s 采样）
  curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.prof
  go tool pprof -http=:8080 cpu.prof   # 火焰图

步骤 3：定位热点函数
  pprof → top10
  常见热点：
    - runtime.mallocgc 占 >50% → 频繁内存分配（逃逸多）
    - runtime.gcDrain → GC 标记压力大（对象太多）
    - runtime.findObject → GC 扫描慢（大堆）
    - regexp.(*Regexp).Find → 正则未预编译
    - encoding/json.Marshal → JSON 序列化热点
    - runtime.casgstatus → 锁竞争/高频 goroutine 切换

步骤 4：根据热点下钻
  # 看具体哪一行代码在分配
  go tool pprof -alloc_space http://localhost:6060/debug/pprof/heap
  (pprof) list hotFunc

步骤 5：验证修复
  - 减少分配：sync.Pool 复用 / 预分配 slice / 字符串拼接改用 strings.Builder
  - 减少 GC：增大 GOGC / 设置 GOMEMLIMIT
  - 优化正则：regexp.Compile 提前编译
  - 优化 JSON：使用 json.RawMessage / easyjson / sonic
```

**Go 代码层面常见 CPU 热点及修复：**

| 热点 | 根因 | 修复 |
|------|------|------|
| `runtime.mallocgc` 占比高 | 堆分配过多 | 减少逃逸、sync.Pool、预分配 |
| `runtime.gcBgMarkWorker` 占比高 | GC 频繁 | 增大 GOGC、减少分配 |
| `runtime.findrunnable` 占比高 | 调度不饱和 | 检查 goroutine 泄漏/阻塞 |
| `runtime.lock2` / `runtime.unlock2` | 锁竞争严重 | 减少锁粒度、分段锁、atomic 替代 |
| `syscall.Syscall` 占比高 | 系统调用过多 | 批量 IO、buffer 优化 |

---

### 场景二：内存泄漏 / 内存持续增长

**现象：** 进程 RSS 持续增长，最终 OOM Kill。服务可能不 Crash（有 GC），但内存曲线只涨不跌。

**排查路径（6 步法）：**

```
步骤 1：确认是否是真的泄漏
  # 看 GC 后内存是否回落
  curl http://localhost:6060/debug/pprof/heap > heap1.prof
  sleep 30
  curl http://localhost:6060/debug/pprof/heap > heap2.prof
  go tool pprof -base=heap1.prof heap2.prof  # 看增量

步骤 2：看 inuse 内存分布
  go tool pprof -inuse_space http://localhost:6060/debug/pprof/heap
  (pprof) top10

步骤 3：追踪分配栈
  go tool pprof -alloc_space http://localhost:6060/debug/pprof/heap
  (pprof) list leakFunc

步骤 4：Goroutine 泄漏排查（最常见的内存泄漏原因）
  curl http://localhost:6060/debug/pprof/goroutine?debug=2 | head -50
  # 看 goroutine 数量是否持续增长
  # 大量 goroutine 阻塞在同一位置 → goroutine 泄漏

步骤 5：检查 goroutine profile 中的阻塞点
  go tool pprof http://localhost:6060/debug/pprof/goroutine
  # 看哪个函数的 goroutine 数远大于预期

步骤 6：GODEBUG 辅助分析
  GODEBUG=gctrace=1 ./myapp
  # 观察：GC 后 heap 是否持续增长
  # gc 1 @0.1s: ... 4->4->4 MB  ← 正常
  # gc 100 @60s: ... 100->100->100 MB ← 持续增长，有问题
```

**Go 内存泄漏的 4 大典型根因：**

| 根因 | 特征 | 代码特征 |
|------|------|---------|
| **Goroutine 泄漏** | goroutine 数持续增长 | channel 未关闭、select 无 ctx.Done() |
| **Slice 截取大数组** | heap profile 显示大底层数组 | `bigSlice[:10]` 未 copy |
| **全局 map/slice 只增不减** | `inuse_space` 中 map 持续增长 | map 无淘汰/过期机制 |
| **time.Ticker/time.After 未 Stop** | 大量 timer 堆积 | 循环中 `time.After` |

---

### 场景三：接口延迟突增 / 慢查询

**现象：** P99/P999 延迟突然从 50ms 涨到 2s，CPU 和内存未见异常。

**排查路径（4 步法）：**

```
步骤 1：profile 对比（基准 vs 异常）
  # 在延迟正常时和异常时各抓一次
  go tool pprof -base=normal.prof slow.prof

步骤 2：锁竞争检查
  go tool pprof http://localhost:6060/debug/pprof/mutex
  # sync.(*Mutex).Lock 占比高 → 锁竞争导致排队等待

步骤 3：阻塞检查
  go tool pprof http://localhost:6060/debug/pprof/block
  # channel send/recv 阻塞在多处 → 生产者消费者速度不匹配

步骤 4：调度器诊断
  GODEBUG=schedtrace=1000 ./myapp
  # idleprocs=0 且全局队列持续有值 → 过载
  # spinning 线程很多 → 大量 M 空闲但 P 不够
```

**常见原因与修复：**

| 原因 | 特征 | 修复 |
|------|------|------|
| GC STW | GC pause 偶尔尖刺 | 减少分配、调 GOGC |
| 锁竞争 | mutex profile 有热点锁 | 减小锁粒度、分段锁、atomic |
| 连接池耗尽 | `net/http` 连接建立延迟高 | 调大 `MaxIdleConnsPerHost` |
| 下游超时 | 调用链远端延迟高 | 设合理的超时、熔断降级 |
| Goroutine 饿死 | `schedtrace` 显示大量 runnable G | 检查是否有 CPU 密集死循环 |

---

### 排查工具全景图

| 症状 | 第一工具 | 第二工具 | 第三工具 |
|------|---------|---------|---------|
| CPU 高 | `pprof/profile` | `pprof/heap -alloc` | `perf top` (系统级) |
| 内存高 | `pprof/heap -inuse` | `pprof/goroutine` | `GODEBUG=gctrace=1` |
| 延迟高 | `pprof/mutex` | `pprof/block` | `GODEBUG=schedtrace=1000` |
| Goroutine 多 | `pprof/goroutine` | `GODEBUG=scheddetail=1` | `/debug/pprof/goroutine?debug=2` |
| 偶发 Crash | `-race` 测试 | `delve` 调试 | core dump |

---

# 第七章：并发模式 ⭐🔥

## 7.1 Go 原生并发模式

### Pipeline（管道模式）
```go
// channel 串联：gen → sq → print
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() { for _, n := range nums { out <- n }; close(out) }()
    return out
}
func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func() { for n := range in { out <- n * n }; close(out) }()
    return out
}
```

### Fan-Out / Fan-In 🔥
```go
// Fan-Out: 一个输入 → 多个 worker
// Fan-In: 多个 worker 输出 → 一个 channel
func fanIn(chs ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    for _, ch := range chs {
        wg.Add(1)
        go func(c <-chan int) { defer wg.Done(); for v := range c { out <- v } }(ch)
    }
    go func() { wg.Wait(); close(out) }()
    return out
}
```

### Or-Done 模式
```go
// 优雅监听多个 channel 的取消
func or(chs ...<-chan struct{}) <-chan struct{} {
    switch len(chs) {
    case 0: return nil
    case 1: return chs[0]
    }
    done := make(chan struct{})
    go func() {
        defer close(done)
        select {
        case <-chs[0]: case <-chs[1]:
        case <-or(append(chs[2:], done)...):
        }
    }()
    return done
}
```

### Worker Pool
```go
func workerPool(n int, jobs <-chan Job, results chan<- Result) {
    var wg sync.WaitGroup
    for i := 0; i < n; i++ {
        wg.Add(1)
        go func() { defer wg.Done(); for j := range jobs { results <- process(j) } }()
    }
    go func() { wg.Wait(); close(results) }()
}
```

---

# 第八章：并发场景实战排查 🔥🔥🔥

> **目标**：不仅理解原理，更要能在生产环境中快速定位和解决并发问题。每个场景包含：问题现象 → 根因分析 → 排查路径 → 修复方案。

---

## 8.1 死锁（Deadlock）⭐🔥🔥

### 场景一：channel 互相等待

```go
// ❌ 死锁：两个 goroutine 互相等待对方
func main() {
    ch := make(chan int)
    ch <- 1      // 无人接收 → 永久阻塞 → fatal error: all goroutines are asleep - deadlock!
    <-ch
}
```

**现象：** `fatal error: all goroutines are asleep - deadlock!` — runtime 检测到所有 goroutine 都在阻塞状态。

**排查路径：**
1. 看 panic 栈信息 → 定位阻塞的 goroutine 和代码行
2. 检查 channel 的 send/recv 配对关系
3. 检查是否有 goroutine 提前退出、未执行到对应的 recv/send
4. `GODEBUG=schedtrace=1000` 观察 goroutine 状态

**常见死锁模式：**
```go
// 模式1：单个 goroutine 对自己 send（无缓冲 channel 自阻塞）
ch := make(chan int)
ch <- 1 // ❌ 无人接收，自己阻塞

// 模式2：循环依赖
// goroutine A 等 B 的 channel，B 等 A 的 channel

// 模式3：Mutex 锁顺序不一致
// goroutine A: Lock(m1) → Lock(m2)
// goroutine B: Lock(m2) → Lock(m1)  // ❌ 交叉等锁
```

### 场景二：Mutex 死锁

```go
var mu sync.Mutex

func f() {
    mu.Lock()
    defer mu.Unlock()
    f() // ❌ 递归调用，重入已持有的锁 → 死锁！
}
// Go 的 Mutex 不可重入（非递归锁）
```

**为什么 Go Mutex 不可重入？** 设计简洁性。可重入锁增加了计数器和所有者追踪的复杂度，且容易掩盖设计问题。如果需要可重入，说明锁粒度或设计需要调整。

**排查 Mutex 死锁：**
```bash
# pprof 查看锁竞争
go tool pprof http://localhost:6060/debug/pprof/mutex
# 查看 goroutine 阻塞栈
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

---

## 8.2 Goroutine 泄漏 ⭐🔥🔥

### 场景一：channel 无人接收/无人发送

```go
// ❌ 泄漏1：发送者阻塞在无人接收的 channel
func leak() {
    ch := make(chan int)
    go func() { ch <- 1 }() // goroutine 永久阻塞，永远不退出
    // 忘记接收 <-ch
}

// ❌ 泄漏2：接收者阻塞在无人发送/未关闭的 channel
func leak() {
    ch := make(chan int)
    go func() {
        for v := range ch { // 等待数据或 close，永远等不到
            _ = v
        }
    }()
    // 忘记 close(ch) 且没有发送数据
}
```

### 场景二：select 无退出条件

```go
// ❌ select 永远等不到退出信号
func worker(ch <-chan int) {
    for {
        select {
        case v := <-ch:
            process(v)
        // ❌ 缺少 ctx.Done() case 或 timeout
        }
    }
}
```

### 场景三：time.After 在循环中泄漏 Timer

```go
// ❌ 每次循环创建新 timer，频繁触发时 timer 堆积
for {
    select {
    case <-ch:
    case <-time.After(5 * time.Second): // 旧 timer 未 GC 前累积
    }
}
```

### 排查路径：

**第一步：pprof goroutine profile**
```bash
curl http://localhost:6060/debug/pprof/goroutine?debug=2 | head -100
# 或
go tool pprof http://localhost:6060/debug/pprof/goroutine
```
观察 goroutine 数量趋势和阻塞点。

**第二步：分析 goroutine 栈**
```
goroutine 100 [chan send, 10 minutes]:
main.leakFunc(0xc0000a6000)
    /app/main.go:42 +0x45
// 👆 阻塞在 channel send 10 分钟 → 大概率泄漏
```

**第三步：使用 runtime 指标监控**
```go
fmt.Println(runtime.NumGoroutine())      // 当前 goroutine 总数
// 持续增长 → goroutine 泄漏
```

**修复模式：**
```go
// ✅ 方案1：用 context 控制退出
func worker(ctx context.Context, ch <-chan int) {
    for {
        select {
        case v := <-ch:
            process(v)
        case <-ctx.Done():
            return // 优雅退出
        }
    }
}

// ✅ 方案2：channel 关闭 + range 自动退出
func worker(ch <-chan int) {
    for v := range ch { // ch 关闭后自动退出
        process(v)
    }
}

// ✅ 方案3：复用 timer
timer := time.NewTimer(5 * time.Second)
defer timer.Stop() // 关键：防止 timer 泄漏！
for {
    timer.Reset(5 * time.Second)
    select {
    case <-ch:
    case <-timer.C:
    }
}
```

---

## 8.3 Channel 阻塞问题 ⭐🔥

### 场景一：无缓冲 channel 的同步阻塞

```go
// 无缓冲 channel 的"挂起等待"不是 bug，是设计行为
ch := make(chan int)
go func() {
    result := compute()  // 假设耗时 5s
    ch <- result         // 发送者等接收者就绪
}()
val := <-ch // ✅ 正常同步等待，这不是阻塞问题
```

### 场景二：真正的阻塞问题 —— 发送/接收速度不匹配

```go
// ❌ 问题：生产者快于消费者，channel 满后生产者阻塞
func fastProducer(ch chan<- int) {
    for i := 0; i < 1000000; i++ {
        ch <- i // 缓冲满了之后一直阻塞
    }
}

func slowConsumer(ch <-chan int) {
    for v := range ch {
        time.Sleep(100 * time.Millisecond) // 处理慢
        _ = v
    }
}
```

**排查路径：**
1. pprof goroutine → 大量 goroutine 阻塞在 `chansend` 或 `chanrecv`
2. GODEBUG schedtrace → 大量 goroutine 在 waiting 状态
3. 业务表现：请求超时 → 上游调用方也阻塞 → 级联故障

**修复方案：**
```go
// ✅ 方案1：有界队列 + 拒绝策略
select {
case ch <- v:
default:
    // 记录 metric，丢弃或返回错误
    metrics.DroppedMessages.Inc()
}

// ✅ 方案2：消费者动态扩缩容
// ✅ 方案3：异步写入 + 持久化队列（如 Kafka）
```

### 场景三：nil channel 的使用场景

```go
// nil channel 的 select case 永远不触发 → 可以"禁用"某个分支
var timeoutCh <-chan time.Time // nil
select {
case v := <-dataCh:
    process(v)
    timeoutCh = nil // 收到数据后禁用超时（如果不需要了）
case <-timeoutCh:   // nil 时永远不会触发
    handleTimeout()
}
```

---

## 8.4 Map 并发 Panic ⭐🔥

### 场景：并发读写 map

```go
// ❌ 经典并发 panic
var m = make(map[string]int)

func read()  { _ = m["key"] }      // goroutine 1
func write() { m["key"] = 1 }      // goroutine 2
// fatal error: concurrent map read and map write
// ⚠️ 注意：并发读 + 并发读 是安全的！
```

**现象：** `fatal error: concurrent map read and map write` 或 `concurrent map writes`，运行时立即 crash，无法 recover。

**为什么不能 recover？** 这是 `fatal error`（Go runtime 的 `throw()`），不是 `panic`。`recover()` 无法捕获。Runtime 检测到 map 内部的 `flags` 字段有并发标志位冲突时直接 throw。

### 排查路径：

**第一步：看 panic 栈**
```
fatal error: concurrent map read and map write

goroutine 12 [running]:
runtime.throw(...)
    runtime/panic.go:...
runtime.mapaccess1_faststr(...)
    runtime/map_faststr.go:...
main.read(...)
    /app/main.go:15
```

**第二步：找出所有并发访问点**
- 检查是否在 goroutine 中直接读写同一个 map
- 检查 HTTP handler 中（每个请求一个 goroutine）是否共享了 map 变量
- 检查 Range map 时是否有 goroutine 在修改 map

**第三步：用 race detector 复现**
```bash
go test -race -run TestMapConcurrent
go run -race main.go
```

**修复方案对比：**

| 方案 | 适用场景 | 读性能 | 写性能 |
|------|---------|--------|--------|
| `sync.RWMutex` | 读多写少，通用 | 好（并发读无锁） | 一般（互斥） |
| `sync.Map` | 读多写少，key 集合稳定 | 好（无锁读） | 差（写新 key 需加锁） |
| 分段锁 | 高并发写 | 较好 | 好（分段并行） |
| Channel + 单 goroutine | 操作序列化需求 | 一般 | 一般 |

```go
// ✅ 方案1：sync.RWMutex（最通用）
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}
func (s *SafeMap) Get(key string) int {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return s.m[key]
}
func (s *SafeMap) Set(key string, val int) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.m[key] = val
}

// ✅ 方案2：sync.Map（读多写少）
var m sync.Map
m.Store("key", 1)
v, ok := m.Load("key")
m.Range(func(k, v interface{}) bool { /* ... */; return true })
```

### 场景进阶：定时清理 map 时的并发安全

```go
// ❌ 危险：清理 goroutine 遍历 map + 业务 goroutine 写入
go func() {
    for {
        time.Sleep(1 * time.Minute)
        mu.Lock()
        for k, v := range m {  // 遍历中如果有时间窗口...
            if expired(v) { delete(m, k) }
        }
        mu.Unlock()
    }
}()
// 看似有锁就安全？不够 — 要注意清理任务可能阻塞业务写入

// ✅ 改进：分批清理 + 短暂加锁
// ✅ 或使用 sync.Map.Range + Delete（原子操作）
```

---

## 8.5 其他高频并发 Bug ⭐

### WaitGroup 使用陷阱

```go
// ❌ 陷阱1：Add 放在 goroutine 内部
var wg sync.WaitGroup
for i := 0; i < 10; i++ {
    go func() {
        wg.Add(1)  // ❌ Add 可能在 Wait 之后才执行！
        defer wg.Done()
        work()
    }()
}
wg.Wait()

// ✅ 正确：Add 在 go 之前
for i := 0; i < 10; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        work()
    }()
}
wg.Wait()

// ❌ 陷阱2：WaitGroup 值拷贝
func f(wg sync.WaitGroup) { // ❌ 拷贝了！内部计数器独立
    wg.Done()
}
// ✅ 必须传指针：func f(wg *sync.WaitGroup)
```

### 闭包变量捕获

```go
// ❌ 经典陷阱
for i := 0; i < 10; i++ {
    go func() {
        fmt.Println(i) // 全部打印 10（或随机值）
    }()
}

// ✅ 修复（Go 1.22 前）
for i := 0; i < 10; i++ {
    i := i // 创建新变量遮蔽
    go func() { fmt.Println(i) }()
}
// Go 1.22+：for range 循环变量每次迭代已自动为新变量
```

### 发送到已关闭的 channel

```go
// ❌ panic: send on closed channel
ch := make(chan int)
close(ch)
ch <- 1 // panic!

// ✅ 防护：在发送方持有 close 权限
// 原则：只在唯一的发送方 goroutine 中 close channel
// 多发送方场景：用 sync.Once 包装 close，或用专门的 "协调者" goroutine
```

---

## 8.6 并发排查工具速查

| 工具 | 用途 | 命令 |
|------|------|------|
| Race Detector | 数据竞争检测 | `go test -race` / `go run -race` |
| pprof goroutine | goroutine 泄漏/阻塞 | `go tool pprof /debug/pprof/goroutine` |
| pprof mutex | 锁竞争分析 | `go tool pprof /debug/pprof/mutex` |
| pprof block | 阻塞分析 | `go tool pprof /debug/pprof/block` |
| GODEBUG schedtrace | 调度器状态 | `GODEBUG=schedtrace=1000` |
| GODEBUG scheddetail | 调度器详情 | `GODEBUG=scheddetail=1,schedtrace=1000` |
| runtime.NumGoroutine | goroutine 数量监控 | 代码内嵌 metrics |
| Delve | 调试器 | `dlv debug ./main.go` |

---

## 8.7 经典面试编码题：交替打印 ⭐🔥🔥

> 场景：面试官要求用 Go 实现 N 个 goroutine 按顺序交替打印。这是高频手写代码题，考察对 channel / sync 原语的理解深度和编码熟练度。

### 题型一：两个 goroutine 交替打印数字（奇偶）

**题目：** 两个 goroutine，一个打印奇数，一个打印偶数，交替输出 1, 2, 3, ..., 100。

**方案一：无缓冲 channel（最直观）**

```go
func printOddEven() {
    chOdd := make(chan struct{})
    chEven := make(chan struct{})
    var wg sync.WaitGroup
    wg.Add(2)

    // goroutine 1: 打印奇数
    go func() {
        defer wg.Done()
        for i := 1; i <= 100; i += 2 {
            <-chOdd                     // 等许可
            fmt.Println("g1:", i)
            chEven <- struct{}{}        // 通知偶数
        }
        <-chOdd // 收掉最后一次通知（防止 goroutine 泄漏）
    }()

    // goroutine 2: 打印偶数
    go func() {
        defer wg.Done()
        for i := 2; i <= 100; i += 2 {
            <-chEven
            fmt.Println("g2:", i)
            chOdd <- struct{}{}
        }
    }()

    chOdd <- struct{}{} // 启动！
    wg.Wait()
}
```

**方案二：有缓冲 channel（信号量）**

```go
func printOddEvenWithSem() {
    sem := make(chan struct{}, 1) // 容量=1，类似信号量
    done := make(chan struct{})
    sem <- struct{}{} // 初始放一个 token

    go func() {
        for i := 1; i <= 100; i += 2 {
            <-sem
            fmt.Println("g1:", i)
            sem <- struct{}{}
        }
    }()

    go func() {
        for i := 2; i <= 100; i += 2 {
            <-sem
            fmt.Println("g2:", i)
            sem <- struct{}{}
        }
        close(done)
    }()

    <-done
}
```

---

### 题型二：两个 goroutine 交替打印字母和数字

**题目：** g1 打印 A, B, C, D, E，g2 打印 1, 2, 3, 4, 5。交替输出 A1B2C3D4E5。

```go
func printLetterNumber() {
    letters := []string{"A", "B", "C", "D", "E"}
    nums := []int{1, 2, 3, 4, 5}

    chLetter := make(chan struct{})
    chNum := make(chan struct{})
    var wg sync.WaitGroup
    wg.Add(2)

    // g1: 打印字母
    go func() {
        defer wg.Done()
        for _, l := range letters {
            <-chLetter
            fmt.Print(l)
            chNum <- struct{}{}
        }
    }()

    // g2: 打印数字
    go func() {
        defer wg.Done()
        for _, n := range nums {
            <-chNum
            fmt.Print(n)
            if n < len(nums) { // 注意：最后一轮不需要再通知字母 goroutine
                chLetter <- struct{}{}
            }
        }
    }()

    chLetter <- struct{}{} // 启动
    wg.Wait()
    fmt.Println()
}
// 输出: A1B2C3D4E5
```

**追问：上面代码中为什么最后一轮不通知字母 goroutine？**
> 因为字母共 5 个，数字也 5 个。最后一轮 g2 打印 "5" 后，g1 已经完成了第 5 个字母 "E"，不需要再等通知。如果多余地 `chLetter <- struct{}{}`，g1 已经退出，send 会永久阻塞 → goroutine 泄漏。

---

### 题型三：N 个 goroutine 按顺序交替打印 ⭐🔥🔥

**题目：** 启动 4 个 goroutine，每个 goroutine 只打印自己的字符。要求按顺序交替输出：A1B2C3D4 A1B2C3D4 ...（共 10 轮）。

**方案一：多个 channel 接力（最易理解）**

```go
func nGoroutinesPrint() {
    const N = 4
    chs := make([]chan struct{}, N)
    for i := range chs {
        chs[i] = make(chan struct{})
    }
    chars := []string{"A", "1", "B", "2"}

    for i := 0; i < N; i++ {
        go func(id int) {
            for round := 0; round < 10; round++ {
                <-chs[id]                        // 等前一个通知
                fmt.Print(chars[id])
                chs[(id+1)%N] <- struct{}{}      // 通知下一个
            }
        }(i)
    }

    chs[0] <- struct{}{} // 启动 g0
    time.Sleep(time.Second) // 等待完成（生产用 WaitGroup）
    fmt.Println()
}
```

**方案二：sync.Cond 实现（更灵活）**

```go
func nGoroutinesWithCond() {
    const N = 4
    var mu sync.Mutex
    cond := sync.NewCond(&mu)
    turn := 0 // 当前该谁执行
    chars := []string{"A", "1", "B", "2"}

    for i := 0; i < N; i++ {
        go func(id int) {
            for round := 0; round < 10; round++ {
                mu.Lock()
                for turn != id { // ⚠️ 必须用 for 而不是 if
                    cond.Wait()
                }
                fmt.Print(chars[id])
                turn = (turn + 1) % N
                mu.Unlock()
                cond.Broadcast() // 通知所有等待者
            }
        }(i)
    }

    time.Sleep(time.Second)
    fmt.Println()
}
```

**方案三：单个 channel + 原子计数器（最简洁）**

```go
func nGoroutinesWithCounter() {
    const N = 4
    ch := make(chan struct{}, 1)
    var counter atomic.Int64
    chars := []string{"A", "1", "B", "2"}

    for i := 0; i < N; i++ {
        go func(id int) {
            for round := 0; round < 10; round++ {
                for {
                    if int(counter.Load())%N == id { // 轮到自己
                        <-ch
                        fmt.Print(chars[id])
                        counter.Add(1)
                        ch <- struct{}{}
                        break
                    }
                    // 不是自己的轮次 → 短暂让出 CPU
                    // ⚠️ 这里实际上有忙等，生产应改用 channel 通知
                }
            }
        }(i)
    }

    ch <- struct{}{}
    time.Sleep(time.Second)
    fmt.Println()
}
```

---

### 题型四：同步输出 + 控制超时退出 🔥

**题目：** 两个 goroutine 交替打印，同时支持 3 秒超时退出整个流程。

```go
func printWithTimeout() {
    ch1 := make(chan struct{})
    ch2 := make(chan struct{})
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    var wg sync.WaitGroup
    wg.Add(2)

    // g1
    go func() {
        defer wg.Done()
        i := 1
        for {
            select {
            case <-ch1:
                fmt.Println("g1:", i)
                i += 2
                ch2 <- struct{}{}
            case <-ctx.Done():
                return
            }
        }
    }()

    // g2
    go func() {
        defer wg.Done()
        i := 2
        for {
            select {
            case <-ch2:
                fmt.Println("g2:", i)
                i += 2
                ch1 <- struct{}{}
            case <-ctx.Done():
                return
            }
        }
    }()

    ch1 <- struct{}{} // 启动
    wg.Wait()
    fmt.Println("done")
}
```

---

### 方案对比与面试话术

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| 无缓冲 channel | 直观易懂，语义清晰 | goroutine 多时 channel 数量多 | 2~3 个 goroutine |
| 有缓冲 channel（信号量） | 代码简洁 | 可读性稍差 | 2 个 goroutine |
| sync.Cond | 灵活，适合复杂条件 | 代码多，易出错（for vs if） | N 个 goroutine，条件复杂 |
| channel 接力链 | 天然 FIFO | channel 数量与 goroutine 数相等 | N 个 goroutine 循环 |
| 原子计数器 | 极简 | 存在忙等（spin），浪费 CPU | 轻度使用 |

**面试关键话术：**
> "在面试中，我会优先选用**无缓冲 channel** 或 **channel 接力链**方案，因为：
> 1. 代码意图直观，面试官一眼能看懂
> 2. channel 天生 FIFO 保证顺序，不需要额外维护 turn 变量
> 3. 如果需要扩展为 N 个 goroutine，改为 channel 数组循环即可
> 4. sync.Cond 适合条件更复杂的场景，但容易写错（必须用 for 而不是 if）"

---

# 第九章：面试杀手级问题 🔥🔥🔥

## 9.1 "100 万个 goroutine 会发生什么？"

**标准答案：** 技术上可行（初始栈 ~2GB）。关注点：栈扩张导致实际内存远 > 2GB；GC 扫描根对象耗时增加（每个 G 的栈都要扫描）；调度器压力（P 有限，大量 G 排队）；OS 线程数有上限。建议用 worker pool 控制并发度。

## 9.2 "goroutine 死循环会怎么样？1.14 前后区别？"

**标准答案：**
- Go 1.13 前：死循环 G 不会让出 CPU → 同 P 的其他 G 饿死（非抢占式）
- Go 1.14+：信号抢占机制 → sysmon 检测超 10ms → 发 SIGURG → G 被强制让出。但如果 GOMAXPROCS=1，抢占后又被调度，整体效率仍低。

## 9.3 "interface{} 底层结构？nil 接口等于 nil 吗？"

**标准答案（必须能背）：**
- 空接口 `interface{}` = `eface{_type, data}`
- 有方法接口 = `iface{tab, data}`
- 接口为 nil ⇔ `_type == nil && data == nil`
- `var p *T = nil; var i interface{} = p` → `i != nil`！因为 `_type = *T ≠ nil`

## 9.4 "GC 如何做到亚毫秒级 STW？"

**标准答案：**
1. 并发标记（标记阶段不 STW）
2. 混合写屏障（Go 1.8+，栈不需写屏障，两次 STW 只在初始和终止）
3. STW 时间仅与根对象数量有关，与堆大小无关
4. 并发清扫（回收白对象不是 STW）

## 9.5 "设计一个高并发 HTTP 服务"

**标准答案要点：**
1. 调大 `MaxIdleConnsPerHost`（默认 2 → 100+）
2. 超时控制（Connect/Dial/ResponseHeader 三级超时）
3. Context 传递（链路取消）
4. 全局 `http.Client` 复用连接（不每次 new）
5. 优雅关闭 `server.Shutdown(ctx)`
6. 限流（令牌桶/信号量控制并发数）
7. 熔断降级 + pprof 兜底

## 9.6 "用 channel 实现一个无锁队列"

```go
type Queue struct { ch chan int }
func NewQueue(size int) *Queue { return &Queue{ch: make(chan int, size)} }
func (q *Queue) Push(v int) bool {
    select { case q.ch <- v: return true; default: return false }
}
func (q *Queue) Pop() (int, bool) {
    select { case v := <-q.ch: return v, true; default: return 0, false }
}
// 面试加分：channel 内部有 mutex，不是真"无锁"
// 真正无锁队列需 atomic + CAS 实现
```

---

# 第十章：Go 工具链与编译 ⭐

## 10.1 常用工具命令

### go vet（静态分析）
```bash
go vet ./...  # 检测可疑代码模式：
# - 不可达代码
# - 错误的 Printf 格式字符串
# - 错误的 struct tag
# - 锁值拷贝（sync.Mutex 被复制）
# - 循环中 goroutine 捕获的变量问题（Go 1.22 后可能不需要）
# go vet 是 CI 的必须环节
```

### go fmt / goimports
```bash
go fmt ./...     # 格式化代码（标准风格）
goimports -w .   # 格式化 + 自动管理 import（增删排序）
# goimports 需单独安装: go install golang.org/x/tools/cmd/goimports@latest
```

### go mod 核心命令
```bash
go mod init <module>        # 初始化模块
go mod tidy                 # 清理未使用的依赖 + 添加缺失的依赖
go mod download             # 仅下载依赖（不修改 go.mod）
go mod verify               # 验证 go.sum 完整性
go mod why <pkg>            # 查看为什么需要某个包
go mod graph                # 打印依赖图
go mod vendor               # 创建 vendor 目录（离线构建）
```

### go.work 工作区（Go 1.18+）⭐

**一句话总结：** `go.work` 允许在本地同时开发多个模块，无需 `replace` 指令。

```bash
go work init ./module-a ./module-b  # 初始化 workspace，包含两个子模块
go work use ./module-c              # 添加新模块
go work sync                        # 同步 workspace 依赖
```

**使用场景：**
> 微服务 monorepo 中，修改共享库 (common-lib) 后不需要 push+tag+go get，直接 `go.work` 引用本地路径就能编译验证。比 `replace` 更正规：`go.work` 不会被意外提交到 git（应该在 `.gitignore` 中），不会污染 `go.mod`。

```go
// go.work 文件示例
go 1.21

use (
    ./services/api-server    // 使用本地路径
    ./libs/common-lib
    ./libs/auth-lib
)
```

### 交叉编译
```bash
GOOS=linux GOARCH=amd64 go build    # Mac → Linux
GOOS=darwin GOARCH=arm64 go build   # Linux → Mac M1
GOOS=windows GOARCH=amd64 go build  # → Windows

# 常用组合：
# GOOS: linux, darwin, windows
# GOARCH: amd64, arm64, 386
# CGO_ENABLED=0 禁用 CGO 实现纯静态链接
```

### 编译器优化相关
```bash
go build -gcflags="-m" main.go        # 逃逸分析输出
go build -gcflags="-m -m" main.go     # 更详细的逃逸分析
go build -gcflags="-S" main.go        # 输出汇编代码
go build -ldflags="-s -w"             # 去除调试信息，减小二进制体积
go build -ldflags="-X main.version=v1.0" # 注入编译期变量
```

---

## 10.2 Go 汇编快速入门 ⭐

### 为什么需要了解？
- 理解函数调用的栈帧布局
- 理解 goroutine 切换的底层机制
- pprof 分析时看懂汇编级热点

### 核心概念

**Go 使用 Plan 9 汇编风格：**
```asm
// 函数调用约定（Go 1.17+ 基于寄存器 ABI）
// - 参数通过寄存器传递：AX, BX, CX, DI, SI, R8-R15
// - 返回值在栈上（调用方预留空间）
// - 栈由低地址向高地址增长（和传统 x86 不同）

TEXT ·Add(SB), NOSPLIT, $0-16   // 函数 Add，栈帧 0 字节，参数+返回值 16 字节
    MOVQ a+0(FP), AX            // 第一个参数 a
    MOVQ b+8(FP), BX            // 第二个参数 b
    ADDQ BX, AX
    MOVQ AX, ret+16(FP)         // 返回值
    RET
```

**四组伪寄存器：**
| 伪寄存器 | 含义 |
|---------|------|
| `FP` | 帧指针 — 指向当前函数参数和返回值的起点 |
| `PC` | 程序计数器 |
| `SB` | 静态基址 — 全局符号的基地址 |
| `SP` | 栈指针 — 指向当前栈帧底部 |

### 面试高频问题

**Q1：Go 函数调用时参数和返回值在栈上怎么布局？**
> 调用方在栈上预留参数空间和返回值空间（从低地址到高地址依次是：局部变量 → 返回值 → 参数）。被调函数通过 FP 偏移访问参数和写入返回值。

**Q2：goroutine 切换在汇编层面做了什么？**
> 保存当前 G 的 PC/SP/BP 到 `g.sched` 字段 → 从目标 G 的 `g.sched` 恢复 → 切换栈指针 → 跳转。核心是 `runtime.mcall`（用户G→g0）和 `runtime.gogo`（g0→用户G）。

---

## 10.3 Go 构建与链接

### 编译流程
```
源码 (.go)
  → lexer/parser → AST
  → type check → 类型化 AST
  → SSA (Static Single Assignment) → 中间表示
  → 优化（内联、逃逸分析、死代码消除、边界检查消除）
  → 代码生成 → .o 目标文件 (.a 归档文件)
  → 链接 → 可执行文件
```

### 面试高频：Go 编译速度为什么快？
1. **依赖图明确**：import 不产生循环依赖（编译期报错），清晰 DAG
2. **只重新编译变更文件**：编译器缓存依赖分析结果
3. **导出数据（export data）**：编译 `.a` 文件时附带类型/符号信息，编译依赖方不需要重新解析源码
4. **无模板展开**：（Go 1.18 泛型用 GC Shape 共享代码，不重复编译）
5. **简单语法**：无宏、无头文件、无继承、无隐式类型转换

---

# 第十一章：补充知识

## 11.1 泛型（Go 1.18+）⭐

**一句话总结：** Go 泛型不是单态化（C++），而是通过字典（dictionary）传递类型信息，GC Shape 共享代码。

```go
func Min[T constraints.Ordered](a, b T) T {
    if a < b { return a }; return b
}
```

**实现特点：** 相同 GC Shape 的类型（如 int64 和 uint64）共享同一份编译后的代码 → 编译速度快于 C++ 模板，但非完全零开销。

## 11.2 Go 1.21+ 重要新特性 ⭐🔥

### slices 和 maps 标准库包（Go 1.21+）

告别手写泛型工具函数，标准库提供了类型安全的 slice/map 操作：

```go
import "slices"

// 常用操作
slices.Contains(s, v)              // 是否包含
slices.Index(s, v)                 // 查找索引
slices.Equal(a, b)                 // 比较两个 slice
slices.Clone(s)                    // 浅拷贝
slices.Delete(s, i, j)             // 删除区间 [i,j)
slices.Insert(s, i, v...)          // 在 i 位置插入
slices.Replace(s, i, j, v...)      // 替换区间
slices.Sort(s)                     // 排序
slices.Max(s) / slices.Min(s)      // 最大/最小值
slices.BinarySearch(s, target)     // 二分查找（返回 index, found）

// maps 包
import "maps"
maps.Clone(m)                      // 浅拷贝 map
maps.Copy(dst, src)                // 复制 map
maps.Equal(a, b)                   // 比较两个 map（key,value 都相等）
maps.DeleteFunc(m, func(k K, v V) bool { ... }) // 条件删除

// cmp 包
import "cmp"
cmp.Compare(a, b)                  // -1 / 0 / +1（泛型三路比较）
cmp.Less(a, b)                     // a < b
cmp.Or(a, b, c...)                 // 返回第一个非零值
```

### clear 内建函数（Go 1.21+）

```go
// 将 map 的所有元素清零（不释放底层内存）
m := map[string]int{"a": 1, "b": 2}
clear(m) // m 仍然是 len=0 但 bucket 内存保留
// vs 旧方法：for k := range m { delete(m, k) }

// 将 slice 所有元素归零
s := []int{1, 2, 3}
clear(s) // s = [0, 0, 0]，len/cap 不变
```

### log/slog 结构化日志（Go 1.21+）🔥

Go 终于有了标准库的结构化日志！比 `log` 包更强大，比 `zap/zerolog` 更标准：

```go
import "log/slog"

// 基础用法
slog.Info("request processed", "method", "GET", "path", "/api/users", "latency", 15*time.Millisecond)
// 输出: INFO request processed method=GET path=/api/users latency=15ms

// 结构化日志 + context
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
logger.Info("user login", slog.String("userID", "123"), slog.Duration("elapsed", time.Second))
// 输出: {"time":"...","level":"INFO","msg":"user login","userID":"123","elapsed":"1s"}

// 常用属性
slog.String("k", "v")   // 字符串
slog.Int("k", 42)        // 整数
slog.Float64("k", 3.14)  // 浮点
slog.Bool("k", true)     // 布尔
slog.Any("k", obj)       // 任意类型（用 fmt.Sprintf 序列化）
slog.Duration("k", d)    // duration
slog.Time("k", t)        // 时间
slog.Group("k", attrs...) // 嵌套组
```

**面试要点：** `slog` 的设计哲学与 `zap` 不同——slog 优先可读性和标准兼容而非极致性能。高吞吐场景仍可能选择 `zap/zerolog`，但大部分应用 slog 够用。

### Go 1.22 for range 变量语义变更

```go
// Go 1.22 之前：循环变量在每次迭代中复用
// Go 1.22 起：每次迭代自动创建新变量
for _, v := range slice {
    go func() { use(v) }() // Go 1.22+ 不再需要 v := v！
}
// ⚠️ 但注意：for i := 0; i < n; i++ 的 i 仍然是复用的！
```

### net/http ServeMux 增强（Go 1.22+）

```go
// Go 1.22 起支持 method + path 参数 + 通配符
mux := http.NewServeMux()
mux.HandleFunc("GET /users/{id}", handleGetUser)        // method + path param
mux.HandleFunc("POST /users", handleCreateUser)
mux.HandleFunc("GET /api/v1/{module}/{action}", handleV1) // 多段通配符
// 通过 r.PathValue("id") 获取路径参数
```

## 11.3 unsafe 包

```go
unsafe.Pointer  // 通用指针，GC 追踪
uintptr         // 整数，GC 不追踪 — 危险！
unsafe.Sizeof   // 类型大小
unsafe.Offsetof // 字段偏移

// ⚠️ uintptr 指向的内存可能被 GC 回收，必须保证引用链
// 正确用法：uintptr -> unsafe.Pointer -> 使用 -> 恢复
```

## 11.4 CGO 调用机制

CGO 调用开销比纯 Go 函数大数十倍：
- G 切换到 M 的 g0 系统栈
- M 锁定到该 G（不能换 P）
- 在 OS 线程栈执行 C 函数
- C 中阻塞 → M 阻塞（不能换 G）

**原则：** 尽量用纯 Go 实现或用 RPC/HTTP 通信代替 CGO。

## 11.5 额外内部知识点

| 知识点 | 一句话 |
|------|------|
| g0 栈 | 每个 M 有 g0 goroutine，运行在系统栈，执行调度/GC 等 runtime 逻辑 |
| finalizer | `runtime.SetFinalizer` 在对象被 GC 时回调，不保证时机 |
| GODEBUG | `GODEBUG=gctrace=1` GC 日志；`schedtrace=1000` 调度追踪 |

---

# 第十二章：面试连环追问链 🔥🔥🔥

**追问链 1：Channel**
```
"channel 底层是什么结构？"
→ "发送时 channel 已满怎么处理？"
→ "有缓冲 channel 接收时，如果 sendq 有等待者怎么处理？"
→ "select 的 case 为什么是随机的？怎么实现的？"
→ "close 后的 channel 还能读到什么？多次 close 会怎样？"
```

**追问链 2：GMP 调度**
```
"go func() 创建后怎么被调度的？"
→ "什么是工作窃取？怎么偷的？"
→ "系统调用时 P 和 M 会怎样？M 被阻塞了吗？"
→ "Go 的调度是抢占式的吗？1.14 怎么实现的？"
→ "netpoller 在调度中起什么作用？"
```

**追问链 3：GC**
```
"Go GC 用的什么算法？"
→ "什么是三色标记？为什么需要写屏障？"
→ "写屏障有几种？Go 用的是哪种？"
→ "STW 在哪些阶段？为什么能做到亚毫秒级？"
→ "GC 触发时机？GOGC 怎么调优？"
```

---

# 第十三章：Go 面试高频编码与算法题 🔥🔥🔥

> **目标**：覆盖 Go 面试中除原理问答外的高频手写代码题。包括并发数据结构实现、设计模式、以及 Go 实现经典算法题。每道题给出现场可写的标准答案。

---

## 13.1 并发数据结构实现

---

### 13.1.1 并发安全 LRU Cache ⭐🔥🔥

**题目**：实现一个并发安全的 LRU（最近最少使用）缓存，支持 Get 和 Put 操作，O(1) 时间复杂度。

```go
type LRUCache struct {
    cap     int
    mu      sync.RWMutex
    cache   map[int]*list.Element
    lruList *list.List
}

type entry struct {
    key, val int
}

func NewLRUCache(capacity int) *LRUCache {
    return &LRUCache{
        cap:     capacity,
        cache:   make(map[int]*list.Element),
        lruList: list.New(),
    }
}

func (c *LRUCache) Get(key int) int {
    c.mu.RLock()
    defer c.mu.RUnlock()

    if elem, ok := c.cache[key]; ok {
        c.lruList.MoveToFront(elem)
        return elem.Value.(*entry).val
    }
    return -1
}

func (c *LRUCache) Put(key, val int) {
    c.mu.Lock()
    defer c.mu.Unlock()

    if elem, ok := c.cache[key]; ok {
        elem.Value.(*entry).val = val
        c.lruList.MoveToFront(elem)
        return
    }

    if c.lruList.Len() >= c.cap {
        oldest := c.lruList.Back()
        c.lruList.Remove(oldest)
        delete(c.cache, oldest.Value.(*entry).key)
    }

    e := &entry{key, val}
    elem := c.lruList.PushFront(e)
    c.cache[key] = elem
}
```

**面试追问：**
- **Q：为什么读操作用 RLock 但还调用了 `MoveToFront`（写操作）？** 
  > 这是上面代码的一个**故意陷阱**。`MoveToFront` 修改了链表结构，属于写操作，在 RLock 下是不安全的（data race）。正确做法：Get 也需要 `Lock()`（互斥锁），或改用其他方案如在 entry 中加 `sync.Mutex`。面试官期望你**识别出这个并发问题**。
  
- **Q：正确方案是什么？**
  > 1) Get 和 Put 都用 `Lock()`（简单但读并发度差）；2) 分段锁减少竞争；3) 改用 `sync.Map` + 时间戳近似 LRU。

**✅ 正确的高并发版（读写均 Lock）：**
```go
func (c *LRUCache) Get(key int) int {
    c.mu.Lock()         // ← 注意：必须用 Lock，不是 RLock
    defer c.mu.Unlock()
    // ... 同上
}
```

---

### 13.1.2 有界阻塞队列 ⭐🔥🔥

**题目**：用 Go 实现一个有界阻塞队列，支持阻塞 Put 和阻塞 Take，队列满时 Put 阻塞，队列空时 Take 阻塞。

```go
type BoundedBlockingQueue struct {
    mu       sync.Mutex
    notFull  *sync.Cond
    notEmpty *sync.Cond
    data     []int
    cap      int
}

func NewBoundedBlockingQueue(cap int) *BoundedBlockingQueue {
    q := &BoundedBlockingQueue{
        data: make([]int, 0, cap),
        cap:  cap,
    }
    q.notFull = sync.NewCond(&q.mu)
    q.notEmpty = sync.NewCond(&q.mu)
    return q
}

func (q *BoundedBlockingQueue) Put(v int) {
    q.mu.Lock()
    defer q.mu.Unlock()

    for len(q.data) == q.cap { // ⚠️ 用 for 不是 if
        q.notFull.Wait()
    }
    q.data = append(q.data, v)
    q.notEmpty.Signal() // 唤醒一个等待的 Take
}

func (q *BoundedBlockingQueue) Take() int {
    q.mu.Lock()
    defer q.mu.Unlock()

    for len(q.data) == 0 { // ⚠️ 用 for 不是 if
        q.notEmpty.Wait()
    }
    v := q.data[0]
    q.data = q.data[1:]
    q.notFull.Signal() // 唤醒一个等待的 Put
    return v
}

func (q *BoundedBlockingQueue) Size() int {
    q.mu.Lock()
    defer q.mu.Unlock()
    return len(q.data)
}
```

**面试追问：**
- **Q：为什么 `Wait()` 要放在 `for` 循环里？**
  > 防止虚假唤醒（spurious wakeup）。即使没有 Signal，Wait 也可能被 runtime 唤醒。用 `for` 在唤醒后重新检查条件，条件不满足继续 Wait。
- **Q：`Signal()` vs `Broadcast()` 怎么选？**
  > 每次 Put/Take 只释放一个空位/元素，用 `Signal()` 唤醒一个等待者即可。`Broadcast()` 会唤醒所有等待者（惊群效应），大部分被唤醒后发现条件仍不满足又继续 Wait，浪费 CPU。
- **Q：如果改成用 channel 实现，区别在哪？**
  > channel 版本更简单（`make(chan int, cap)` + send/recv），但缺少 `Size()` 方法且不支持批量操作。面试核心是展示你理解 `sync.Cond` 的使用。

---

### 13.1.3 并发安全计数器 / 信号量 ⭐⭐

**信号量实现（channel 版本，最常用）：**
```go
type Semaphore struct {
    ch chan struct{}
}

func NewSemaphore(n int) *Semaphore {
    return &Semaphore{ch: make(chan struct{}, n)}
}

func (s *Semaphore) Acquire() { s.ch <- struct{}{} }
func (s *Semaphore) Release() { <-s.ch }

// 非阻塞获取
func (s *Semaphore) TryAcquire() bool {
    select {
    case s.ch <- struct{}{}:
        return true
    default:
        return false
    }
}
```

**并发安全计数器（atomic 版本，无锁）：**
```go
type Counter struct {
    val atomic.Int64
}

func (c *Counter) Inc() int64       { return c.val.Add(1) }
func (c *Counter) Dec() int64       { return c.val.Add(-1) }
func (c *Counter) Get() int64       { return c.val.Load() }
func (c *Counter) Reset()           { c.val.Store(0) }
func (c *Counter) Cas(old, new int64) bool { return c.val.CompareAndSwap(old, new) }
```

---

## 13.2 并发设计模式实战

---

### 13.2.1 并发安全单例模式 ⭐🔥

```go
// 方案一：sync.Once（推荐）
type Singleton struct{}

var (
    instance *Singleton
    once     sync.Once
)

func GetInstance() *Singleton {
    once.Do(func() {
        instance = &Singleton{}
    })
    return instance
}

// 方案二：init() 初始化（饿汉式）
var instance2 = &Singleton{} // 包加载时即初始化

func GetInstance2() *Singleton { return instance2 }

// 方案三：双重检查锁（不推荐，Go 有更简洁的方案）
var (
    mu       sync.Mutex
    instance3 *Singleton
)

func GetInstance3() *Singleton {
    if instance3 == nil { // 快速路径，避免每次都加锁
        mu.Lock()
        defer mu.Unlock()
        if instance3 == nil {
            instance3 = &Singleton{}
        }
    }
    return instance3
}
// 面试话术：Go 中用 sync.Once 即可，双重检查锁在 Go 中不常用且容易写错
```

**追问：sync.Once 和 init() 的区别？**
> `init()` 在包加载时执行（程序启动），不管是否会用到。`sync.Once` 在首次调用时执行（懒加载），更灵活且节省启动时间。

---

### 13.2.2 优雅关闭（多个 goroutine 按序退出）⭐🔥🔥

**题目**：启动 N 个 worker goroutine，要求收到信号后按顺序优雅关闭：先停止接收新任务 → 等待所有 worker 处理完 → 清理资源。

```go
func gracefulShutdown() {
    ctx, cancel := context.WithCancel(context.Background())

    const numWorkers = 5
    jobs := make(chan int, 100)
    var wg sync.WaitGroup

    // 启动 worker pool
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for {
                select {
                case job, ok := <-jobs:
                    if !ok { // channel 已关闭 → 退出
                        fmt.Printf("worker %d: channel closed, exiting\n", id)
                        return
                    }
                    process(job)
                case <-ctx.Done():
                    fmt.Printf("worker %d: ctx cancelled, exiting\n", id)
                    // 清空剩余 job（可选）
                    for job := range jobs {
                        process(job)
                    }
                    return
                }
            }
        }(i)
    }

    // 生产者
    go func() {
        for i := 0; i < 50; i++ {
            select {
            case jobs <- i:
            case <-ctx.Done():
                return
            }
        }
    }()

    // 等待 SIGTERM / SIGINT
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)
    <-sigCh

    fmt.Println("received shutdown signal")

    // 步骤1：通知所有 worker 准备退出
    cancel()

    // 步骤2：关闭 jobs channel（停止接收新任务）
    close(jobs)

    // 步骤3：等待所有 worker 处理完
    done := make(chan struct{})
    go func() {
        wg.Wait()
        close(done)
    }()

    // 步骤4：超时兜底
    select {
    case <-done:
        fmt.Println("all workers exited gracefully")
    case <-time.After(10 * time.Second):
        fmt.Println("shutdown timeout, force exit")
    }

    // 步骤5：清理资源
    fmt.Println("cleanup done")
}
```

**面试话术要点：**
> 1. `cancel()` + `close(channel)` 双保险：cancel 立即通知所有 select，close channel 让 `range` 循环退出
> 2. `wg.Wait()` 放在单独的 goroutine + 超时兜底：防止某个 worker 卡死导致整体阻塞
> 3. 关闭顺序：先 cancel → 再 close channel → 再 Wait → 超时兜底

---

### 13.2.3 令牌桶限流器 ⭐🔥

```go
type TokenBucket struct {
    rate       float64 // 每秒产生 token 数
    burst      int     // 桶容量（允许突发）
    tokens     float64
    lastUpdate time.Time
    mu         sync.Mutex
}

func NewTokenBucket(rate float64, burst int) *TokenBucket {
    return &TokenBucket{
        rate:       rate,
        burst:      burst,
        tokens:     float64(burst),
        lastUpdate: time.Now(),
    }
}

func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    now := time.Now()
    elapsed := now.Sub(tb.lastUpdate).Seconds()
    tb.tokens += elapsed * tb.rate // 补充 token
    if tb.tokens > float64(tb.burst) {
        tb.tokens = float64(tb.burst) // 不超过桶容量
    }
    tb.lastUpdate = now

    if tb.tokens >= 1 {
        tb.tokens--
        return true
    }
    return false
}
```

---

### 13.2.4 发布订阅模式（Pub-Sub）⭐

```go
type PubSub struct {
    mu     sync.RWMutex
    topics map[string][]chan string
}

func NewPubSub() *PubSub {
    return &PubSub{topics: make(map[string][]chan string)}
}

// Subscribe 返回一个只读 channel，订阅者从此 channel 接收消息
func (ps *PubSub) Subscribe(topic string) <-chan string {
    ps.mu.Lock()
    defer ps.mu.Unlock()

    ch := make(chan string, 10) // 带缓冲防止慢消费者阻塞生产者
    ps.topics[topic] = append(ps.topics[topic], ch)
    return ch
}

// Publish 向某个 topic 的所有订阅者发送消息
func (ps *PubSub) Publish(topic, msg string) {
    ps.mu.RLock()
    defer ps.mu.RUnlock()

    for _, ch := range ps.topics[topic] {
        select {
        case ch <- msg:
        default: // ⚠️ 消费者满，丢弃消息（可根据业务调整策略）
        }
    }
}

// Unsubscribe 取消订阅（简化版）
func (ps *PubSub) Unsubscribe(topic string, sub <-chan string) {
    ps.mu.Lock()
    defer ps.mu.Unlock()

    subs := ps.topics[topic]
    for i, ch := range subs {
        if ch == sub {
            ps.topics[topic] = append(subs[:i], subs[i+1:]...)
            close(ch)
            break
        }
    }
}
```

---

## 13.3 Go 实现经典数据结构题 ⭐🔥🔥

> 以下题目为通用高频题，但给出规范 Go 风格的标准答案。

---

### 13.3.1 用栈实现队列 / 用队列实现栈

**栈实现队列（两个栈，一个入队一个出队）：**
```go
type MyQueue struct {
    in  []int
    out []int
}

func NewMyQueue() *MyQueue { return &MyQueue{} }

func (q *MyQueue) Push(x int) {
    q.in = append(q.in, x)
}

func (q *MyQueue) Pop() int {
    q.moveIfEmpty()
    v := q.out[len(q.out)-1]
    q.out = q.out[:len(q.out)-1]
    return v
}

func (q *MyQueue) Peek() int {
    q.moveIfEmpty()
    return q.out[len(q.out)-1]
}

func (q *MyQueue) Empty() bool {
    return len(q.in) == 0 && len(q.out) == 0
}

func (q *MyQueue) moveIfEmpty() {
    if len(q.out) == 0 {
        for len(q.in) > 0 {
            top := q.in[len(q.in)-1]
            q.in = q.in[:len(q.in)-1]
            q.out = append(q.out, top)
        }
    }
}
// 摊还 O(1)
```

**队列实现栈（两个队列/一个队列）：**
```go
type MyStack struct {
    q []int
}

func NewMyStack() *MyStack { return &MyStack{} }

func (s *MyStack) Push(x int) {
    s.q = append(s.q, x)
    // 将前面 n-1 个元素移到末尾
    for i := 0; i < len(s.q)-1; i++ {
        s.q = append(s.q, s.q[0])
        s.q = s.q[1:]
    }
}

func (s *MyStack) Pop() int {
    v := s.q[0]
    s.q = s.q[1:]
    return v
}

func (s *MyStack) Top() int    { return s.q[0] }
func (s *MyStack) Empty() bool { return len(s.q) == 0 }
```

---

### 13.3.2 最小栈（O(1) 获取最小值）⭐🔥

```go
type MinStack struct {
    data    []int
    minData []int // 辅助栈存储每个状态的最小值
}

func NewMinStack() *MinStack { return &MinStack{} }

func (s *MinStack) Push(x int) {
    s.data = append(s.data, x)
    // minData 栈顶总是当前 data 的最小值
    if len(s.minData) == 0 || x <= s.GetMin() {
        s.minData = append(s.minData, x)
    } else {
        s.minData = append(s.minData, s.GetMin())
    }
}

func (s *MinStack) Pop() {
    s.data = s.data[:len(s.data)-1]
    s.minData = s.minData[:len(s.minData)-1]
}

func (s *MinStack) Top() int     { return s.data[len(s.data)-1] }
func (s *MinStack) GetMin() int  { return s.minData[len(s.minData)-1] }
```

---

### 13.3.3 链表高频题（Go 写法）⭐⭐

```go
type ListNode struct {
    Val  int
    Next *ListNode
}

// 反转链表（迭代）
func reverseList(head *ListNode) *ListNode {
    var prev *ListNode
    curr := head
    for curr != nil {
        next := curr.Next
        curr.Next = prev
        prev = curr
        curr = next
    }
    return prev
}

// 反转链表（递归）
func reverseListRecursive(head *ListNode) *ListNode {
    if head == nil || head.Next == nil {
        return head
    }
    newHead := reverseListRecursive(head.Next)
    head.Next.Next = head
    head.Next = nil
    return newHead
}

// 合并两个有序链表
func mergeTwoLists(l1, l2 *ListNode) *ListNode {
    dummy := &ListNode{}
    curr := dummy
    for l1 != nil && l2 != nil {
        if l1.Val < l2.Val {
            curr.Next = l1
            l1 = l1.Next
        } else {
            curr.Next = l2
            l2 = l2.Next
        }
        curr = curr.Next
    }
    if l1 != nil {
        curr.Next = l1
    }
    if l2 != nil {
        curr.Next = l2
    }
    return dummy.Next
}

// 判断环形链表（快慢指针）
func hasCycle(head *ListNode) bool {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            return true
        }
    }
    return false
}

// 找到环形链表的入口
func detectCycle(head *ListNode) *ListNode {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            slow = head
            for slow != fast {
                slow = slow.Next
                fast = fast.Next
            }
            return slow
        }
    }
    return nil
}
```

---

### 13.3.4 二叉树高频题（Go 写法）⭐⭐

```go
type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// 前序遍历（迭代）
func preorderTraversal(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    var res []int
    stack := []*TreeNode{root}
    for len(stack) > 0 {
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        res = append(res, node.Val)
        if node.Right != nil {
            stack = append(stack, node.Right)
        }
        if node.Left != nil {
            stack = append(stack, node.Left)
        }
    }
    return res
}

// 中序遍历（迭代）
func inorderTraversal(root *TreeNode) []int {
    var res []int
    var stack []*TreeNode
    curr := root
    for curr != nil || len(stack) > 0 {
        for curr != nil {
            stack = append(stack, curr)
            curr = curr.Left
        }
        curr = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        res = append(res, curr.Val)
        curr = curr.Right
    }
    return res
}

// 层序遍历（BFS）
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }
    var res [][]int
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        levelSize := len(queue)
        var level []int
        for i := 0; i < levelSize; i++ {
            node := queue[0]
            queue = queue[1:]
            level = append(level, node.Val)
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        res = append(res, level)
    }
    return res
}

// 二叉树最大深度
func maxDepth(root *TreeNode) int {
    if root == nil {
        return 0
    }
    left := maxDepth(root.Left)
    right := maxDepth(root.Right)
    return max(left, right) + 1
}

// 对称二叉树
func isSymmetric(root *TreeNode) bool {
    if root == nil {
        return true
    }
    var check func(p, q *TreeNode) bool
    check = func(p, q *TreeNode) bool {
        if p == nil && q == nil {
            return true
        }
        if p == nil || q == nil {
            return false
        }
        return p.Val == q.Val && check(p.Left, q.Right) && check(p.Right, q.Left)
    }
    return check(root.Left, root.Right)
}
```

---

## 13.4 高频算法题 Go 实现 ⭐🔥🔥

---

### 13.4.1 排序算法（手写快排 + 归并）⭐🔥

**快速排序：**
```go
func quickSort(nums []int) {
    if len(nums) <= 1 {
        return
    }
    pivot := partition(nums)
    quickSort(nums[:pivot])
    quickSort(nums[pivot+1:])
}

func partition(nums []int) int {
    pivot := nums[len(nums)-1]
    i := -1 // i 指向最后一个 ≤ pivot 的元素
    for j := 0; j < len(nums)-1; j++ {
        if nums[j] <= pivot {
            i++
            nums[i], nums[j] = nums[j], nums[i]
        }
    }
    nums[i+1], nums[len(nums)-1] = nums[len(nums)-1], nums[i+1]
    return i + 1
}
```

**归并排序：**
```go
func mergeSort(nums []int) []int {
    if len(nums) <= 1 {
        return nums
    }
    mid := len(nums) / 2
    left := mergeSort(nums[:mid])
    right := mergeSort(nums[mid:])
    return merge(left, right)
}

func merge(left, right []int) []int {
    res := make([]int, 0, len(left)+len(right))
    i, j := 0, 0
    for i < len(left) && j < len(right) {
        if left[i] <= right[j] {
            res = append(res, left[i])
            i++
        } else {
            res = append(res, right[j])
            j++
        }
    }
    res = append(res, left[i:]...)
    res = append(res, right[j:]...)
    return res
}
```

---

### 13.4.2 二分查找及其变体 ⭐⭐

```go
// 标准二分查找
func binarySearch(nums []int, target int) int {
    lo, hi := 0, len(nums)-1
    for lo <= hi {
        mid := lo + (hi-lo)/2 // 防溢出
        if nums[mid] == target {
            return mid
        } else if nums[mid] < target {
            lo = mid + 1
        } else {
            hi = mid - 1
        }
    }
    return -1
}

// 查找第一个等于 target 的索引（左边界）
func leftBound(nums []int, target int) int {
    lo, hi := 0, len(nums)
    for lo < hi {
        mid := lo + (hi-lo)/2
        if nums[mid] >= target {
            hi = mid
        } else {
            lo = mid + 1
        }
    }
    if lo < len(nums) && nums[lo] == target {
        return lo
    }
    return -1
}

// 查找最后一个等于 target 的索引（右边界）
func rightBound(nums []int, target int) int {
    lo, hi := 0, len(nums)
    for lo < hi {
        mid := lo + (hi-lo)/2
        if nums[mid] <= target {
            lo = mid + 1
        } else {
            hi = mid
        }
    }
    if lo-1 >= 0 && nums[lo-1] == target {
        return lo - 1
    }
    return -1
}
```

---

### 13.4.3 滑动窗口模板 ⭐🔥

```go
// 最长无重复字符子串 (LeetCode 3)
func lengthOfLongestSubstring(s string) int {
    window := make(map[byte]int) // char → count
    left, maxLen := 0, 0
    for right := 0; right < len(s); right++ {
        window[s[right]]++
        // 收缩窗口条件：有重复字符
        for window[s[right]] > 1 {
            window[s[left]]--
            left++
        }
        maxLen = max(maxLen, right-left+1)
    }
    return maxLen
}

// 通用滑动窗口模板
func slidingWindowTemplate(s string) {
    need := make(map[byte]int)   // 需要的字符及其数量
    window := make(map[byte]int) // 窗口内的字符及其数量
    left, right := 0, 0
    valid := 0 // 已满足条件的字符种类数

    for right < len(s) {
        // 1. 扩大窗口：加入 right 指向的字符
        c := s[right]
        right++

        // 2. 更新窗口数据
        // ...

        // 3. 判断是否需要收缩窗口
        for windowNeedsShrink() {
            // 4. 收缩窗口：移除 left 指向的字符
            d := s[left]
            left++
            // 更新窗口数据
            _ = d
        }
    }
    _ = need
    _ = window
    _ = valid
}
```

---

### 13.4.4 哈希表高频题 ⭐⭐

```go
// 两数之和
func twoSum(nums []int, target int) []int {
    m := make(map[int]int) // val → index
    for i, v := range nums {
        if j, ok := m[target-v]; ok {
            return []int{j, i}
        }
        m[v] = i
    }
    return nil
}

// 三数之和（排序 + 双指针）
func threeSum(nums []int) [][]int {
    sort.Ints(nums)
    var res [][]int
    for i := 0; i < len(nums)-2; i++ {
        if i > 0 && nums[i] == nums[i-1] { // 跳过重复
            continue
        }
        lo, hi := i+1, len(nums)-1
        for lo < hi {
            sum := nums[i] + nums[lo] + nums[hi]
            if sum == 0 {
                res = append(res, []int{nums[i], nums[lo], nums[hi]})
                for lo < hi && nums[lo] == nums[lo+1] { lo++ }
                for lo < hi && nums[hi] == nums[hi-1] { hi-- }
                lo++
                hi--
            } else if sum < 0 {
                lo++
            } else {
                hi--
            }
        }
    }
    return res
}

// LRU Cache（非并发安全版，更侧重算法）
// 见 13.1.1 并发安全版，去掉锁即可
```

---

### 13.4.5 动态规划高频题 ⭐🔥

```go
// 最长递增子序列 (LIS)
func lengthOfLIS(nums []int) int {
    // dp[i] = 以 nums[i] 结尾的最长递增子序列长度
    dp := make([]int, len(nums))
    maxLen := 0
    for i := 0; i < len(nums); i++ {
        dp[i] = 1
        for j := 0; j < i; j++ {
            if nums[j] < nums[i] {
                dp[i] = max(dp[i], dp[j]+1)
            }
        }
        maxLen = max(maxLen, dp[i])
    }
    return maxLen
}

// 最长公共子序列 (LCS)
func longestCommonSubsequence(text1, text2 string) int {
    m, n := len(text1), len(text2)
    // dp[i][j] = text1[:i] 和 text2[:j] 的 LCS 长度
    dp := make([][]int, m+1)
    for i := range dp {
        dp[i] = make([]int, n+1)
    }
    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if text1[i-1] == text2[j-1] {
                dp[i][j] = dp[i-1][j-1] + 1
            } else {
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
            }
        }
    }
    return dp[m][n]
}

// 0-1 背包
func knapsack(weights, values []int, capacity int) int {
    n := len(weights)
    dp := make([]int, capacity+1)
    for i := 0; i < n; i++ {
        for w := capacity; w >= weights[i]; w-- { // 倒序！防重复使用
            dp[w] = max(dp[w], dp[w-weights[i]]+values[i])
        }
    }
    return dp[capacity]
}
```

---

### 13.4.6 字符串高频题 ⭐⭐

```go
// 字符串反转（注意中文）
func reverseString(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}

// 判断回文
func isPalindrome(s string) bool {
    // 预处理：只保留字母数字，转小写
    var cleaned []rune
    for _, r := range s {
        if unicode.IsLetter(r) || unicode.IsDigit(r) {
            cleaned = append(cleaned, unicode.ToLower(r))
        }
    }
    for i, j := 0, len(cleaned)-1; i < j; i, j = i+1, j-1 {
        if cleaned[i] != cleaned[j] {
            return false
        }
    }
    return true
}
```

---

## 13.5 题型分布与复习策略

| 类别 | 题目数 | 面试命中率 | 优先复习 |
|------|--------|----------|---------|
| 并发数据结构（LRU/阻塞队列/信号量） | 3 | 🔥🔥🔥 极高 | ✅ 第一优先级 |
| 并发设计模式（单例/优雅关闭/限流器/PubSub） | 4 | 🔥🔥🔥 极高 | ✅ 第一优先级 |
| 经典数据结构（栈队列/链表/二叉树） | ~12 | 🔥🔥 高 | ✅ 第二优先级 |
| 排序算法（快排/归并） | 2 | 🔥🔥 高 | ✅ 第二优先级 |
| 二分查找 | 3 | 🔥🔥 高 | ✅ 第二优先级 |
| 滑动窗口 | 1 | 🔥🔥 高 | ✅ 第二优先级 |
| 哈希表题（两数和/三数和） | 2 | 🔥🔥 高 | 第二优先级 |
| 动态规划（LIS/LCS/背包） | 3 | 🔥 中 | 第三优先级 |
| 字符串（反转/回文） | 2 | 🔥 中 | 第三优先级 |

> **核心原则**：Go 面试中的算法轮次，考察重点是「Go 并发编码能力」>「数据结构实现」>「纯算法」。优先掌握 13.1（并发数据结构）和 13.2（并发设计模式），它们是 Go 面试区分度的关键。

---

# 附录一：完整面试题速查表

| # | 问题 | 等级 | 核心答案关键词 |
|------|------|------|------|
| 1 | interface 底层结构？nil 接口陷阱？ | ⭐🔥🔥 | eface/iface, 类型+值都为 nil |
| 2 | GMP 调度模型？P 为什么需要？ | ⭐🔥🔥 | G=任务, M=线程, P=局部性+减少锁竞争 |
| 3 | 三色标记 + 写屏障？亚毫秒 GC？ | ⭐🔥🔥 | 并发标记, 混合写屏障, 两次微秒级 STW |
| 4 | channel 底层？发送接收全流程？ | ⭐🔥🔥 | hchan=环形队列+等待队列+mutex |
| 5 | slice 扩容机制？ | ⭐🔥 | <256 翻倍, ≥256 阶梯 ~1.25x |
| 6 | map 并发安全？sync.Map？ | ⭐🔥 | 非并发安全, read+dirty 读写分离 |
| 7 | defer 执行顺序与参数求值？ | ⭐🔥 | LIFO, 参数立即求值, 闭包读最终值 |
| 8 | Context 取消传播？ | ⭐🔥 | 树状递归, close channel 广播 |
| 9 | 逃逸分析是什么？常见场景？ | ⭐🔥 | 编译期栈/堆决策, 返回指针/装箱/闭包 |
| 10 | goroutine 泄漏排查？ | ⭐🔥 | pprof goroutine, channel 永久阻塞 |
| 11 | Mutex 正常/饥饿模式？ | ⭐ | 自旋 vs 信号量 FIFO, >1ms 进入饥饿 |
| 12 | sync.Once 实现原理？ | ⭐ | 双重检查 + defer 标记 done |
| 13 | 反射三定律？为什么慢？ | ⭐ | 类型检查+堆分配+间接调用+无法内联 |
| 14 | MVS 算法？与 npm 区别？ | ⭐ | 最小满足版本, 确定性构建 |
| 15 | 高并发 HTTP 设计要点？ | ⭐ | MaxIdleConnsPerHost, 超时, 优雅关闭 |

---

> **使用建议：**
> - 面试前 2h：通读各章"总结（速记版）"
> - 面试前 30min：看附录速查表 + 第十二章连环追问链 + 覆盖率自检表
> - 算法轮次前 1h：通读第十三章（并发编码+数据结构+算法题）
> - 面试临场：回忆"五、面试高频问题"中的标准答案结构
> - 日常积累：对照源码阅读（runtime/chan.go, runtime/map.go, runtime/proc.go, runtime/mgc.go）

---

# 附录二：面试覆盖率自检表

> 使用说明：逐项核对，确保每个知识点都到达 ✅ 状态。标注 ❌ 的需重点补充复习。

## 一、并发模型（必考区）

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| goroutine 本质（轻量线程、动态栈） | ✅ | ✅ | ✅ | ✅ |
| GMP 调度模型（G/M/P 角色与协作） | ✅ | ✅ | ✅ | ✅ |
| 工作窃取（Work Stealing）机制 | ✅ | ✅ | ✅ | — |
| 系统调用 Handoff 机制 | ✅ | ✅ | ✅ | — |
| 信号抢占式调度（1.14+） | ✅ | ✅ | ✅ | ✅ |
| netpoller 与调度器协作 | ✅ | ✅ | ✅ | — |
| g0 与用户栈切换 | ✅ | ✅ | ✅ | — |
| GODEBUG schedtrace 排查 | ✅ | ✅ | ✅ | ✅ |

## 二、Channel 与 Select（必考区）

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| hchan 底层结构（环形队列+等待队列） | ✅ | ✅ | ✅ | — |
| 发送/接收完整流程 | ✅ | ✅ | ✅ | — |
| sendq 优先机制（缓冲满时接收者优先取 buf） | ✅ | ✅ | ✅ | — |
| select 随机化 + lockorder 机制 | ✅ | ✅ | ✅ | — |
| channel 关闭后行为 | ✅ | ✅ | ✅ | — |
| nil channel 行为 | ✅ | ✅ | ✅ | — |
| time.After 循环中的 Timer 泄漏 | ✅ | ✅ | ✅ | ✅ |
| channel 阻塞排查 | ✅ | ✅ | ✅ | ✅ |

## 三、Sync 包（必考区）

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| Mutex 正常模式 vs 饥饿模式 | ✅ | ✅ | ✅ | — |
| Mutex 自旋条件 | ✅ | ✅ | ✅ | — |
| RWMutex 写优先设计 | ✅ | ✅ | ✅ | — |
| Once 双重检查 + defer 标记 done | ✅ | ✅ | ✅ | — |
| WaitGroup 底层实现 | ✅ | ✅ | ✅ | ✅ |
| sync.Pool 原理 + victim 机制 | ✅ | ✅ | ✅ | — |
| sync.Map read+dirty 读写分离 | ✅ | ✅ | ✅ | ✅ |
| sync.Cond 条件变量 | ✅ | ✅ | ✅ | — |
| atomic 原子操作（CPU LOCK 指令） | ✅ | ✅ | ✅ | — |

## 四、Context（必考区）

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| Context 接口设计 | ✅ | ✅ | ✅ | — |
| 取消传播机制（树状递归） | ✅ | ✅ | ✅ | — |
| WithCancel/WithTimeout/WithDeadline | ✅ | ✅ | ✅ | — |
| Context.Value 使用规范 + key 类型选择 | ✅ | ✅ | ✅ | — |
| context.Value 的查找链路（向上递归） | ✅ | ✅ | ✅ | — |
| cancel 函数必须调用的原因 | ✅ | ✅ | ✅ | ✅ |

## 五、内存与 GC（必考区）

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| 内存分配器五级架构（mcache/mcentral/mheap/pageAlloc/OS） | ✅ | ✅ | ✅ | — |
| 对象分级（Tiny/Small/Large + 67种size class） | ✅ | ✅ | ✅ | — |
| mspan 结构 + allocBits 位图 vs freelist 演进 | ✅ | ✅ | ✅ | — |
| mallocgc 完整分配路径（三条路径） | ✅ | ✅ | ✅ | — |
| Tiny 分配器合并分配机制 | ✅ | ✅ | ✅ | — |
| page allocator（radix tree 页管理） | ✅ | ✅ | ✅ | — |
| 栈 vs 堆分配 | ✅ | ✅ | ✅ | — |
| 逃逸分析（编译器保守策略） | ✅ | ✅ | ✅ | — |
| 三色标记算法 | ✅ | ✅ | ✅ | — |
| 写屏障（插入/删除/混合） | ✅ | ✅ | ✅ | — |
| 混合写屏障原理（Go 1.8+） | ✅ | ✅ | ✅ | — |
| 两次 STW（Mark Setup + Mark Termination） | ✅ | ✅ | ✅ | — |
| GC 触发条件（GOGC/2min/runtime.GC） | ✅ | ✅ | ✅ | — |
| Mark Assist 背压机制 | ✅ | ✅ | ✅ | — |
| GC 根对象定义（栈/全局变量/寄存器） | ✅ | ✅ | ✅ | — |
| GOGC 调优 + GOMEMLIMIT（1.19+） | ✅ | ✅ | ✅ | ✅ |
| Go 不用分代 GC 的原因 | ✅ | ✅ | ✅ | — |
| GC 核心指标（STW/CPU占比/频率/分配速率） | ✅ | ✅ | ✅ | — |
| 常见 GC 算法对比（引用计数/标记清除/标记整理/复制/分代） | ✅ | ✅ | ✅ | — |

## 六、数据结构（必考区）

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| slice 底层结构（24B header） | ✅ | ✅ | ✅ | — |
| slice 扩容机制（Go 1.18+ 阶梯公式 + 源码级 growslice） | ✅ | ✅ | ✅ | — |
| roundupsize 内存对齐扩容细节 | ✅ | ✅ | ✅ | — |
| 截取共享底层数组陷阱 | ✅ | ✅ | ✅ | ✅ |
| nil slice vs empty slice | ✅ | ✅ | ✅ | — |
| map 底层 hmap + bmap | ✅ | ✅ | ✅ | — |
| map 扩容（翻倍 vs 等量） | ✅ | ✅ | ✅ | — |
| 一次 evacuate 逐步骤搬迁流程 | ✅ | ✅ | ✅ | — |
| 翻倍扩容 bucket 分流映射关系 | ✅ | ✅ | ✅ | — |
| map 渐进式搬迁 + 扩容期间读写行为 | ✅ | ✅ | ✅ | — |
| map 迭代器结构 + 扩容时遍历行为 | ✅ | ✅ | ✅ | — |
| map 并发安全方案对比 | ✅ | ✅ | ✅ | ✅ |
| map 不可取地址 + 不可做 key 类型 | ✅ | ✅ | ✅ | — |
| map 删除后内存释放问题 | ✅ | ✅ | ✅ | — |
| string 底层结构 + 不可变性 | ✅ | ✅ | ✅ | — |
| string ↔ []byte 互转拷贝 | ✅ | ✅ | ✅ | — |

## 七、语言语义（必考区）

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| defer 执行顺序 LIFO | ✅ | ✅ | ✅ | — |
| defer 参数立即求值 | ✅ | ✅ | ✅ | — |
| defer 命名返回值可修改 | ✅ | ✅ | ✅ | — |
| open-coded defer（1.14+） | ✅ | ✅ | ✅ | — |
| panic/recover 机制 | ✅ | ✅ | ✅ | — |
| recover 只在直接 defer 中有效 | ✅ | ✅ | ✅ | ✅ |
| interface 底层（iface/eface） | ✅ | ✅ | ✅ | — |
| nil 接口陷阱 | ✅ | ✅ | ✅ | ✅ |
| itab 方法派发流程 | ✅ | ✅ | ✅ | — |
| 装箱代价与逃逸 | ✅ | ✅ | ✅ | — |
| 值传递 vs 引用语义 | ✅ | ✅ | ✅ | — |
| 方法集（T vs *T） | ✅ | ✅ | ✅ | — |
| for range 循环变量复用（1.22 前后） | ✅ | ✅ | ✅ | ✅ |
| 闭包变量捕获陷阱 | ✅ | ✅ | ✅ | ✅ |
| 多返回值底层（调用方栈帧） | ✅ | ✅ | ✅ | — |

## 八、Go 内存模型（Happens-Before）

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| Happens-Before 定义与意义 | ✅ | ✅ | ✅ | — |
| channel 的 happens-before 保证 | ✅ | ✅ | ✅ | ✅ |
| Mutex 的 happens-before 保证 | ✅ | ✅ | ✅ | — |
| atomic 的可见性边界 | ✅ | ✅ | ✅ | ✅ |
| 无同步并发访问的风险 | ✅ | ✅ | ✅ | ✅ |

## 九、网络与 IO

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| Go IO 模型（同步 API + 异步底层） | ✅ | ✅ | ✅ | — |
| netpoller 与 epoll 协作 | ✅ | ✅ | ✅ | — |
| net/http 服务端流程 | ✅ | ✅ | ✅ | — |
| http.Client 连接池（MaxIdleConnsPerHost） | ✅ | ✅ | ✅ | — |
| HTTP 中间件模式（Handler 嵌套） | ✅ | ✅ | ✅ | — |
| http.Hijack（WebSocket 底座） | ✅ | ✅ | ✅ | — |
| http.Server 超时参数（防 Slowloris） | ✅ | ✅ | ✅ | — |
| 优雅关闭（Shutdown） | ✅ | ✅ | ✅ | — |
| goroutine-per-connection 模式 | ✅ | ✅ | ✅ | — |

## 十、并发场景排查

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| 死锁排查（channel 互相等待 + Mutex 交叉锁） | ✅ | ✅ | ✅ | ✅ |
| goroutine 泄漏排查（pprof + 根因分析） | ✅ | ✅ | ✅ | ✅ |
| channel 阻塞问题（生产者消费者速度不匹配） | ✅ | ✅ | ✅ | ✅ |
| map 并发 panic（fatal error vs panic 区别） | ✅ | ✅ | ✅ | ✅ |
| WaitGroup 使用陷阱（Add 位置 + 值拷贝） | ✅ | ✅ | ✅ | ✅ |
| 发送到已关闭 channel 的防护 | ✅ | ✅ | ✅ | ✅ |
| Race Detector 原理与局限 | ✅ | ✅ | ✅ | — |
| pprof 实战（CPU/内存/goroutine/锁竞争） | ✅ | ✅ | ✅ | ✅ |

## 十一、工程实践

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| Go Modules + MVS 算法 | ✅ | ✅ | ✅ | — |
| go.work 工作区（多模块本地开发） | ✅ | ✅ | ✅ | — |
| 错误处理（%w / errors.Is / errors.As） | ✅ | ✅ | ✅ | — |
| Table-Driven Tests | ✅ | ✅ | ✅ | — |
| Benchmark 正确写法 | ✅ | ✅ | ✅ | — |
| Struct Tag 原理 | ✅ | ✅ | ✅ | — |
| init() 执行顺序 | ✅ | ✅ | ✅ | — |
| Functional Options 模式 | ✅ | ✅ | ✅ | — |
| 反射三定律 + 方法调用 + Type/Value 区别 | ✅ | ✅ | ✅ | — |
| 泛型实现（GC Shape） | ✅ | ✅ | ✅ | — |
| Go 1.21+ slices/maps/slog/cmp/clear | ✅ | ✅ | ✅ | — |
| CGO 开销与使用原则 | ✅ | ✅ | ✅ | — |
| unsafe.Pointer vs uintptr | ✅ | ✅ | ✅ | — |
| go generate 代码生成 | ✅ | ✅ | ✅ | — |
| go vet / go fmt / goimports | ✅ | ✅ | ✅ | — |
| 交叉编译 | ✅ | ✅ | ✅ | — |
| Go 汇编基础（Plan 9） | ✅ | ✅ | ✅ | — |
| Go 编译速度为什么快 | ✅ | ✅ | ✅ | — |

## 十二、线上排查实战

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| CPU 100% 排查（pprof + 热点定位 + 修复） | ✅ | ✅ | ✅ | ✅ |
| 内存泄漏排查（heap profile + goroutine泄漏 + 4大根因） | ✅ | ✅ | ✅ | ✅ |
| 接口延迟突增排查（mutex/block + schedtrace + 4大原因） | ✅ | ✅ | ✅ | ✅ |
| 排查工具全景图（症状→工具映射） | ✅ | — | ✅ | — |

## 十三、并发编程模式

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| Pipeline 管道模式 | ✅ | — | — | — |
| Fan-Out / Fan-In | ✅ | — | — | — |
| Or-Done 模式 | ✅ | — | — | — |
| Worker Pool | ✅ | — | — | — |

## 十四、交替打印编码题（高频手写题）

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| 两个 goroutine 交替打印（channel） | ✅ | ✅ | ✅ | ✅ |
| 两个 goroutine 交替打印（Mutex+Cond） | ✅ | ✅ | ✅ | ✅ |
| N 个 goroutine 按序打印（channel 接力链） | ✅ | ✅ | ✅ | ✅ |
| 交替打印 + Context 超时退出 | ✅ | ✅ | ✅ | ✅ |
| 方案对比 + 面试话术 | ✅ | — | ✅ | — |

## 十五、并发数据结构实现（必考）

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| 并发安全 LRU Cache | ✅ | ✅ | ✅ | ✅ |
| 有界阻塞队列（sync.Cond） | ✅ | ✅ | ✅ | ✅ |
| 信号量（channel 实现） | ✅ | ✅ | ✅ | ✅ |
| 并发安全计数器（atomic） | ✅ | ✅ | ✅ | — |

## 十六、并发设计模式

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| 并发安全单例（sync.Once / 双重检查） | ✅ | ✅ | ✅ | ✅ |
| 优雅关闭（多 goroutine 按序退出） | ✅ | ✅ | ✅ | ✅ |
| 令牌桶限流器 | ✅ | ✅ | ✅ | — |
| 发布订阅模式（Pub-Sub） | ✅ | ✅ | ✅ | — |

## 十七、经典数据结构题（Go 实现）

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| 栈实现队列 / 队列实现栈 | ✅ | — | — | — |
| 最小栈（O(1) GetMin） | ✅ | — | — | — |
| 链表反转（迭代+递归） | ✅ | — | — | — |
| 合并有序链表 | ✅ | — | — | — |
| 环形链表检测（快慢指针） | ✅ | — | — | — |
| 二叉树遍历（前/中/层序） | ✅ | — | — | — |
| 二叉树深度 / 对称二叉树 | ✅ | — | — | — |

## 十八、高频算法题（Go 实现）

| 知识点 | 是否覆盖 | 是否深入底层 | 是否含面试问答 | 是否有场景题 |
|--------|--------|------------|--------------|------------|
| 快速排序 + 归并排序 | ✅ | — | — | — |
| 二分查找 + 左右边界变体 | ✅ | — | — | — |
| 滑动窗口模板 + 例题 | ✅ | — | — | — |
| 两数之和 / 三数之和 | ✅ | — | — | — |
| 最长递增子序列（LIS） | ✅ | — | — | — |
| 最长公共子序列（LCS） | ✅ | — | — | — |
| 0-1 背包 | ✅ | — | — | — |
| 字符串反转 / 回文 | ✅ | — | — | — |

---

## 覆盖率统计

| 类别 | 知识点总数 | 覆盖数 | 覆盖率 |
|------|---------|--------|--------|
| 并发模型 | 8 | 8 | 100% |
| Channel 与 Select | 8 | 8 | 100% |
| Sync 包 | 9 | 9 | 100% |
| Context | 6 | 6 | 100% |
| 内存与 GC | 19 | 19 | 100% |
| 数据结构 | 16 | 16 | 100% |
| 语言语义 | 15 | 15 | 100% |
| Go 内存模型 | 5 | 5 | 100% |
| 网络与 IO | 9 | 9 | 100% |
| 并发场景排查 | 8 | 8 | 100% |
| 线上排查实战 | 4 | 4 | 100% |
| 交替打印编码题 | 5 | 5 | 100% |
| 并发数据结构实现 | 4 | 4 | 100% |
| 并发设计模式 | 4 | 4 | 100% |
| 经典数据结构题（Go实现） | 7 | 7 | 100% |
| 高频算法题（Go实现） | 8 | 8 | 100% |
| 工程实践 | 18 | 18 | 100% |
| 并发编程模式 | 4 | 4 | 100% |
| **总计** | **157** | **157** | **100%** ✅ |

---

> **自检完成时间**：每次面试前逐项检查，确保 ✅ 全满。
