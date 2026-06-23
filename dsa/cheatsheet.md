# Data Structures & Algorithms Notes

Working notes on the coding-interview patterns: the core idea behind each, the signal
that tells you to reach for it, and the template that falls out once you've spotted it.
Distilled from the Hello Interview coding curriculum.

One idea runs through most of these: **the brute force is almost always an O(n²) rescan,
and the pattern is whatever lets you reuse the work you already did at the previous
position** rather than recomputing it. Two pointers, sliding window, and prefix sums are
all the same trick wearing different clothes.

---

# Part 0 — The structures themselves

Before the patterns, the building blocks. Each is shown from scratch so the behaviour and
the cost of every operation are explicit, not hidden behind a library call.

## Dynamic array

The workhorse. A contiguous block that doubles its capacity when full, which is what makes
append **amortised** O(1): most appends are free, the occasional resize copies everything,
and it averages out to constant.

| Operation | Cost | Why |
|-----------|------|-----|
| index / set | O(1) | direct offset into contiguous memory |
| append | O(1) amortised | doubling spreads the copy cost |
| insert / delete at front | O(n) | everything after shifts |
| search (unsorted) | O(n) | no structure to exploit |

Python's `list` already is this; you rarely reimplement it, but knowing the resize is why
`append` is cheap and `insert(0, x)` is not.

## Stack (LIFO)

Last in, first out. A list used at one end is all you need; both ends of the operation
touch the same side.

```python
class Stack:
    def __init__(self):
        self._items = []

    def push(self, x):        # O(1)
        self._items.append(x)

    def pop(self):            # O(1)
        if not self._items:
            raise IndexError("pop from empty stack")
        return self._items.pop()

    def peek(self):           # O(1)
        return self._items[-1]

    def is_empty(self):
        return not self._items
```

## Queue (FIFO)

First in, first out. A plain list is wrong here: `pop(0)` is O(n) because every remaining
element shifts. Use a **doubly-ended queue** (`collections.deque`), a doubly linked list
under the hood, so both ends are O(1).

```python
from collections import deque

class Queue:
    def __init__(self):
        self._items = deque()

    def enqueue(self, x):     # O(1)
        self._items.append(x)

    def dequeue(self):        # O(1) — the whole point of deque
        if not self._items:
            raise IndexError("dequeue from empty queue")
        return self._items.popleft()

    def is_empty(self):
        return not self._items
```

A `deque` is also your double-ended queue directly: `append`/`appendleft`/`pop`/`popleft`
are all O(1), which is why BFS and sliding-window-maximum reach for it.

## Singly linked list

Nodes, each pointing to the next. O(1) insert/delete *if you already hold the node*, but
O(n) to find anything. The dummy head makes front operations uniform.

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

class LinkedList:
    def __init__(self):
        self.head = None

    def push_front(self, val):            # O(1)
        self.head = ListNode(val, self.head)

    def find(self, target):               # O(n)
        cur = self.head
        while cur:
            if cur.val == target:
                return cur
            cur = cur.next
        return None

    def delete(self, target):             # O(n) to locate
        dummy = ListNode(next=self.head)  # dummy avoids head special-case
        prev, cur = dummy, self.head
        while cur:
            if cur.val == target:
                prev.next = cur.next
                break
            prev, cur = cur, cur.next
        self.head = dummy.next

    def reverse(self):                    # O(n), the three-pointer dance
        prev, cur = None, self.head
        while cur:
            nxt = cur.next
            cur.next = prev
            prev = cur
            cur = nxt
        self.head = prev
```

## Hash map

Average O(1) insert/lookup/delete by hashing the key to a bucket; degrades to O(n) only
under pathological collisions. This is what makes the "seen before?" check in two-sum,
deduping, and frequency counting constant time. In Python it's `dict` / `set`; the thing
worth remembering is *why* it's O(1) and that ordering is not guaranteed by the structure
(insertion order is a CPython implementation detail you shouldn't lean on algorithmically).

## Binary heap

A complete binary tree stored in a flat array: node `i` has children `2i+1` and `2i+2`,
parent `(i-1)//2`. The heap invariant (parent ≤ children, for a min-heap) is restored by
**sift-up** after a push and **sift-down** after a pop. Both are O(log n) because the tree
height is log n.

```python
class MinHeap:
    def __init__(self):
        self.h = []

    def push(self, x):                    # O(log n)
        self.h.append(x)
        self._sift_up(len(self.h) - 1)

    def pop(self):                        # O(log n)
        last = len(self.h) - 1
        self.h[0], self.h[last] = self.h[last], self.h[0]
        top = self.h.pop()
        if self.h:
            self._sift_down(0)
        return top

    def peek(self):                       # O(1)
        return self.h[0]

    def _sift_up(self, i):
        while i > 0:
            parent = (i - 1) // 2
            if self.h[i] >= self.h[parent]:
                break
            self.h[i], self.h[parent] = self.h[parent], self.h[i]
            i = parent

    def _sift_down(self, i):
        n = len(self.h)
        while True:
            smallest, l, r = i, 2 * i + 1, 2 * i + 2
            if l < n and self.h[l] < self.h[smallest]:
                smallest = l
            if r < n and self.h[r] < self.h[smallest]:
                smallest = r
            if smallest == i:
                break
            self.h[i], self.h[smallest] = self.h[smallest], self.h[i]
            i = smallest
```

In practice you use `heapq` (which is exactly this, operating on a plain list), but the
sift logic is worth being able to write.

## Trie (prefix tree)

```python
class TrieNode:
    def __init__(self):
        self.children = {}        # char -> TrieNode
        self.is_word = False

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word):       # O(L)
        node = self.root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
        node.is_word = True

    def search(self, word):       # O(L)
        node = self._walk(word)
        return node is not None and node.is_word

    def starts_with(self, prefix):  # O(L)
        return self._walk(prefix) is not None

    def _walk(self, s):
        node = self.root
        for ch in s:
            if ch not in node.children:
                return None
            node = node.children[ch]
        return node
```

## Graph representations

Two ways to store edges, and the choice drives every traversal's cost.

```python
# Adjacency list — O(V + E) space, iterate a node's neighbours in O(degree).
# The default for sparse graphs (most interview graphs).
def build_adj_list(n, edges, directed=False):
    adj = {i: [] for i in range(n)}
    for u, v in edges:
        adj[u].append(v)
        if not directed:
            adj[v].append(u)
    return adj

# Adjacency matrix — O(V^2) space, edge lookup in O(1).
# Worth it only for dense graphs or when you constantly test "is u-v connected?".
def build_adj_matrix(n, edges, directed=False):
    m = [[0] * n for _ in range(n)]
    for u, v in edges:
        m[u][v] = 1
        if not directed:
            m[v][u] = 1
    return m
```

## Union-Find (disjoint set)

The structure for "are these two things in the same group?" and "merge two groups". With
path compression and union by rank, each operation is effectively O(1) (inverse-Ackermann).
The backbone of connected-components and cycle-detection in undirected graphs (and Kruskal's
MST).

```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n

    def find(self, x):                    # path compression
        while self.parent[x] != x:
            self.parent[x] = self.parent[self.parent[x]]
            x = self.parent[x]
        return x

    def union(self, a, b):                # union by rank
        ra, rb = self.find(a), self.find(b)
        if ra == rb:
            return False                  # already connected (a cycle, if undirected)
        if self.rank[ra] < self.rank[rb]:
            ra, rb = rb, ra
        self.parent[rb] = ra
        if self.rank[ra] == self.rank[rb]:
            self.rank[ra] += 1
        return True
```

---

# Part I — Array and string patterns

## Two pointers: shrink from both ends

When an array is **sorted** and you're hunting for a pair (or triplet) that meets some
sum condition, start one pointer at each end and walk them inward. If the current sum is
too small, the only way to grow it is to move the left pointer right; too big, move the
right pointer left. Each move provably eliminates a whole set of pairs, which is what
collapses the O(n²) pair search into O(n).

```python
def two_sum(nums, target):
    left, right = 0, len(nums) - 1
    while left < right:
        s = nums[left] + nums[right]
        if s == target:
            return True
        if s < target:
            left += 1
        else:
            right -= 1
    return False
```

Reach for it on: pairs/triplets in a sorted array, container-capacity problems, anything
about a relationship between elements at opposite ends.

## Sliding window: a contiguous range with running state

For a **contiguous** subarray or substring under some constraint, keep a window and a
small piece of state describing it. The win is incremental: add the entering element and
subtract the leaving one instead of rescanning the window each step.

- **Fixed length** when you know the size `k`: extend to size `k`, record, then slide.
- **Variable length** when the size is unknown ("longest/shortest ... such that"): expand
  `end` every step, and `while` the window is invalid, contract `start`.

```python
def variable_window(nums):
    state, start, best = {}, 0, 0
    for end in range(len(nums)):
        # add nums[end] to state
        while invalid(state):
            # remove nums[start] from state
            start += 1
        best = max(best, end - start + 1)
    return best
```

Both are O(n): `start` and `end` each cross the array once.

## Prefix sum: precompute, then subtract

If you'll answer many range-sum queries, precompute cumulative sums once. Any range sum
is then the difference of two prefixes in O(1). Build with a leading zero so the first
element isn't a special case: `range(i, j)` sum is `prefix[j + 1] - prefix[i]`.

The hashmap variant answers "how many subarrays sum to `k`": carry a running sum and a map
of `sum -> count seen`; at each index the answer count is the number of times
`running - k` has appeared. Turns O(n²) into O(n).

## Intervals: sort first, then decide what by

Interval problems are a sort followed by a linear sweep. The whole game is *what you sort
by*:

- **Sort by start** to detect overlaps and merge. After sorting, an interval overlaps the
  previous one if it starts before the previous one ends; to merge, extend the last
  interval's end to the max of the two.
- **Sort by end** to greedily keep the most non-overlapping intervals. Always taking the
  one that finishes earliest leaves the most room for the rest.

# Part II — Linear structures

## Stack: last in, first out

The right tool whenever the most recently opened thing must close first: bracket
matching, nested decoding, expression parsing. Push on open, and on a close check the top
matches before popping; a valid sequence ends with an empty stack.

```python
def valid_parentheses(s):
    stack, match = [], {")": "(", "]": "[", "}": "{"}
    for ch in s:
        if ch in match:
            if not stack or stack.pop() != match[ch]:
                return False
        else:
            stack.append(ch)
    return not stack
```

A **monotonic stack** (kept sorted) answers "next greater / next smaller element" in O(n).
Keep it decreasing for next-greater: when the incoming element beats the top, that
incoming element *is* the answer for everything you pop. Because the stack is monotonic,
you know the popped elements can't be anyone else's answer either, which is why the total
work stays linear.

```python
def next_greater(nums):
    res, stack = [-1] * len(nums), []   # stack holds indices, values decreasing
    for i, x in enumerate(nums):
        while stack and nums[stack[-1]] < x:
            res[stack.pop()] = x
        stack.append(i)
    return res
```

## Linked list: pointer hygiene

Two tricks cover most problems:

- **Dummy head node** so building or deleting from the front isn't a special case.
- **Fast and slow pointers** (fast moves two, slow moves one) to find the middle or detect
  a cycle in one pass, O(1) space.

In-place reversal is the three-pointer dance (`prev`, `curr`, `next`), flipping one link
per step (see `LinkedList.reverse` in Part 0). The fast/slow pattern:

```python
def middle(head):                 # slow lands on the middle node
    slow = fast = head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
    return slow

def has_cycle(head):              # fast laps slow iff there's a loop
    slow = fast = head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
        if slow is fast:
            return True
    return False
```

# Part III — Searching and ordering

## Binary search: halve the space

Obvious on a sorted array (`O(log n)`), but the real leverage is **binary search on the
answer**: if some `feasible(x)` flips from false to true at a threshold (minimum speed,
smallest capacity, etc.), binary search that threshold instead of trying every value. The
loop invariant that never bites you: `while left <= right`, and once `left` passes
`right` the space is empty.

## Heap: the top-K workhorse

A heap gives you the min (or max) in O(1) and push/pop in O(log n). For "K largest", keep
a **min-heap of size K**: push each element, and when it grows past K pop the smallest, so
the heap always holds the K best seen, in O(n log K). Python's `heapq` is min-only;
negate values for a max-heap, and push tuples `(priority, item)` to order by priority.

```python
import heapq

def k_largest(nums, k):
    heap = []                     # min-heap of the k best so far
    for x in nums:
        heapq.heappush(heap, x)
        if len(heap) > k:
            heapq.heappop(heap)   # evict the smallest, keeping the top k
    return heap                   # heap[0] is the kth largest
```

# Part IV — Trees and graphs

The two traversals are the foundation; almost everything else is one of them with extra
bookkeeping. Both are O(V + E), and both need a **visited set on graphs** (trees can't
cycle, so they don't).

## DFS: go deep, then backtrack

Recurse down a path and unwind when it dead-ends; the call stack *is* the backtracking.
On trees the power is in **return values**: each call folds its subtrees' results (depth,
sum, validity) and hands them up. On grids, treat each cell as a node with up/down/
left/right neighbours via a directions array, guarding bounds and visited. Classic uses:
connected components (number of islands), and the boundary flip (start from the edges and
mark what's reachable rather than testing each interior cell).

```python
def dfs_graph(adj, start):
    visited = set()
    def go(node):
        if node in visited:
            return
        visited.add(node)
        for nxt in adj[node]:
            go(nxt)
    go(start)
    return visited

DIRS = [(-1, 0), (1, 0), (0, -1), (0, 1)]

def dfs_grid(grid, r, c, visited):
    if not (0 <= r < len(grid) and 0 <= c < len(grid[0])):
        return
    if (r, c) in visited or grid[r][c] == 0:
        return
    visited.add((r, c))
    for dr, dc in DIRS:
        dfs_grid(grid, r + dr, c + dc, visited)
```

## BFS: go wide, get shortest paths

A queue processes nodes in order of distance, so in an **unweighted** graph the first time
you reach a node is along a shortest path. Mark visited *on enqueue*, not dequeue, or
nodes get added twice. Snapshot the queue length at the top of each loop to process one
level at a time (level-order output, or counting steps).

```python
from collections import deque

def bfs_shortest(adj, start, target):
    visited = {start}
    queue = deque([start])
    dist = 0
    while queue:
        for _ in range(len(queue)):      # one whole level per iteration
            node = queue.popleft()
            if node == target:
                return dist
            for nxt in adj[node]:
                if nxt not in visited:
                    visited.add(nxt)     # mark on enqueue, not dequeue
                    queue.append(nxt)
        dist += 1
    return -1
```

## Topological sort: ordering a DAG

For dependency ordering, Kahn's algorithm: compute each node's **indegree**, enqueue every
indegree-0 node, and peel, decrementing neighbours and enqueueing any that hit zero. If you
can't emit all `n` nodes, the leftovers form a **cycle**, which is your free cycle check.

```python
from collections import deque

def topo_sort(n, adj):
    indeg = [0] * n
    for u in adj:
        for v in adj[u]:
            indeg[v] += 1
    queue = deque(u for u in range(n) if indeg[u] == 0)
    order = []
    while queue:
        u = queue.popleft()
        order.append(u)
        for v in adj[u]:
            indeg[v] -= 1
            if indeg[v] == 0:
                queue.append(v)
    return order if len(order) == n else []   # [] => a cycle exists
```

# Part V — Exhaustive and optimisation patterns

## Backtracking: choose, explore, unchoose

DFS over a solution-space tree to enumerate combinations, permutations, subsets, or paths.
The discipline is: make a choice, recurse, then **undo the choice** before trying the
next; **prune** branches that can't possibly succeed. Append a *copy* of the running path
to results, since the path keeps mutating underneath you.

```python
def subsets(nums):
    res, path = [], []
    def backtrack(start):
        res.append(path[:])              # copy: path keeps mutating
        for i in range(start, len(nums)):
            path.append(nums[i])         # choose
            backtrack(i + 1)             # explore
            path.pop()                   # unchoose
    backtrack(0)
    return res
```

## Dynamic programming: cache overlapping subproblems

Applies when the problem has **optimal substructure** (the optimum is built from optimal
sub-answers) and **overlapping subproblems** (naive recursion recomputes the same things).
Define the state, find the recurrence, pin the base cases. Top-down memoisation is usually
easiest to derive; bottom-up tabulation is easiest to space-optimise (if the recurrence
only needs the last two values, keep two variables, not an array).

```python
from functools import cache

@cache                                    # top-down: recursion + memoisation
def climb(n):
    if n <= 1:
        return 1
    return climb(n - 1) + climb(n - 2)

def climb_iter(n):                        # bottom-up, O(1) space
    a, b = 1, 1
    for _ in range(n - 1):
        a, b = b, a + b
    return b
```

## Greedy: commit and never look back

Make the locally optimal choice each step and never revise it. Only correct when a local
optimum is provably global, so the test is to **hunt for a counterexample**; if a greedy
choice can sabotage a later one (longest increasing subsequence is the canonical trap),
fall back to DP's exhaustive search. Sorting is the usual setup, so cost is typically
O(n log n).

# Part VI — Specialised structures

## Trie: prefixes shared as a tree

A character tree where words sharing a prefix share nodes, with an end-of-word flag per
node. Insert/search/prefix-match are all O(L) in the word length; total space is O(C), the
character count across all words. The natural fit for autocomplete and dictionary lookups.
Full implementation in [Part 0](#trie-prefix-tree).

## Matrices: peel the layers

2D traversal is mostly careful index bookkeeping. Spiral order peels the outer ring
(top row left-to-right, right column top-to-bottom, bottom row right-to-left, left column
bottom-to-top) and repeats inward. Rotate-in-place is transpose-then-reverse-each-row;
set-zeroes reuses the first row and column as marker storage to stay O(1) space.

---

# Picking a pattern under pressure

- "Sorted array" → two pointers, or binary search.
- "Contiguous subarray/substring" → sliding window (prefix sum if it's about sums).
- "Longest/shortest/at most K" over a window → variable-length sliding window.
- "Top K / Kth largest / merge K lists" → heap.
- "Next greater/smaller element" → monotonic stack.
- "Shortest path, unweighted" → BFS; "all paths / components / cycles" → DFS.
- "All combinations/permutations/subsets" → backtracking.
- "Dependencies / prerequisites / ordering" → topological sort.
- "Count the ways / min cost / can you reach", with reusable sub-answers → DP.
- "Prefix / autocomplete / dictionary" → trie.
