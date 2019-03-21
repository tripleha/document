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
    - 一般的二叉排序树(BST)
        (在没有重复元素的情况下)左子树的元素都小于当前节点，右子树的元素都大于当前节点。根据这一规则所维护的树结构就是二叉排序树。
        - 搜索
            在未找到节点之前总会根据当前节点的值选择左或者右子树继续查找，直到查到节点没有相应子树为止。
        - 插入
            与搜索过程相似，若值不在树中，则最终会找到一个这样的节点，它缺少相应的孩子，所以新加入的节点可以直接加入作为叶子节点。
        - 删除
            如果该节点只有单棵子树，则可以子承父位然后删除该节点。如果该节点有两个孩子，则需要找到左子树的最右边节点，然后让该节点代替自己的位置。可以知道该节点只有左孩子，所以让其左孩代替其位置就好。
    - 树堆(Treap)
        BST 和 heap 的结合，在树的节点增加随机的权重，BST 本身的 key 用于排序，附加的随机权重用于维护堆的特性，只有左旋右旋两种旋转模式。插入和删除都先按照 BST 来，然后再根据权重来旋转，目标是维护堆的性质。因为随机的特性，所以复杂度能够满足 O(log n)。效率并不是特别高，但是比起其他平衡树实现上要简单一些，这里就不详细说了。
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
        2-3-4 树和红黑树是等价的，而 2-3-4 树是一种4阶b树。这里就不深入探讨了，感兴趣的可以去搜索相关博客或者自己根据两者的特点自己思考(所有红色节点和其父节点合并，就可以变成一个 2-3-4 树了)。
        红黑树也是一种自平衡二叉树，但是不像 AVL 树这样严格。
        - 红黑树的规范:
            1.根节点必须是黑色的。
            2.空节点是黑色的，叶子节点都有两个空子节点。
            3.每个红色节点的子节点必须是黑色的，也就是搜索路径上不能有两个连续的红色节点。
            4.每条从根节点到叶子节点的搜索路径上的黑色节点数量相同。(所以能够保证复杂度在 O(log n) )
        - 搜索
            与普通二叉排序树一样。
        - 插入
            若插入的值不存在于树中，则该节点一定被作为叶子节点插入。为了不违背 4 ，插入的叶子节点必定是红色的。如果父节点也是红色节点，此时违背了 3 。则必须进行调整。这个调整的过程与其叔叔节点(父节点的兄弟节点)的颜色息息相关。
            - 叔叔节点为黑色
                只需要进行父亲节点到祖父节点的旋转，并转换它们的颜色即可解决问题，不需要回溯。emmm 具体的过程就不赘述了，脑内风暴。。。
            - 叔叔节点为红色
                把父亲节点和叔叔节点染黑，然后祖父节点染红。然后以祖父节点为插入节点的视角进行回溯，重复上述判断。若是根节点被染红，则直接变回黑色，完整搜索路径黑色节点增加 1 。
            
            从上面的分析中可以看出，最多只需要一轮旋转，其他步骤都是在染色，对树的结构改动不像 AVL 树那样频繁。
        - 删除
            像普通二叉排序树那样选择删除节点(需要找前继节点替换时不需要交换颜色，只交换值就行)。这样选出的删除节点一定没有右孩子(或只有右孩子)。这时如果被删除节点是红色的，则直接正常删除再拼接孩子即可。但是如果被删除节点是黑色的，那么直接删除会违反 4 ，此时如果该节点有一个孩子，则它的孩子肯定是红色的。则只需替换值删除即可(删红留黑)。
            如果被删除的节点是叶子节点(度为0)。则直接删除它一定违反 4 。此时需要进行调整，和插入情况相反，此时是路径变短，所以思想是需要新的黑节点来补充，或者整个树路径都变短。
            - 兄弟节点为黑色
            - 兄弟节点为红色
                通过旋转来转化为兄弟是黑色的情况。因为兄弟是红色，则兄弟的侄子一定是黑色，且一定有两个黑色侄子，所以按父亲节点旋转，则有一个侄子会变成兄弟节点。转完原兄弟和父亲节点交换颜色，原兄弟节点变成祖父节点。
    - 多路搜索树
        - b树
            不再限定是一个二叉树，而是限定为m阶(m路查找树)，(除根节点外，根节点必须包含最少一个关键字)每个节点下关键字个数在 m/2-1 到 m-1 之间，而儿子节点个数(度)在 m/2 到 m 之间。它们属于自平衡树，它们的平衡不是依靠旋转来维护的，而是通过分裂和合并来维护的。
            这种结构是专门为了管理大规模数据而存在的。因为在单个节点上的索引引入了新的复杂度，所以具体用起来可能效率不如常用的内存式索引数据结构。b树的节点是对应持久化于外存上的。所以一般讨论b树的复杂度是以IO次数为计量的，因为相比于单节点上的内存搜索，IO操作更加耗时(就不是一个数量级的)。
            - 搜索
                在节点上的搜索使用二分，找到对应值或者确定区间进入下一个节点。
            - 插入
                插入操作过程与搜索类似，承接插入操作的总是叶子节点。当节点因插入操作元素个数超过阈值时会进行分裂，分裂会使得中间元素进位到上层节点，这样下层节点的分裂导致上层节点关键词增加，如果超过上限则上层节点继续分裂，根节点的分裂最终让树高增加 1 。
            - 删除
                对比于插入操作，删除操作不一定发生在叶子节点上。当发生在非叶子节点上时找到直接后继键值替补，后继键值一定位于叶子节点，这时再在叶子节点删除后继键值。如果发生在叶子节点上则直接删除即可。叶子节点键值删除后会导致数量小于下限，此时需要借助相邻的兄弟节点来解决问题。如果兄弟节点有足够的键值数量则进行键值转移。如果相邻的兄弟节点键值数量都不够则进行合并，合并需要两节点中间的父节点键值下移一起合并，最终导致父节点键值减少。父节点若键值减少，则也需要进行同样操作。若根节点向下合并，则树高减少 1 。

            b树操作的设计准则是尽量减少树结构的改变。相比键值在节点间转移，b树结构的改变的消耗更大。
        - b+树
            是b树的一种变种，将记录都保存在叶子节点，每次搜索都必须要搜索到叶子节点才能取得记录。
            对比于b树，非叶子节点(索引节点)因为不需要存储记录，可以用更小的空间存储单个节点。而且b+树对于范围查询非常友好，因为底层叶子节点会互相按照顺序链接成链表，范围查询可以在确定首位后直接扫描链表完成。
            - 搜索
                和b树一样，不过要注意此时确定子节点是范围是半开半闭的。搜索用时稳定，因为每次都需要搜索到叶子节点。
            - 插入
                和b树一样，不过要注意子节点会包含父节点的键值。
            - 删除
                删除一定会发生在叶子节点，索引节点的键值只是帮助索引的，所以能不动就不需要动。和b树原理差不多。

            b+树还有另一种描述，即每个节点的键值个数和子节点个数相同的描述，每个键值为子节点的开头第一个元素。
        - b*树
            是b+树的变种。对于b+树的优化主要体现在空间利用放面。它规定每个节点必须至少包含 2/3 键值。为了支持这一规定，每个节点还要维护指向下一个兄弟节点的指针。虽然提高了空间利用率，但是也使得插入删除操作的维护成本提高。

- 跳表(skiplist)
    有序数组支持随机访问，可以在搜索之中使用二分法使得搜索复杂度为 O(log n) 。但是非尾部的插入和删除操作需要移动元素，而且有序数组需要连续的内存空间，所以这些条件限制了有序数组的使用场合。更多情况下我们采用的是链式结构。有序链表对比数组的好处是不需要预分配连续内存空间，插入删除没有移动元素的问题，缺点是无法支持随机访问，所以无法使用二分法来加速搜索。
    跳表是空间换时间的思路的产物。跳表最下层的链表就是本体有序链表，在上层还有多层链表。上层的多级链表是进行索引用的。这样有点像b+树的概念，上层节点是索引节点，不过又有不同，因为多层链表之间的节点都是公用的，多层链表实际上是每个节点内部的链接数组实现的。
    跳表通过良好的上层索引链表的节点分布达到了 O(log n) 的复杂度。跳表第 0 层是本体有序链表，所有元素都可以在这一层访问到。然后往上层，每层包含节点数量减半(概率因子为 2 时)。因为这一过程使用概率计算达到的，所以在数据量足够的情况下，每层节点都可以看作均匀分布的，那么每次搜索都可以过滤掉一半剩余范围的节点，如此看来就和二分有异曲同工之妙了。关于概率因子对于跳表效率的影响有专业的解析，这里就不具体说了，一般认为 e 是最佳的，实际应用一半取 2 。

    https://stackoverflow.com/questions/31580869/skip-lists-are-they-really-performing-as-good-as-pugh-paper-claim/34003558#34003558
    关于跳表和平衡BST的性能争论就没停过。这个帖子下面赞数最多的回答最后一句话非常不错。
    A technique is only as good as it can be applied in the hands of the implementor.
    我个人认为，作为同复杂度的数据结构，其实 skiplist 和 平衡BST 都有其独特的优势，我们更应该关注它们独特的一面，而不是在相同复杂度的地方做实现上的比较(插查删)。。。相比于 平衡BST ，跳表在范围查询的表现上更加好，但空间成本稍高。

    代码示例 python3
    ```python
    import random

    class SkipList:

        _max_level = 32 # 索引层的最大高度

        def __init__(self):
            self._level = 0
            self._head = sl_Node() # 跳表第一个节点不做任何记录，只是开头
            self._head._next = [None for i in range(self._max_level+1)]
            self._head._num = self._max_level

        def insert(self, key: int, value: int) -> None:
            node = self._head
            update = [None for i in range(self._max_level+1)] # 记录该插入节点各层前驱的数组
            for i in range(self._level, -1, -1):
                # 先找到当前跳表最大层数内可能的所有前驱
                # 每层前驱要么指到末尾，要么指到比插入节点 key 大的节点
                while node._next[i] is not None and node._next[i]._key < key:
                    node = node._next[i]
                update[i] = node
            node = node._next[0] # 最终找到了第 0 层
            if node is None or node._key != key:
                now_level = 0
                while random.randint(0, 1) == 0 and now_level < self._max_level:
                    now_level += 1
                # 上面的概率计算可以保证跳表的节点每层分布的数量比例稳定
                # 等比数列 1/2 1/4 1/8 1/16 ...
                if now_level > self._level:
                    # 用 head 补足前驱位置
                    for i in range(self._level+1, now_level+1):
                        update[i] = self._head
                    self._level = now_level
                node = sl_Node()
                node._key = key
                node._value = value
                node._num = now_level
                node._next = [None for i in range(now_level+1)]
                for i in range(now_level+1):
                    node._next[i] = update[i]._next[i]
                    update[i]._next[i] = node
            elif node._key == key:
                node._value = value

        def erase(self, key: int) -> None:
            # 删除与插入类似，都是先找前驱
            # 然后修改前驱指针指向删除节点的同层后继
            node = self._head
            update = [None for i in range(self._max_level+1)]
            for i in range(self._level, -1, -1):
                while node._next[i] is not None and node._next[i]._key < key:
                    node = node._next[i]
                update[i] = node
            node = node._next[0]
            if node is None or node._key != key:
                # 不存在删除节点
                return
            for i in range(node._num+1):
                if i > node._num:
                    # 已经超过了删除节点的层数
                    break
                update[i]._next[i] = node._next[i]
            while self._level > 0 and self._head._next[self._level] is None:
                # 若删除节点很高会导致所占层数降低
                self._level -= 1

        def find(self, key: int) -> (bool, int):
            node = self._head
            for i in range(self._level, -1, -1):
                while node._next[i] is not None and node._next[i]._key < key:
                    node = node._next[i]
            node = node._next[0]
            if node is None or node._key != key:
                return False, 0
            return True, node._value
    ```
    从代码可以看出跳表的增删查操作逻辑基本差不多，实现简单，红黑树写起来可不止这么几行代码。

- 堆
    - 二叉堆(最大 or 最小)
        因为最大堆和最小堆本质一样，接下来都按最小堆说。
        父节点比子树上的节点小或等的树。
        - 完全二叉树和满二叉树
            为什么这个东西要在这里说呢。。。虽然堆不一定是完全二叉树，但是平时完全二叉堆在实现上我们喜欢用数组，写起来也简单。
            满二叉树是完全二叉树的特例。满二叉树指二叉树每层节点个数都是满的。完全二叉树是除了最后一层其他都是满的，并且最后一层按最左边开始填节点。所以基于这个特性，完全二叉树可以用数组实现，根节点下标为1，对于下标为 n 的节点，其左孩子下标为 2n ，右孩子下标为 2n+1 。
        - 插入
            先将插入元素放至尾部，然后将其与父节点比较，若小则上浮，重复相同步骤。
        - 删除
            将末尾的元素填到首部，然后若其比其一子节点大则将其与子节点中最小的交换，重复相同步骤。
        ```python
        class MinHeap:
            # 最小二叉堆实现
            def __init__(self):
                self._hp = [0,]
                self._sz = 0

            def top(self) -> int:
                return self._hp[1] if self._sz > 0 else 0

            def push(self, key: int) -> None:
                self._hp.append(key)
                self._sz += 1
                if self._sz == 1:
                    return
                idx = self._sz
                pidx = idx >> 1
                while pidx > 0 and self._hp[pidx] > self._hp[idx]:
                    # 将插入元素上浮至正确位置
                    # 整形能够直接这样交换值
                    self._hp[idx] ^= self._hp[pidx]
                    self._hp[pidx] ^= self._hp[idx]
                    self._hp[idx] ^= self._hp[pidx]
                    idx = pidx
                    pidx >>= 1

            def pop(self) -> int:
                if self._sz == 0:
                    return 0
                if self._sz == 1:
                    self._sz -= 1
                    return self._hp.pop()
                return_value = self._hp[1]
                self._hp[1] = self._hp.pop()
                self._sz -= 1
                # 用末尾元素覆盖首位
                idx = 1
                sidx = idx << 1
                while sidx < self._sz:
                    # 将当前首位元素下移至正确位置
                    if self._hp[sidx] > self._hp[sidx+1]:
                        sidx += 1
                    if self._hp[sidx] >= self._hp[idx]:
                        break
                    self._hp[idx] ^= self._hp[sidx]
                    self._hp[sidx] ^= self._hp[idx]
                    self._hp[idx] ^= self._hp[sidx]
                    idx = sidx
                    sidx <<= 1
                if sidx == self._sz and self._hp[sidx] < self._hp[idx]:
                    # 如果最后到达了末尾可能会出现只有一个节点的情况是不在上面循环中处理的
                    # 所以需要多比较一次来确保完成下移
                    self._hp[idx] ^= self._hp[sidx]
                    self._hp[sidx] ^= self._hp[idx]
                    self._hp[idx] ^= self._hp[sidx]
                return return_value
        ```
    - 可并堆(针对合并优化，可以实现两个堆的快速合并，适用于经常需要合并堆的场景)
        (挖个坑。。。
        - 二项堆
        - 斐波那契堆
        - 左偏堆

## 重要的算法过程

- 排序

    - 选择和冒泡
        选择排序的过程是每次选择最小的元素和头部交换，然后待排序的数组大小减 1 ，不断重复至全部完成排序。
        冒泡排序是每次比较相邻位置，把大的往后移动，这样达到了每轮遍历都能将待排序数组最大的移动到末尾，然后待排序数组大小减 1 。
        ```python
        def selectSort(src: list) -> list:
            # 选择排序
            result = src[:]
            sz = len(src)
            for i in range(sz):
                tmin = result[i]
                tpos = i
                for j in range(i+1, sz):
                    if tmin > result[j]:
                        tmin = result[j]
                        tpos = j
                if i != tpos:
                    result[i], result[tpos] = result[tpos], result[i]
            return result
        
        def bubbleSort(src: list) -> list:
            # 冒泡排序
            result = src[:]
            sz = len(src)
            for i in range(sz):
                is_sorted = True
                for j in range(sz-i-1):
                    if result[j] > result[j+1]:
                        is_sorted = False
                        result[j], result[j+1] = result[j+1], result[j]
                if is_sorted:
                    break
            return result
        ```
    - 插入排序
        插入排序的算法原理是通过不断减少数组的逆序数量从而使数组最终有序。具体的做法是每次都将未排序区的一个数插入到左侧已经排好顺序的数组中，插入的方式是一步一步的交换过去。
        插入排序的每次交换都能够让逆序数量减少 1 ，所以交换次数时逆序数量，如果数据已经部分有序，那么插入排序效率会比较高。
        ```python
        def insertSort(src: list) -> list:
            # 插入排序
            result = src[:]
            sz = len(src)
            for i in range(sz):
                for j in range(i, 0, -1):
                    # 将原本第 i 位的元素往前交换，插入到正确位置
                    if result[j] < result[j-1]:
                        result[j], result[j-1] = result[j-1], result[j]
                    else:
                        break
            return result
        ```
    - 希尔排序
        是插入排序的优化版本，因为插入排序一次交换只能减少 1 逆序数，所以在数据量大的情况下会比较慢。为了加快逆序数的减少，产生了希尔排序。希尔排序刚开始比较的步长大于 1 ，在排序的过程中会最终变回 1 。这样保证了数组能够最终排序完成。而因为一刚开始比较的步长大，所以交换是最少也能减少 1 的逆序数，或者能够减少更多的逆序数，所以比插入排序快。其实这个步长就是把数组分割成多个子序列进行插入排序。
        希尔排序不是稳定排序(相等元素无法保证相对位置)。
        - 步长增量序列的选择
            不同的步长增量序列会带来不同的最坏情况下复杂度和平均复杂度。非常复杂，我们记一下结论就好。感兴趣的可以自行查找相关资料。
            ||计算式|最坏时间复杂度|
            |--|--|--|
            |Shell|N/2<sup>k</sup>|O(N<sup>2</sup>)|
            |Hibbard|2<sup>k</sup>-1|O(N<sup>1.5</sup>)|
            |Knuth|h<sub>k</sub>=3*h<sub>k-1</sub>+1|O(N<sup>1.5</sup>)|
            |Sedgewick|max(9*4<sup>j</sup>−9∗2<sup>j</sup>+1,4<sup>k</sup>−3∗2<sup>k</sup>+1)|O(N<sup>4/3</sup>)|
        ```python
        def shellSort(src: list) -> list:
            result = src[:]
            sz = len(src)
            steps = [1, 5, 19, 41, 109, 209, 505, 929, 2161, 3905, 8929, 16001, 36289, 64769, 146305, 260609]
            for h in steps[::-1]:
                if sz <= h:
                    continue
                for i in range(sz):
                    for j in range(i, h-1, -h):
                        if result[j] < result[j-h]:
                            result[j], result[j-h] = result[j-h], result[j]
                            t += 1
                        else:
                            break
            return result
        ```
        通过实验，希尔排序的确能够实现比插入排序更少的交换次数。
    - 快速排序(快排)与三种优化
        快排是最快的通用排序算法。它的原理是每次选取一个数，然后将小于它的元素放一边，大于的放另一边，然后在对该数切分的两个子数组进行排序。
    - 多路归并排序
    - 堆排序

- 树的遍历

    - 前序中序后序层序
    - 线索二叉树
