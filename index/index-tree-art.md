---
title: ART 索引树
date: 2020-06-25 11:29:09
tags: 索引
---

总结动态基数树 [ART](https://db.in.tum.de/~leis/papers/ART.pdf) 的概念，插入、删除和搜索算法。

<!-- more -->

## 1.  概念

ART 由字典树 Trie Tree 和基数树 Radix Tree 衍生而来，都同属于前缀树。

- [字典树](https://zh.wikipedia.org/wiki/Trie)：Key 按字节逐个拆成有层序的子节点，并标记是否为数据节点，全量展开导致长 KEY 耗内存。
- [基数树](https://zh.wikipedia.org/wiki/%E5%9F%BA%E6%95%B0%E6%A0%91)：空间优化后的字典树，Key 不直接按字节拆解，而每个节点尽力保存最长前缀，大量长 KEY 只在不交叉时耗内存。
- 动态基数树：查找优化后的基数树
  - 分离前缀节点和数据节点：插入的值放只放到叶子节点，内部节点只存放公共前缀。
  - 内部节点分类：按前缀容量分为 4, 16, 48 和 256 四个等级，各等级在查找时都能保证快速，只在必要时膨胀节省内存。

详细的概念和算法伪代码参考论文：[ART.pdf](https://db.in.tum.de/~leis/papers/ART.pdf)



## 2.  插入

- 流程：从根节点开始下沉，不断比较公共前缀并裁剪，最终找到则更新值，未找到则生成叶子节点。
- 有 3 种 case
  - 不分裂：插入根节点，或一般情况下插入新叶子节点。
  - 会分裂：分裂已有叶子节点、分裂内部节点。

### 2.1.  插入新叶子节点

- 首次插入：直接替换 root
- 插入新叶子节点：有序插入到父节点的 `Keys` 和 `Children` 数组，注意处理叶子节点和内部节点完全匹配的 case

![](https://images.yinzige.com/insert-inner-leaf.jpg)

### 2.2.  分裂旧叶子节点

- 完整比较 KEY，若已存在则更新值。若不存在则进行 lazy expansion：
- 比较并裁剪出最长公共前缀 Str，用 Str 生成新的 Node4 内部节点。
- 有序修改 2 个指针：
  - 断开父节点指向旧叶子节点的指针，改为指向新内部节点。
  - 新内部节点新增两个指针，有序指向新、旧叶子节点。

![](https://images.yinzige.com/insert-lazy-expansion.jpg)

### 2.3.  分裂内部节点

当 Key 与内部节点前缀不完全匹配时，类比 2.2，比较公共前缀并生成 Node4 内部节点，有序修改指针。
注意：旧内部节点的前缀在比较时需裁剪。

![](https://images.yinzige.com/insert-inner-lazy-expansion.jpg)

## 3.  节点膨胀

当内部节点的子节点个数达到上限时，需向上膨胀扩容。

### 3.1.  Node4 到 Node16

二者均要求 `Keys` 有序，且与 `Children` 在索引上要一一对应，扩容到更大的 Node16 时则直接复制。



### 3.2.  Node16 到 Node48

将 Node16 的 16 个 Key 按 ASCII 码值作为索引落到 Node48 的 256 个 `Keys` 中，旧 `Children` 则直接复制。

![](https://images.yinzige.com/20200711135310.png)注：此处在实现时，Node48 的 256 个 `Keys` 初值都是 0，即都指向 `Children[0]`，需手动错位，参考 [node_op.go](https://github.com/wuYin/trees/blob/197436713a2859e42fff5069ee8310677ba0bad1/art/node_op.go#L67)



### 3.3.  Node48 到 Node256

将 Node48 的 256 个 Key 直接复制，但 48 个无序 Child 要对应 Key 有序地放入 Node256 的 `Children` 

![](https://images.yinzige.com/20200711135402.png)注：单个字节范围是 `0~255`，Node256 设计上就能全部容纳，不再膨胀。





## 4.  搜索

搜索是插入和删除的公共操作。

- 流程：不断比较前缀并下沉，直到遇到叶子节点，再比较 Key 即可。
- 模式：ART 对下沉时比较公共前缀做了优化，每个内部节点存储了其公共前缀的长度，但是只存储了一部分公共前缀（有上限，控制内存占用），比较 Key 时需动态切换：
  - 悲观模式：若 Key 比部分前缀短，则在当前节点内比较。
  - 乐观模式：若 Key 比部分前缀长，则继续下沉找出完整前缀（通常为最左叶子节点的 Key，最快）来比较。

如下图的 ART，内部节点前缀长度上限为 8，若：

- 搜索 Key 是 **"0x1234567"**，则在内部节点  **"12345678"** 处为悲观模式，直接比较前缀，发现没有值为 `'7'` 的叶子。
- 搜索 Key 是 **"0x123456789ABCY"**，则会切换为乐观模式，让搜索 Key 与最左叶子节点的完整 Key 进行比较，发现 `'Y'` 与 `'D'`  不匹配。

<img src="https://images.yinzige.com/20200711144201.png" style="zoom:50%;" />





## 5.  删除

删除是插入的镜像操作。插入时节点会膨胀，前缀要裁剪；则删除时节点会收缩，前缀要合并。

- 流程：从根节点开始下沉，不断比较公共前缀，最终找到并删除叶子节点。
- 同样有 3 种 case
  - 不收缩：删除根节点（整棵树，没 GC 兜底则要做好内存释放），或一般情况下直接删除叶子节点。
  - 会收缩
    - 内部节点压缩：父节点只有一个叶子节点时，合并父子节点为新叶子节点，即论文中的 path compression
    - 内部节点降级：内部节点的子节点个数少于下限时，则降级收缩。

### 5.1.  删除叶子节点

父节点将其直接从 `Keys` 和 `Children ` 中删除，保证删除后依旧有序，注意叶子节点替换的 case

![](https://images.yinzige.com/20200711163347.png)

### 5.2.  路径压缩

类似 5.1，将 Node4 父节点替换为子节点即可，但注意要合并前缀。

![](https://images.yinzige.com/20200711163138.png)

### 5.3.  内部节点降级

用低一级的满载新节点来替换，5.1 其实是 Node4 降级为 Leaf 的特殊情况。

![](https://images.yinzige.com/20200711162415.png)





## 6. 总结

ART 的创新点在于对节点进行分级，提高查询速度的同时，尽可能地减少内存占用。

对于 Key 每个字节的搜索，Node4 最多只需对 4 个 Key 迭代查询，Node16 可用 CPU SSE 指令优化，而 Node48 和 Node256 则直接是 O(1) 的读操作。

有得有失，如此设计增加了实现的复杂度，必须处理好节点膨胀、路径压缩和降级等操作。细节上也要注意，如节点替换会涉及到指向指针的指针，必须避免指针指向无效的地址；Node16 膨胀到 Node48 时要将索引错位，Node48 查询时同样错位等等。



## 7. 参考

[Multi-ART](https://zhuanlan.zhihu.com/p/65414186)

[Radix Tree在Hyper中的实现](https://blog.csdn.net/matrixyy/article/details/70182527)