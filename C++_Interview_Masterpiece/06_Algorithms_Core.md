# 06 — 算法与数据结构核心体系：面试高频到进阶图论/DP

> **定位**：补齐 C++ 面试中高频算法与数据结构体系，强调复杂度、稳定性、边界条件、适用场景和可直接手写的实现框架。

---

## 目录

1. [排序算法全体系](#1-排序算法全体系)
2. [基础算法思想](#2-基础算法思想)
3. [高频数据结构](#3-高频数据结构)
4. [搜索算法：DFS / BFS / 双向 BFS](#4-搜索算法dfs--bfs--双向-bfs)
5. [图论：最短路、最小生成树、拓扑排序](#5-图论最短路最小生成树拓扑排序)
6. [动态规划：线性 DP、背包、LCS、LIS](#6-动态规划线性-dp背包lcslis)

---

## 1. 排序算法全体系

### 1.1 九大排序对比

| 算法 | 平均时间 | 最坏时间 | 额外空间 | 稳定性 | 适用场景 |
|---|---:|---:|---:|---|---|
| 冒泡排序 | O(n²) | O(n²) | O(1) | 稳定 | 教学、小规模、近乎有序可提前退出 |
| 选择排序 | O(n²) | O(n²) | O(1) | 不稳定 | 交换次数少但比较多，工程少用 |
| 插入排序 | O(n²) | O(n²) | O(1) | 稳定 | 小数组、近乎有序，常作快排/归并小区间优化 |
| 希尔排序 | 依 gap | O(n²) | O(1) | 不稳定 | 插入排序改进，工程较少单独使用 |
| 归并排序 | O(n log n) | O(n log n) | O(n) | 稳定 | 链表排序、外部排序、需要稳定性 |
| 快速排序 | O(n log n) | O(n²) | O(log n) | 不稳定 | 通用内存排序，常数小，需防退化 |
| 堆排序 | O(n log n) | O(n log n) | O(1) | 不稳定 | 要求 O(1) 额外空间且不要求稳定 |
| 计数排序 | O(n+k) | O(n+k) | O(k) | 稳定可实现 | 整数范围小 |
| 基数排序 | O(d(n+k)) | O(d(n+k)) | O(n+k) | 稳定 | 定长整数/字符串，位数有限 |
| 桶排序 | 依分布 | O(n²) | O(n+k) | 取决于桶内排序 | 数据均匀分布 |

### 1.2 快速排序

核心思想：选 pivot，把数组划分为 `< pivot`、`== pivot`、`> pivot` 三部分，然后递归处理左右区间。

学习重点：快排不是“永远 O(n log n)”。它的平均性能很好，但如果划分极度不均衡，递归树会退化成链表，高度变成 O(n)，总复杂度变成 O(n²)。

最坏场景：

- 每次 pivot 都选到最小/最大元素。
- 已排序数组却固定选首/尾元素。
- 大量重复元素且使用二路划分。

优化方式：

- 随机 pivot。
- 三数取中。
- 三路划分处理大量重复元素。
- 小区间切换插入排序。
- 递归深度过大时切换堆排序（introsort 思想）。

三路快排框架：

```cpp
void quick_sort(std::vector<int>& a, int l, int r) {
    if (l >= r) return;
    int pivot = a[l + (r - l) / 2];
    int lt = l, i = l, gt = r;
    while (i <= gt) {
        if (a[i] < pivot) std::swap(a[lt++], a[i++]);
        else if (a[i] > pivot) std::swap(a[i], a[gt--]);
        else ++i;
    }
    quick_sort(a, l, lt - 1);
    quick_sort(a, gt + 1, r);
}
```

代码讲解：

- `lt` 左侧始终维护 `< pivot` 区间，`gt` 右侧始终维护 `> pivot` 区间，`i` 扫描未知区间。
- 遇到小于 pivot 的元素，与 `lt` 位置交换后，`lt` 和 `i` 同时前进。
- 遇到大于 pivot 的元素，与 `gt` 位置交换后，只移动 `gt`，因为换回来的元素还没检查。
- 遇到等于 pivot 的元素，只移动 `i`，自然形成中间的 `== pivot` 区间。
- 三路划分能显著优化大量重复元素输入；否则普通二路快排可能在重复值场景下频繁做无效递归。

复杂度：平均 O(n log n)，最坏 O(n²)，递归栈平均 O(log n)、最坏 O(n)。工程实现通常配合随机 pivot、插入排序 cutoff、递归深度保护。

### 1.3 归并排序

特点：稳定，时间复杂度稳定 O(n log n)，代价是 O(n) 额外空间。

```cpp
void merge_sort(std::vector<int>& a, int l, int r, std::vector<int>& tmp) {
    if (l >= r) return;
    int m = l + (r - l) / 2;
    merge_sort(a, l, m, tmp);
    merge_sort(a, m + 1, r, tmp);

    int i = l, j = m + 1, k = l;
    while (i <= m && j <= r) {
        if (a[i] <= a[j]) tmp[k++] = a[i++]; // <= 保证稳定
        else tmp[k++] = a[j++];
    }
    while (i <= m) tmp[k++] = a[i++];
    while (j <= r) tmp[k++] = a[j++];
    for (int p = l; p <= r; ++p) a[p] = tmp[p];
}
```

代码讲解：

- 递归先把左右两半排好，再用双指针 `i/j` 合并到临时数组 `tmp`。
- `a[i] <= a[j]` 时优先取左半边元素，因此相等元素的原始相对顺序不变，这就是稳定性的来源。
- 最后把 `[l, r]` 的合并结果从 `tmp` 拷回原数组。

复杂度：每层合并 O(n)，递归层数 O(log n)，总 O(n log n)；额外空间 O(n)。归并排序非常适合链表排序和外部排序，因为顺序合并对连续随机访问要求低。

### 1.4 堆排序

堆排序使用大根堆：堆顶最大，每次把堆顶放到未排序区末尾。

复杂度：建堆 O(n)，每次调整 O(log n)，总 O(n log n)，额外空间 O(1)，不稳定。

```cpp
void sift_down(std::vector<int>& a, int n, int i) {
    while (true) {
        int largest = i;
        int l = 2 * i + 1, r = 2 * i + 2;
        if (l < n && a[l] > a[largest]) largest = l;
        if (r < n && a[r] > a[largest]) largest = r;
        if (largest == i) break;
        std::swap(a[i], a[largest]);
        i = largest;
    }
}

void heap_sort(std::vector<int>& a) {
    int n = static_cast<int>(a.size());
    for (int i = n / 2 - 1; i >= 0; --i) sift_down(a, n, i);
    for (int end = n - 1; end > 0; --end) {
        std::swap(a[0], a[end]);
        sift_down(a, end, 0);
    }
}
```

代码讲解：

- `sift_down` 假设左右子树已经是堆，只需要把位置 `i` 的元素向下调整到正确位置。
- 建堆从最后一个非叶子节点 `n / 2 - 1` 开始向前调整；叶子节点天然满足堆性质。
- 排序阶段每次把最大值 `a[0]` 交换到末尾，然后在缩小后的堆上继续下沉堆顶。
- 堆排序不稳定：相等元素可能因为堆调整和交换改变相对顺序。

复杂度：建堆 O(n)，每次取最大 O(log n)，总 O(n log n)，额外空间 O(1)。适合空间敏感但不要求稳定性的场景。

---

## 2. 基础算法思想

### 2.1 二分查找边界处理

推荐使用左闭右开 `[l, r)`，减少边界歧义。

```cpp
int lower_bound_index(const std::vector<int>& a, int target) {
    int l = 0, r = static_cast<int>(a.size());
    while (l < r) {
        int m = l + (r - l) / 2;
        if (a[m] < target) l = m + 1;
        else r = m;
    }
    return l; // 第一个 >= target 的位置
}
```

代码讲解：

- 区间定义为 `[l, r)`，所以初始 `r = a.size()`，循环条件是 `l < r`。
- 当 `a[m] < target`，答案不可能在 `m` 及其左侧，令 `l = m + 1`。
- 否则 `m` 可能就是第一个 `>= target` 的位置，不能丢掉，令 `r = m`。
- 循环结束时 `l == r`，该位置就是插入点；如果返回值等于 `a.size()`，表示所有元素都小于 target。

常见错误：

- `mid = (l + r) / 2` 可能溢出。
- 更新边界时没有排除 `mid`，导致死循环。
- 没有明确找的是第一个、最后一个还是任意一个。

### 2.2 贪心算法

贪心每一步做局部最优选择。可用条件：局部最优能推出全局最优，通常需要交换论证或反证证明。

典型题：区间调度、跳跃游戏、分发饼干、最小箭数引爆气球。

### 2.3 回溯算法

回溯是 DFS + 状态恢复。

```cpp
void backtrack(State& state) {
    if (is_solution(state)) {
        record(state);
        return;
    }
    for (auto choice : choices(state)) {
        apply(state, choice);
        backtrack(state);
        undo(state, choice);
    }
}
```

典型题：全排列、组合、子集、N 皇后、数独。

### 2.4 分治思想

分治三步：分解子问题、递归求解、合并结果。典型算法：归并排序、快速排序、二分、最近点对。

---

## 3. 高频数据结构

### 3.1 链表反转

```cpp
ListNode* reverse_list(ListNode* head) {
    ListNode* prev = nullptr;
    ListNode* cur = head;
    while (cur) {
        ListNode* next = cur->next;
        cur->next = prev;
        prev = cur;
        cur = next;
    }
    return prev;
}
```

### 3.2 环检测

Floyd 快慢指针：

```cpp
bool has_cycle(ListNode* head) {
    ListNode* slow = head;
    ListNode* fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) return true;
    }
    return false;
}
```

代码讲解：

- `fast` 每次走两步，`slow` 每次走一步；如果存在环，快指针一定会在环内追上慢指针。
- 循环条件必须同时检查 `fast` 和 `fast->next`，否则 `fast->next->next` 会解引用空指针。
- 时间复杂度 O(n)，空间复杂度 O(1)，比哈希集合记录访问节点更省空间。

找入环点：相遇后，一个指针回到 head，两个指针每次走一步，再次相遇即入口。原因是：设头到入口距离为 `a`，入口到相遇点距离为 `b`，环剩余距离为 `c`，相遇时有 `2(a+b)=a+b+k(b+c)`，可推出 `a` 与 `c` 在模环长意义下等价。

### 3.3 有序链表合并

```cpp
ListNode* merge_two_lists(ListNode* a, ListNode* b) {
    ListNode dummy{0};
    ListNode* tail = &dummy;
    while (a && b) {
        if (a->val <= b->val) {
            tail->next = a;
            a = a->next;
        } else {
            tail->next = b;
            b = b->next;
        }
        tail = tail->next;
    }
    tail->next = a ? a : b;
    return dummy.next;
}
```

代码讲解：

- `dummy` 哨兵节点避免单独处理头节点为空或第一次接入节点的特殊情况。
- `tail` 永远指向结果链表最后一个节点，每接入一个节点后向后移动。
- 该实现复用原链表节点，不创建新节点，因此空间 O(1)；但输入链表结构会被改写。
- 使用 `<=` 时，相等元素优先取链表 `a`，可以保持两个输入链表内部的相对顺序。

### 3.4 二叉树遍历

递归前序/中序/后序：

```cpp
void preorder(TreeNode* root) {
    if (!root) return;
    visit(root);
    preorder(root->left);
    preorder(root->right);
}
```

层序遍历：

```cpp
std::vector<std::vector<int>> level_order(TreeNode* root) {
    std::vector<std::vector<int>> ans;
    if (!root) return ans;
    std::queue<TreeNode*> q;
    q.push(root);
    while (!q.empty()) {
        int n = static_cast<int>(q.size());
        auto& level = ans.emplace_back();
        for (int i = 0; i < n; ++i) {
            TreeNode* cur = q.front(); q.pop();
            level.push_back(cur->val);
            if (cur->left) q.push(cur->left);
            if (cur->right) q.push(cur->right);
        }
    }
    return ans;
}
```

代码讲解：

- 队列中保存当前层的节点，进入每一轮时先记录 `q.size()`，这个大小就是当前层节点数。
- 内层循环只处理当前层；新加入的左右孩子留到下一轮处理。
- `emplace_back()` 新建一层数组并返回引用，减少额外查找最后一层的代码。

最近公共祖先：

```cpp
TreeNode* lca(TreeNode* root, TreeNode* p, TreeNode* q) {
    if (!root || root == p || root == q) return root;
    TreeNode* left = lca(root->left, p, q);
    TreeNode* right = lca(root->right, p, q);
    if (left && right) return root;
    return left ? left : right;
}
```

代码讲解：

- 如果当前节点为空，或当前节点就是 `p/q`，直接返回当前节点，表示“在这棵子树里找到了目标或没有目标”。
- 分别在左右子树查找：如果左右都非空，说明 `p` 和 `q` 分别位于两侧，当前 `root` 就是最近公共祖先。
- 如果只有一侧非空，说明两个目标都在那一侧，或目前只找到一个目标，继续向上返回该结果。

复杂度：每个节点最多访问一次，时间 O(n)；递归深度等于树高，空间 O(h)。

### 3.5 哈希表实现原理

核心组件：

- hash 函数：把 key 映射到整数。
- bucket 数组：按 `hash % bucket_count` 定位桶。
- 冲突解决：拉链法、开放寻址、再哈希。
- 负载因子：`size / bucket_count`，过高会触发 rehash。

冲突解决对比：

| 方案 | 原理 | 优点 | 缺点 |
|---|---|---|---|
| 拉链法 | 每个桶挂链表/节点 | 删除简单，负载因子可 > 1 | 指针跳转，cache locality 差 |
| 线性探测 | 冲突后向后找空位 | cache 友好 | 聚集问题 |
| 二次探测 | 按二次序列探测 | 缓解聚集 | 删除和扩容复杂 |
| 双重哈希 | 第二个 hash 决定步长 | 分布好 | 计算成本高 |

---

## 4. 搜索算法：DFS / BFS / 双向 BFS

### 4.1 DFS 递归实现

```cpp
void dfs(int u, const std::vector<std::vector<int>>& g, std::vector<int>& vis) {
    vis[u] = 1;
    for (int v : g[u]) {
        if (!vis[v]) dfs(v, g, vis);
    }
}
```

风险：图很深时递归栈溢出。

### 4.2 DFS 手动栈实现

```cpp
void dfs_iter(int s, const std::vector<std::vector<int>>& g) {
    std::vector<int> vis(g.size());
    std::stack<int> st;
    st.push(s);
    while (!st.empty()) {
        int u = st.top(); st.pop();
        if (vis[u]) continue;
        vis[u] = 1;
        for (int v : g[u]) {
            if (!vis[v]) st.push(v);
        }
    }
}
```

### 4.3 BFS 普通队列

BFS 适合无权图最短路。

```cpp
std::vector<int> bfs(int s, const std::vector<std::vector<int>>& g) {
    std::vector<int> dist(g.size(), -1);
    std::queue<int> q;
    dist[s] = 0;
    q.push(s);
    while (!q.empty()) {
        int u = q.front(); q.pop();
        for (int v : g[u]) {
            if (dist[v] == -1) {
                dist[v] = dist[u] + 1;
                q.push(v);
            }
        }
    }
    return dist;
}
```

代码讲解：

- `dist` 同时承担 visited 和距离记录作用，初始为 -1 表示未访问。
- BFS 按层扩展，所以第一次到达某个点时的距离一定是无权图最短距离。
- 每条边最多被检查一次，邻接表下复杂度 O(V+E)。

### 4.4 双向 BFS

适合起点和终点都已知、分支因子较大的无权最短路。每次扩展较小的 frontier。

```cpp
int bidirectional_bfs(int begin, int target, const std::vector<std::vector<int>>& g) {
    if (begin == target) return 0;
    std::unordered_set<int> front{begin}, back{target}, visited{begin, target};
    int step = 0;
    while (!front.empty() && !back.empty()) {
        if (front.size() > back.size()) std::swap(front, back);
        std::unordered_set<int> next;
        ++step;
        for (int u : front) {
            for (int v : g[u]) {
                if (back.count(v)) return step;
                if (!visited.count(v)) {
                    visited.insert(v);
                    next.insert(v);
                }
            }
        }
        front = std::move(next);
    }
    return -1;
}
```

代码讲解：

- `front` 和 `back` 分别表示从起点和终点扩展出的当前边界。
- 每轮扩展节点数更少的一侧，可以把搜索规模从近似 `b^d` 降到 `2*b^(d/2)`，在分支因子大时收益明显。
- 一旦新扩展节点出现在对侧 frontier 中，就找到了两边搜索的交汇点，当前 `step` 即最短距离。
- 适用前提是无权图或每条边代价相同；带权图不能直接用普通 BFS/双向 BFS。

---

## 5. 图论：最短路、最小生成树、拓扑排序

### 5.1 Dijkstra 单源最短路径

适用条件：边权非负。若存在负权边，不能使用 Dijkstra。

朴素实现：适合稠密图，时间 O(V²)。

```cpp
std::vector<int> dijkstra_dense(const std::vector<std::vector<int>>& w, int s) {
    const int INF = 1e9;
    int n = static_cast<int>(w.size());
    std::vector<int> dist(n, INF), used(n);
    dist[s] = 0;
    for (int i = 0; i < n; ++i) {
        int u = -1;
        for (int j = 0; j < n; ++j) {
            if (!used[j] && (u == -1 || dist[j] < dist[u])) u = j;
        }
        if (u == -1 || dist[u] == INF) break;
        used[u] = 1;
        for (int v = 0; v < n; ++v) {
            if (w[u][v] < INF && dist[v] > dist[u] + w[u][v]) {
                dist[v] = dist[u] + w[u][v];
            }
        }
    }
    return dist;
}
```

代码讲解：

- 每轮从未确定最短路的点中选择 `dist` 最小的点 `u`，这一步在朴素实现中需要 O(V) 扫描。
- 由于所有边权非负，一旦 `u` 被选中，它的最短距离就不会再被后续路径改小。
- 邻接矩阵适合稠密图；如果边很多，O(V²) 反而可能比堆优化更简单稳定。

堆优化：适合稀疏图，时间 O((V+E) log V)。

```cpp
using Edge = std::pair<int, int>; // {to, weight}

std::vector<int> dijkstra_heap(const std::vector<std::vector<Edge>>& g, int s) {
    const int INF = 1e9;
    std::vector<int> dist(g.size(), INF);
    using Node = std::pair<int, int>; // {dist, vertex}
    std::priority_queue<Node, std::vector<Node>, std::greater<Node>> pq;
    dist[s] = 0;
    pq.push({0, s});
    while (!pq.empty()) {
        auto [du, u] = pq.top(); pq.pop();
        if (du != dist[u]) continue;
        for (auto [v, w] : g[u]) {
            if (dist[v] > du + w) {
                dist[v] = du + w;
                pq.push({dist[v], v});
            }
        }
    }
    return dist;
}
```

代码讲解：

- 优先队列中可能存在同一个点的旧距离；`if (du != dist[u]) continue;` 用于丢弃过期条目。
- 每次松弛边 `(u, v, w)`，如果经过 `u` 能让 `v` 更短，就更新 `dist[v]` 并把新状态压入堆。
- 不能用于负权边：负权边会破坏“当前堆顶点已经确定最短路”的贪心前提。

### 5.2 Bellman-Ford 与 SPFA

Bellman-Ford 可处理负权边，并可检测从源点可达的负环。复杂度 O(VE)。

```cpp
struct BFEdge { int u, v, w; };

bool bellman_ford(int n, int s, const std::vector<BFEdge>& edges, std::vector<int>& dist) {
    const int INF = 1e9;
    dist.assign(n, INF);
    dist[s] = 0;
    for (int i = 0; i < n - 1; ++i) {
        bool changed = false;
        for (auto [u, v, w] : edges) {
            if (dist[u] != INF && dist[v] > dist[u] + w) {
                dist[v] = dist[u] + w;
                changed = true;
            }
        }
        if (!changed) break;
    }
    for (auto [u, v, w] : edges) {
        if (dist[u] != INF && dist[v] > dist[u] + w) return false; // 有负环
    }
    return true;
}
```

代码讲解：

- 外层最多执行 `n - 1` 轮，因为不含环的最短路径最多包含 `n - 1` 条边。
- 每轮遍历所有边并尝试松弛；如果一整轮没有变化，可以提前结束。
- 第 `n` 轮如果还能继续松弛，说明存在从源点可达的负权环，最短路无定义。

SPFA 是 Bellman-Ford 的队列优化，平均可能较快，但最坏仍可退化到 O(VE)。面试中必须说明不能把 SPFA 当作稳定多项式优化保证。

### 5.3 Floyd 多源最短路

适合点数较小的稠密图，复杂度 O(V³)，可处理负权边，但不能存在负环。

```cpp
void floyd(std::vector<std::vector<int>>& d) {
    int n = static_cast<int>(d.size());
    for (int k = 0; k < n; ++k)
        for (int i = 0; i < n; ++i)
            for (int j = 0; j < n; ++j)
                if (d[i][k] < INF && d[k][j] < INF)
                    d[i][j] = std::min(d[i][j], d[i][k] + d[k][j]);
}
```

代码讲解：

- `k` 是中转点，最外层枚举 `k` 表示“只允许使用编号不超过 k 的点作为中转点”逐步放宽限制。
- 转移 `d[i][j] = min(d[i][j], d[i][k] + d[k][j])` 表示从 i 到 j 的最短路是否可以通过 k 变短。
- 示例中的 `INF` 应定义为足够大的常量；相加前必须判断两段都不是 INF，避免溢出或假路径。

### 5.4 最小生成树 Prim / Kruskal

Prim：从一个点开始，不断选择连接已选集合和未选集合的最小边。适合稠密图，堆优化后适合邻接表。

Kruskal：按边权排序，使用并查集跳过成环边。复杂度 O(E log E)。

```cpp
struct DSU {
    std::vector<int> p, sz;
    explicit DSU(int n) : p(n), sz(n, 1) { std::iota(p.begin(), p.end(), 0); }
    int find(int x) { return p[x] == x ? x : p[x] = find(p[x]); }
    bool unite(int a, int b) {
        a = find(a); b = find(b);
        if (a == b) return false;
        if (sz[a] < sz[b]) std::swap(a, b);
        p[b] = a; sz[a] += sz[b];
        return true;
    }
};
```

代码讲解：

- `find` 使用路径压缩，把查询路径上的节点直接挂到根上，降低后续查询成本。
- `unite` 使用按集合大小合并，把小树挂到大树下面，避免树退化。
- Kruskal 中按边权从小到大尝试加入边；如果 `unite` 返回 false，说明两个端点已经连通，加入该边会成环，应跳过。

### 5.5 拓扑排序

适用于 DAG，用于课程表、构建依赖、任务调度。

```cpp
std::vector<int> topo_sort(const std::vector<std::vector<int>>& g) {
    int n = static_cast<int>(g.size());
    std::vector<int> indeg(n), order;
    for (int u = 0; u < n; ++u)
        for (int v : g[u]) ++indeg[v];
    std::queue<int> q;
    for (int i = 0; i < n; ++i) if (indeg[i] == 0) q.push(i);
    while (!q.empty()) {
        int u = q.front(); q.pop();
        order.push_back(u);
        for (int v : g[u]) if (--indeg[v] == 0) q.push(v);
    }
    if (static_cast<int>(order.size()) != n) return {}; // 有环
    return order;
}
```

代码讲解：

- `indeg[v]` 表示还有多少前置依赖未完成。
- 初始把所有入度为 0 的节点入队，它们没有前置依赖，可以最先执行。
- 每弹出一个节点，相当于完成该任务，所以它指向的后继节点入度减 1。
- 如果最终输出数量小于节点数，说明存在环，无法形成合法拓扑序。

---

## 6. 动态规划：线性 DP、背包、LCS、LIS

### 6.1 线性 DP：爬楼梯

基础版：每次爬 1 或 2 阶。

状态定义：`dp[i]` 表示到第 i 阶的方法数。

转移：`dp[i] = dp[i - 1] + dp[i - 2]`。

```cpp
int climb_stairs(int n) {
    if (n <= 1) return 1;
    int a = 1, b = 1;
    for (int i = 2; i <= n; ++i) {
        int c = a + b;
        a = b;
        b = c;
    }
    return b;
}
```

代码讲解：

- 该问题本质是 Fibonacci：到第 `i` 阶只能从 `i-1` 走 1 阶或从 `i-2` 走 2 阶。
- 只需要前两项，所以用 `a/b` 滚动变量把空间从 O(n) 优化为 O(1)。
- 面试中先写清楚状态定义和转移，再做空间优化，逻辑更容易被接受。

约束变种：不能连续走两次 2 阶。

状态定义：

- `dp1[i]`：到第 i 阶，最后一步走 1 阶的方法数。
- `dp2[i]`：到第 i 阶，最后一步走 2 阶的方法数。

转移：

- `dp1[i] = dp1[i-1] + dp2[i-1]`
- `dp2[i] = dp1[i-2]`，因为不能连续走 2 阶。

### 6.2 01 背包

每个物品最多选一次。

```cpp
int knapsack01(const std::vector<int>& w, const std::vector<int>& val, int cap) {
    std::vector<int> dp(cap + 1);
    for (int i = 0; i < static_cast<int>(w.size()); ++i) {
        for (int c = cap; c >= w[i]; --c) {
            dp[c] = std::max(dp[c], dp[c - w[i]] + val[i]);
        }
    }
    return dp[cap];
}
```

代码讲解：

- `dp[c]` 表示容量不超过 `c` 时能获得的最大价值。
- 对每个物品，只能从上一轮状态转移而来，所以容量必须倒序枚举。
- 如果正序枚举，`dp[c - w[i]]` 可能已经在本轮使用过第 i 个物品，会错误变成完全背包。

容量倒序是关键：防止同一物品被重复使用。

### 6.3 完全背包

每个物品可选无限次。

```cpp
int complete_knapsack(const std::vector<int>& w, const std::vector<int>& val, int cap) {
    std::vector<int> dp(cap + 1);
    for (int i = 0; i < static_cast<int>(w.size()); ++i) {
        for (int c = w[i]; c <= cap; ++c) {
            dp[c] = std::max(dp[c], dp[c - w[i]] + val[i]);
        }
    }
    return dp[cap];
}
```

代码讲解：

- 完全背包允许同一个物品使用多次，因此容量正序枚举。
- 当计算 `dp[c]` 时，`dp[c - w[i]]` 可以是本轮刚更新过的状态，表示继续选择第 i 个物品。
- 和 01 背包的主要差异不是状态定义，而是枚举顺序。

容量正序允许同一物品重复转移。

### 6.4 LCS 最长公共子序列

`dp[i][j]` 表示 `a[0..i)` 与 `b[0..j)` 的 LCS 长度。

```cpp
int lcs(const std::string& a, const std::string& b) {
    int n = a.size(), m = b.size();
    std::vector<std::vector<int>> dp(n + 1, std::vector<int>(m + 1));
    for (int i = 1; i <= n; ++i) {
        for (int j = 1; j <= m; ++j) {
            if (a[i - 1] == b[j - 1]) dp[i][j] = dp[i - 1][j - 1] + 1;
            else dp[i][j] = std::max(dp[i - 1][j], dp[i][j - 1]);
        }
    }
    return dp[n][m];
}
```

代码讲解：

- 如果 `a[i-1] == b[j-1]`，这个字符可以接在两个前缀的 LCS 后面，所以来自 `dp[i-1][j-1] + 1`。
- 如果不相等，只能丢掉 `a` 的最后一个字符或 `b` 的最后一个字符，取二者较大值。
- 使用 `n+1`、`m+1` 的表可以把空串边界统一放在第 0 行/列，避免大量 if 判断。

### 6.5 LIS 最长递增子序列

O(n²) DP：

```cpp
int lis_dp(const std::vector<int>& a) {
    int n = a.size(), ans = 0;
    std::vector<int> dp(n, 1);
    for (int i = 0; i < n; ++i) {
        for (int j = 0; j < i; ++j) {
            if (a[j] < a[i]) dp[i] = std::max(dp[i], dp[j] + 1);
        }
        ans = std::max(ans, dp[i]);
    }
    return ans;
}
```

代码讲解：

- `dp[i]` 表示以 `a[i]` 结尾的 LIS 长度，而不是前 i 个元素整体的 LIS 长度。
- 枚举所有 `j < i`，只要 `a[j] < a[i]`，就可以把 `a[i]` 接到以 `a[j]` 结尾的序列后面。
- 该写法直观但 O(n²)，适合先讲清楚状态定义。

O(n log n) 贪心 + 二分：

```cpp
int lis_binary(const std::vector<int>& a) {
    std::vector<int> tail;
    for (int x : a) {
        auto it = std::lower_bound(tail.begin(), tail.end(), x);
        if (it == tail.end()) tail.push_back(x);
        else *it = x;
    }
    return static_cast<int>(tail.size());
}
```

代码讲解：

- `tail` 保持递增，`lower_bound` 找到第一个 `>= x` 的位置。
- 如果 `x` 比所有末尾都大，就能扩展出更长递增子序列。
- 否则用更小的 `x` 替换某个长度的末尾值，表示“同样长度下保留更小结尾”，给后续元素更多接上的机会。
- 若题目要求“非递减子序列”，应把 `lower_bound` 改为 `upper_bound`。

`tail[len - 1]` 表示长度为 `len` 的递增子序列末尾最小可能值；它不一定是真实答案序列，但长度正确。
