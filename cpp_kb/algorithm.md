# 数据结构与算法知识点

## 重点数据结构

- hash表
    将 key 序列化之后再经过 hash 归到一个区间内，根据 hash 值确定 value 下标。
    采用数组存储 value ，为了维持平均 O(1) 的时间复杂度，必须保证 count/size < 1 。
    - hash 碰撞的解决
        当两个不同 key 值通过 hash 函数之后得到相同的 hash 值，则发生了碰撞。 hash 函数并不能保证无碰撞，因为原则上 hash 是将无限的值域通过计算映射到有限的值域中，所以产生碰撞是在所难免的，好的 hash 函数只能尽量减小碰撞的概率。
        当碰撞发生时，可以通过一些方法来解决。
        - 探测法
            有线性，二次等多种规则的探测法，本质上就是当插入一个值时发现数组中位置已经被占用，则采取一个固定规则确认下一个插入位置。
        - 开链法
            数组每个位置实际上是一个链表，当碰撞发生时直接将元素插入链表中。
    
        探测法和开链法在发生碰撞时都无法达到 O(1) 的复杂度。
        探测法的好处是不需要额外的数据结构支持，直接用数组即可，实现较为简单。
        开链法的好处是不会像探测法那样，发生碰撞的元素可能会因为插入了其他地方而占用原来正确 hash 的元素的位置，即不会造成碰撞放大。

    当插入使得 count > size 时，则需要扩容。
    - 扩容与 rehash
        一次性将所有元素都 rehash 是很耗时的，在应用中一般不允许这种突然的耗时操作。所以一般扩容采用 bulk rehash 的方式，开启一个扩容的新 hash表 ，在每次 hash表 操作的时候都将一小部分 rehash ，最终使得所有元素 rehash 。
        - 容量与质数与 2<sup>n</sup>
            容量的选择与 hash表 的效果息息相关。
            2<sup>n</sup> 与快速 hash 求模运算。
            质数与更均匀的 hash 分布。

- LRU
    least recent used ，基于最近最少被用到的淘汰策略的页面置换算法(或缓存淘汰策略)。
    对于 LRU 队列， get 操作查找元素是否在队列中(如果在队列中则移动到尾部)， put 操作将元素放入队尾(如果在队列中则移动到尾部)。因为元素会在队列中频繁移动，所以存储结构选用双向链表最为合适。
    要做到 get 和 put 都是 O(1) 复杂度，光使用链表是无法做的，所以为了加速查询操作，使用 hash表 来作为索引， hash表 value 所指向的是链表上的节点，如此便可以移动节点位置而不会导致 hash表 索引的失效，并且查询操作的复杂度变为 O(1) 。
    在编写代码时要注意节点为头节点或者尾节点的情况，头尾节点要注意其指针指空。
    
    代码示例 python3:
    ```python
    class LinkNode:

        def __init__(self, key: int = 0, value: int = 0):
            self.key = key
            self.value = value
            self.next = None
            self.last = None

    class LRUCache:

        def __init__(self, capacity: int):
            self.capacity = capacity
            self.count = 0
            self.keyIndex = dict()
            self.lruHead = None
            self.lruTail = None

        def get(self, key: int) -> int:
            if self.capacity == 0:
                return -1
            if key not in self.keyIndex:
                return -1
            if key == self.lruTail.key:
                return self.lruTail.value
            node = self.keyIndex[key]
            node.next.last = node.last
            if key == self.lruHead.key:
                self.lruHead = node.next
            else:
                node.last.next = node.next
            self.lruTail.next = node
            node.last = self.lruTail
            self.lruTail = node
            self.lruTail.next = None
            return self.lruTail.value

        def put(self, key: int, value: int) -> None:
            if self.capacity == 0:
                return
            if self.count == 0:
                node = LinkNode(key, value)
                self.keyIndex[key] = node
                self.lruHead = node
                self.lruTail = node
                self.count += 1
                return
            passValue = self.get(key)
            if passValue == value:
                return
            if passValue != -1:
                self.lruTail.value = value
                return
            node = LinkNode(key, value)
            self.keyIndex[key] = node
            self.lruTail.next = node
            node.last = self.lruTail
            self.lruTail = node
            if self.count + 1 > self.capacity:
                self.keyIndex.pop(self.lruHead.key)
                self.lruHead = self.lruHead.next
                self.lruHead.last = None
                return
            self.count += 1
    ```

    - leveldb 的 LRUCache
        其实 leveldb 所实现的更像是 2Q 算法。

- 并查集(Union Find)

    是一种特别的树形(森林)结构，互相有关系的元素会在同一颗树上。它有两种操作，一种是 find 操作，查找一个元素所在树的树根元素；一种是 union 操作，可以连接两个元素所在的树使得它们并为一棵树。
    并查集的设计有不同考量，一般会参考应用场景进行不同的设计。一般要提高 find 的效率就会牺牲 union 的效率，提升 union 的效率就会牺牲 find 的效率。还有综合 union 和 find 的效率的方式。
    - 快速 find
        find O(1) 复杂度，union O(n) 复杂度。
        ```python
        class qfind:
            def __init__(self, size: int):
                self._uf = [i for i in range(size)]

            def find(self, p: int) -> int:
                return self._uf[p]
            
            def union(self, p: int, q: int) -> None:
                p = self.find(p)
                q = self.find(q)
                if p == q:
                    return
                for i in range(len(self._uf)):
                    if self._uf[i] == p:
                        self._uf[i] = q
        ```
    - 快速 union
        union 和 find 的复杂度都为树高度。union 最差 O(n) 复杂度，find 最差 O(n) 复杂度。
        ```python
        class qunion:
            def __init__(self, size: int):
                self._uf = [i for i in range(size)]
            
            def find(self, p: int) -> int:
                while p != self._uf[p]:
                    p = self._uf[p]
                return p

            def union(self, p: int, q: int) -> None:
                p = self.find(p)
                q = self.find(q)
                if p != q:
                    self._uf[p] = q
        ```
    - 加权的快速 union (并且进行路径压缩)
        为了防止树过于高，采用一个额外的数组来存储节点作为根节点的树的高度，这样在 union 的时候就可以让小树总是作为大树根节点的分支，从而将最坏情况变为 O(log n)。
        - 路径压缩
            并查集最优的状态应该是所有节点都是直接连接到最终根节点上。所以有时我们可以在 find 的时候将途经的节点指向最终节点。不过大多数时候加权的方式已经够用了。
            而且在路径压缩的过程中不需要改变高度数组。
            加权加上路径压缩之后平均时间复杂度降到 Ackerman 函数的反函数，非常接近 O(1) 。
        ```python
        class wqunion:
            def __init__(self, size: int):
                self._uf = [i for i in range(size)]
                self._wg = [1 for i in range(size)] # 存该点为根的树的高度

            def find(self, p: int) -> int:
                # while p != self._uf[p]:
                #     p = self._uf[p]
                # return p
                # 路径压缩的方式有很多种，这种递归的方式会带来函数调用的损耗
                # 所以有时可以选择一些压缩不太彻底的方式
                if p != self._uf[p]:
                    self._uf[p] = self.find(self._uf[p])
                return self._uf[p]
            
            def union(self, p: int, q: int) -> None:
                p = self.find(p)
                q = self.find(q)
                if p == q:
                    return
                if self._wg[p] < self._wg[q]:
                    self._uf[p] = q
                    # 此时树的高度不会增加
                else:
                    self._uf[q] = p
                    if self._wg[p] == self._wg[q]:
                        # 只有在合并两个相等的树的时候才用将树高度加1
                        self._wg[p] += 1
        ```

- 各种平衡树
    树是一种特殊的图，n 个节点和 n-1 条边，根据特定规则维护的树结构被运用在各个场景中用来加速搜索。树能够将很多 O(n) 复杂度的搜索过程转换成 O(log n) 的复杂度(树结构比较均匀时)，因为用于搜索的树结构一般是剪裁式搜索，能够通过当前节点信息确定目标所在子树，然后便可以进入目标子树，裁剪掉其他子树的搜索。
    - 一般的二叉排序树
        (在没有重复元素的情况下)左子树的元素都小于当前节点，右子树的元素都大于当前节点。根据这一规则所维护的树结构就是二叉排序树。
        - 搜索
            在未找到节点之前总会根据当前节点的值选择左或者右子树继续查找，直到查到节点没有相应子树为止。
        - 插入
            与搜索过程相似，若值不在树中，则最终会找到一个这样的节点，它缺少相应的孩子，所以新加入的节点可以直接加入作为叶子节点。
        - 删除
            如果该节点只有单棵子树，则可以子承父位然后删除该节点。如果该节点有两个孩子，则需要找到左子树的最右边节点，然后让该节点代替自己的位置。可以知道该节点只有左孩子，所以让其左孩代替其位置就好。
    - AVL 树
        最早提出的自平衡树，也是平衡度最高的，任意节点的子树最大高度差为 1 。在树更新的过程中需要时时注意平衡。节点有平衡因子信息，平衡因子即左右子树的高度差，当一颗树的平衡因子绝对值大于 1 时，则表明该树需要平衡。
        - 搜索
            与普通二叉排序树一样。
        - 插入
            插入操作和二叉排序树相同，但是插入后要考虑平衡操作。插入节点为总是叶子节点，则要往父节点方向检查父节点为根的树是否平衡(检查平衡因子)，不平衡则需要做相应旋转，旋转的方式根据节点和节点左右孩子的平衡因子共同决定，并且可能会改变高度。当检查到某节点发现该节点为根的树高度没有改变时则可以退出了。
        - 删除
            维基百科上说:将要删除的节点通过旋转操作移动到叶子节点，然后直接删除即可。emmm，感觉这样做删除完还是得回溯，因为很难确保删除之后树高度不变。
            一般的实现是像二叉排序树那样删除，删除之后再根据高度变化做旋转，类似插入操作。
    - 红黑树
        也是一种自平衡二叉树，但是不像 AVL 树这样严格。
    - 多路搜索树
        - b树
            不再限定是一个二叉树，而是限定为m阶(m路查找树)，(除根节点外，根节点必须包含最少一个关键字)每个节点下关键字个数在 m/2-1 到 m-1 之间，而儿子节点个数(度)在 m/2 到 m 之间。它们属于自平衡树，它们的平衡不是依靠旋转来维护的，而是通过分裂和合并来维护的。
            这种结构是专门为了管理大规模数据而存在的。
            - 搜索
                在节点上的搜索使用二分，找到对应值或者确定区间进入下一个节点。
            - 插入
                插入元素也是通过，下层节点的分裂导致上层节点关键词增加，超过上限继续分裂，根节点的分裂最终让树高增加。
        - b+树

- 跳表(skiplist)

- 堆

## 重要的算法过程

- 排序

    - 选择和冒泡
    - 插入排序
    - 希尔排序
    - 快速排序(快排)与三种优化
