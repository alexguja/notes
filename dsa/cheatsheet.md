# Data Structures & Algorithms Notes

Working notes on the coding-interview patterns: the core idea behind each, the signal
that tells you to reach for it, and the template that falls out once you've spotted it.
Distilled from the Hello Interview coding curriculum.

One idea runs through most of these: **the brute force is almost always an O(n²) rescan,
and the pattern is whatever lets you reuse the work you already did at the previous
position** rather than recomputing it. Two pointers, sliding window, and prefix sums are
all the same trick wearing different clothes.

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

A **monotonic stack** (kept sorted) answers "next greater / next smaller element" in O(n).
Keep it decreasing for next-greater: when the incoming element beats the top, that
incoming element *is* the answer for everything you pop. Because the stack is monotonic,
you know the popped elements can't be anyone else's answer either, which is why the total
work stays linear.

## Linked list: pointer hygiene

Two tricks cover most problems:

- **Dummy head node** so building or deleting from the front isn't a special case.
- **Fast and slow pointers** (fast moves two, slow moves one) to find the middle or detect
  a cycle in one pass, O(1) space.

In-place reversal is the three-pointer dance (`prev`, `curr`, `next`), flipping one link
per step.

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

## BFS: go wide, get shortest paths

A queue processes nodes in order of distance, so in an **unweighted** graph the first time
you reach a node is along a shortest path. Mark visited *on enqueue*, not dequeue, or
nodes get added twice. Snapshot the queue length at the top of each loop to process one
level at a time (level-order output, or counting steps).

## Topological sort: ordering a DAG

For dependency ordering, Kahn's algorithm: compute each node's **indegree**, enqueue every
indegree-0 node, and peel, decrementing neighbours and enqueueing any that hit zero. If you
can't emit all `n` nodes, the leftovers form a **cycle**, which is your free cycle check.

# Part V — Exhaustive and optimisation patterns

## Backtracking: choose, explore, unchoose

DFS over a solution-space tree to enumerate combinations, permutations, subsets, or paths.
The discipline is: make a choice, recurse, then **undo the choice** before trying the
next; **prune** branches that can't possibly succeed. Append a *copy* of the running path
to results, since the path keeps mutating underneath you.

## Dynamic programming: cache overlapping subproblems

Applies when the problem has **optimal substructure** (the optimum is built from optimal
sub-answers) and **overlapping subproblems** (naive recursion recomputes the same things).
Define the state, find the recurrence, pin the base cases. Top-down memoisation is usually
easiest to derive; bottom-up tabulation is easiest to space-optimise (if the recurrence
only needs the last two values, keep two variables, not an array).

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
