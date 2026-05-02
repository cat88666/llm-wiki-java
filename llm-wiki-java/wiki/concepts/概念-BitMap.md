---
type: concept
status: active
name: "BitMap"
layer: L4
aliases: ["位图", "Bitmap", "BitSet", "布隆过滤器基础"]
related:
  - "[[概念-前缀树]]"
  - "[[概念-线性数据结构]]"
sources:
  - "../../raw/note/📚 Hollis Java/数据结构/✅什么是BitMap？有什么用？ 30f3673e113881f38f2ce34a527f4a44.md"
created: 2026-05-02
updated: 2026-05-02
lint_notes: ""
---

# BitMap

> 用 1 bit 标记一个整数是否存在的紧凑数组——核心优势是极致的空间压缩（32 位整数用 1 bit 代替 32 bit），代价是只能表示"存在/不存在"的 boolean 语义。

## 第一性原理

存储 N 个整数最朴素的方法是数组，每个 int 占 4 字节（32 bit）。但如果只需要知道某个整数"有没有"，实际只需 1 bit（0/1）。BitMap 的存在理由：**当数据语义可以退化为 boolean（存在/不存在），bit 数组将空间压缩 32 倍**。

## 核心机制

### 结构

BitMap 本质是一个 bit 数组：
- 下标 = 整数值
- 值 = 0（不存在）或 1（存在）

存储 {1, 4, 6}：

```
index: 0 1 2 3 4 5 6 7 ...
value: 0 1 0 0 1 0 1 0 ...
```

### 空间对比

存储 3 个 unsigned int（1, 4, 6）：
- 朴素数组：3 × 4 字节 = 12 字节 = 96 bit
- BitMap：只需 7 bit（覆盖到最大值 6）

扩展：存储 10 亿个 int 的去重集合：
- HashSet：10亿 × ~40 字节 ≈ 40 GB
- BitMap：10亿 bit = 125 MB

### Java BitSet

`java.util.BitSet` 是 BitMap 的标准实现：

```java
BitSet bs = new BitSet(100);
bs.set(1);   // 标记 1 存在
bs.set(4);
bs.get(4);   // true
bs.clear(4); // 移除
```

### 布隆过滤器（Bloom Filter）

布隆过滤器是 BitMap 的扩展：
- 用 k 个哈希函数将元素映射到 k 个 bit 位
- 查询时 k 个位全为 1 → 可能存在（有误判率）
- 查询时有一个位为 0 → 一定不存在（无漏判）

## 关键权衡

1. **只能存 boolean**：无法存具体值，只能回答"是否存在"
2. **整数范围决定空间**：值域越大，BitMap 越大（与存储元素数量无关）
3. **布隆过滤器有误判率**：可以接受一定误判换取极小空间；无法删除元素（除非 Counting Bloom Filter）
4. **不支持排序/遍历具体值**：按位遍历效率低

## 与其他概念的关系

- 与 [[概念-前缀树]] 类比：都是通过结构化编码大幅压缩存储；Trie 针对字符串，BitMap 针对整数集合
- 支撑了 L5 Redis：Redis 的 `SETBIT`/`GETBIT` 命令即位图，用于用户签到、UV 计数
- 支撑了 L7 分布式：布隆过滤器是缓存穿透防护的核心手段

## 应用边界

**适合**：
- 海量整数去重（10 亿整数是否存在某值，125 MB 即可）
- 用户签到/在线状态（userId → bit 位）
- 布隆过滤器前置过滤（缓存穿透、垃圾邮件过滤、爬虫 URL 去重）
- 大数据排序（整数值域有限时，遍历 BitMap 即完成排序）

**不适合**：
- 值域过大的整数（如 64 位随机 ID，需 2^64 bit = 2 EB）
- 需要存储具体值或关联数据
- 需要删除操作（标准 BitMap 无法删除）
