# DSA / LeetCode Roadmap — Pattern-First

The goal is not to grind a list of problems. It's to internalize ~15 recognizable
**patterns** so that when you read a new problem, you recognize "this is a sliding
window" or "this is DP on intervals" within a minute. Once a pattern clicks, dozens
of LeetCode problems become the same problem wearing a different costume.

Each pattern below has:
- **What it is** — the core idea.
- **Recognize it when** — signals in the problem statement that point to this pattern.
- **Depends on** — patterns you should already have before tackling this one.
- **Complexity intuition** — what you're trading.
- **Anchor problems** — 3-5 problems that cover the pattern's variants, not 30.

Study a pattern until you can explain out loud *why* it works, then do the anchor
problems without looking at the solution for at least 20 minutes before you peek.

---

## Phase 0 — Prerequisites (don't skip, even if it feels basic)

Before patterns, you need fluent command of:
- **Big-O reasoning** — count operations, not lines of code. Be able to look at nested
  loops, recursion trees, or a hash lookup and state complexity instantly.
- **Arrays, strings, hash maps, sets** as first-class tools — know their operation
  costs (insert/lookup/delete) cold.
- **Recursion** — base case, recursive case, and being able to draw the call tree.
  Almost everything from Phase 3 onward (trees, backtracking, DP) is recursion with
  extra bookkeeping.

If any of these feel shaky, spend a few days here first. Everything else compounds
on top of this.

**Exercises (do these on paper or out loud, no code yet):**
- Given three snippets — a single loop, two nested loops, and a loop with a
  hash-set lookup inside — state each one's time complexity and justify it in
  one sentence.
- Write the recursive definition (base case + recursive case) for: factorial,
  sum of a list, and reversing a string. Draw the call tree for `factorial(5)`
  by hand.
- Trace naive recursive Fibonacci(5) by hand and count how many times `fib(2)`
  gets recomputed. This redundancy is exactly what Phase 8 (DP) exists to
  eliminate.
- From memory, describe how a hash set backed by an array with chaining
  handles collisions, and state what happens to lookup cost if every key
  collides into the same bucket.

---

## Phase 1 — Linear scanning patterns (the foundation layer)

These four patterns are all about **not re-scanning the array**. They're the
first rung because almost every later pattern (sliding window inside DP, two
pointers inside graph problems, etc.) reuses this instinct.

### 1.1 Arrays & Hashing
- **What it is:** Trade space for time. Use a hash map/set to turn an O(n) lookup
  into O(1), collapsing an O(n²) brute force into O(n).
- **Recognize it when:** "find a pair/triplet that sums to X," "find duplicates,"
  "group items by some derived key," "have I seen this before."
- **Depends on:** Phase 0 only.
- **Complexity intuition:** O(n) time, O(n) space — you're paying memory to avoid
  a second loop.
- **Anchor problems:** Two Sum, Group Anagrams, Top K Frequent Elements,
  Longest Consecutive Sequence, Contains Duplicate.
- **Exercises:**
  - Explain out loud why Two Sum is O(n) with a hash map but O(n log n) if you
    sort and use two pointers instead (both work — know the tradeoff).
  - Solve without looking at notes: Valid Anagram, Subarray Sum Equals K,
    Encode and Decode Strings.
  - Twist: solve Two Sum assuming the array is *already sorted* — does the
    hash map approach still make sense, or is there a better tool now? (This
    previews Two Pointers.)

### 1.2 Two Pointers
- **What it is:** Two indices moving through a structure (usually sorted, or from
  both ends) instead of nested loops.
- **Recognize it when:** Array is sorted (or can be sorted), "pair that satisfies
  a condition," palindrome checks, merging two sequences.
- **Depends on:** Arrays & Hashing (to know when hashing *isn't* the better tool).
- **Complexity intuition:** O(n) or O(n log n) if sorting is needed first — beats
  the O(n²) nested-loop version.
- **Anchor problems:** Valid Palindrome, Two Sum II (sorted), 3Sum, Container With
  Most Water, Trapping Rain Water.
- **Exercises:**
  - Explain why Two Pointers requires sorted (or sortable) input while Arrays
    & Hashing doesn't.
  - Solve: Sort Colors (Dutch National Flag), Remove Duplicates from Sorted
    Array, Boats to Save People.
  - Twist: 3Sum finds triplets summing to 0 — extend the same technique to
    4Sum and state how the complexity changes and why.

### 1.3 Sliding Window
- **What it is:** Two Pointers' cousin — maintain a *contiguous* window and grow/shrink
  it instead of recomputing from scratch each time.
- **Recognize it when:** "longest/shortest/max/min substring or subarray that
  satisfies a condition," contiguous + condition = window.
- **Depends on:** Two Pointers directly (window = two pointers moving in the same
  direction with state tracked in between).
- **Complexity intuition:** O(n) — each pointer moves forward at most n times total.
- **Anchor problems:** Best Time to Buy/Sell Stock, Longest Substring Without
  Repeating Characters, Longest Repeating Character Replacement, Minimum Window
  Substring, Sliding Window Maximum.
- **Exercises:**
  - State the invariant your window maintains for Longest Substring Without
    Repeating Characters, and explain why the window only ever grows/shrinks
    instead of resetting to size 0 and rescanning.
  - Solve: Permutation in String, Longest Substring with At Most K Distinct
    Characters, Fruit Into Baskets.
  - Twist: Sliding Window Maximum needs a monotonic deque, not just window
    bounds — explain why a plain window-pointer approach isn't enough there,
    and connect it to the Stack pattern below.

### 1.4 Stack (incl. Monotonic Stack)
- **What it is:** LIFO for tracking "most recent unresolved thing." Monotonic stack
  keeps elements in increasing/decreasing order to answer "next greater/smaller"
  in O(n) instead of O(n²).
- **Recognize it when:** Matching pairs (parentheses), "next greater/smaller
  element," needing to undo the most recent decision, expression evaluation.
- **Depends on:** Arrays & Hashing (conceptually independent, but usually taught
  after since it's the last "simple" linear-structure pattern).
- **Complexity intuition:** O(n) — each element pushed/popped at most once.
- **Anchor problems:** Valid Parentheses, Min Stack, Evaluate RPN, Daily
  Temperatures, Largest Rectangle in Histogram.
- **Exercises:**
  - Explain why a monotonic stack processes each element in O(1) amortized
    time even though it "looks" like nested loops (each element is pushed and
    popped at most once).
  - Solve: Generate Parentheses, Asteroid Collision, Online Stock Span.
  - Twist: Largest Rectangle in Histogram uses a monotonic increasing stack —
    trace through `[2,1,5,6,2,3]` by hand and note exactly when each bar gets
    popped and why.

---

## Phase 2 — Search over sorted / decision space

### 2.1 Binary Search
- **What it is:** Halve the search space each step. Works not just on sorted
  arrays but on any monotonic "yes/no" decision space (binary search on the answer).
- **Recognize it when:** Sorted array, or you can phrase the problem as "find the
  smallest/largest X such that condition(X) is true" and condition is monotonic.
- **Depends on:** Phase 0 only, but conceptually pairs with Two Pointers (both
  exploit ordering).
- **Complexity intuition:** O(log n) instead of O(n).
- **Anchor problems:** Binary Search, Search in Rotated Sorted Array, Find
  Minimum in Rotated Sorted Array, Koko Eating Bananas (binary search on answer),
  Median of Two Sorted Arrays.
- **Exercises:**
  - Write the binary-search-on-answer template from scratch (define a
    monotonic `condition(x)`, then binary search over the range where it
    flips) and apply it conceptually to Koko Eating Bananas before coding.
  - Solve: Find Peak Element, Search a 2D Matrix, Time Based Key-Value Store.
  - Twist: Median of Two Sorted Arrays binary-searches over a *partition
    index*, not a value — explain what invariant the partition must satisfy
    for the answer to be correct.

---

## Phase 3 — Recursive structures: Linked Lists & Trees

Linked lists are the bridge from arrays to pointer-based recursive thinking.
Trees are where recursion becomes the primary tool, and everything after this
phase (backtracking, graphs, DP) leans on the recursion muscle you build here.

### 3.1 Linked List — Fast & Slow Pointers, In-place Reversal
- **What it is:** Two pointers moving at different speeds to detect cycles/find
  midpoints; reversing pointers in place instead of using extra space.
- **Recognize it when:** "detect a cycle," "find the middle," "reverse a list/
  part of a list," anything where you can't random-access like an array.
- **Depends on:** Two Pointers (same idea, pointer-based structure instead of
  index-based).
- **Anchor problems:** Reverse Linked List, Linked List Cycle, Reorder List,
  Remove Nth Node From End, Merge K Sorted Lists.
- **Exercises:**
  - Explain why fast/slow pointers detect a cycle — why must they eventually
    meet if one exists? — and derive why the fast pointer conventionally moves
    at 2x speed rather than 3x.
  - Solve: Palindrome Linked List, Add Two Numbers, Copy List with Random
    Pointer.
  - Twist: reverse a linked list both iteratively and recursively, then
    compare the space complexity of each approach.

### 3.2 Trees — DFS & BFS
- **What it is:** DFS (pre/in/post-order, usually recursive) explores depth-first;
  BFS (queue-based) explores level-by-level. Most tree problems are "what do I
  compute on the way down vs. on the way up (return value)."
- **Recognize it when:** Anything with a tree/binary tree, "level order," "path
  from root to leaf," "is this tree balanced/valid/symmetric."
- **Depends on:** Recursion (Phase 0) + Stack/Queue (Phase 1) for the iterative
  versions.
- **Complexity intuition:** O(n) time (visit each node once), O(h) space for DFS
  recursion stack where h = height.
- **Anchor problems:** Invert Binary Tree, Maximum Depth of Binary Tree, Same
  Tree, Binary Tree Level Order Traversal, Validate BST, Lowest Common Ancestor
  of a BST, Binary Tree Maximum Path Sum, Serialize/Deserialize Binary Tree.
- **Exercises:**
  - For each of Validate BST, Maximum Depth, and Diameter of Binary Tree,
    state whether the answer is computed "on the way down" (passed as an
    argument) or "on the way up" (returned from children).
  - Solve: Diameter of Binary Tree, Subtree of Another Tree, Kth Smallest
    Element in a BST, Construct Binary Tree from Preorder and Inorder
    Traversal.
  - Twist: convert your recursive DFS solution for Maximum Depth into an
    iterative version using an explicit stack, then do the same for BFS level
    order using a queue.

### 3.3 Tries (Prefix Trees)
- **What it is:** A tree specialized for strings — each node is a character,
  paths from root spell prefixes.
- **Recognize it when:** "prefix," "autocomplete," "word search in a dictionary,"
  repeated prefix lookups.
- **Depends on:** Trees (it *is* a tree, just with a fixed branching alphabet).
- **Anchor problems:** Implement Trie, Design Add and Search Words, Word Search II.
- **Exercises:**
  - Explain the space/time tradeoff a trie makes vs. storing all words in a
    hash set for prefix queries.
  - Solve: Longest Word in Dictionary, Replace Words.
  - Twist: add wildcard support (`.`) to your trie search (as in Design Add
    and Search Words) — explain why this turns lookup into a DFS over the
    trie instead of an O(depth) walk.

---

## Phase 4 — Priority & ordering

### 4.1 Heap / Priority Queue
- **What it is:** Always access the min/max in O(log n) instead of O(n) scan or
  O(n log n) full sort.
- **Recognize it when:** "kth largest/smallest," "top k," "merge k sorted
  things," scheduling/greedy-by-priority problems.
- **Depends on:** Trees (a heap *is* a tree, just array-backed and shape-constrained).
- **Anchor problems:** Kth Largest Element in an Array, Top K Frequent Elements
  (revisit with a heap this time), Find Median from Data Stream, Merge K Sorted
  Lists (revisit), Task Scheduler.
- **Exercises:**
  - Explain why a heap gives O(log n) insert/extract-min but O(n) to find an
    arbitrary element — and why that tradeoff is fine for "top k" problems.
  - Solve: K Closest Points to Origin, Last Stone Weight, Reorganize String.
  - Twist: Find Median from Data Stream uses *two* heaps — explain what
    invariant is kept between the max-heap and min-heap halves.

---

## Phase 5 — Exhaustive search with pruning

### 5.1 Backtracking
- **What it is:** DFS over a decision tree where you build a partial solution,
  recurse, and *undo* (backtrack) when a branch fails or is fully explored.
- **Recognize it when:** "all possible," "all combinations/permutations/subsets,"
  constraint satisfaction (N-Queens, Sudoku), "generate all valid ___."
- **Depends on:** Trees/DFS directly — backtracking is DFS on an implicit tree
  of choices instead of an explicit data structure.
- **Complexity intuition:** Usually exponential (2^n or n!) — the pattern is about
  pruning that exponential space early, not avoiding it entirely.
- **Anchor problems:** Subsets, Combination Sum, Permutations, Word Search,
  Palindrome Partitioning, N-Queens.
- **Exercises:**
  - For Subsets vs. Combination Sum vs. Permutations, draw (on paper) the
    shape of each decision tree and state where pruning happens in each.
  - Solve: Combination Sum II, Letter Combinations of a Phone Number,
    Palindrome Partitioning (revisit with an "is this substring a palindrome"
    cache).
  - Twist: for N-Queens, explain what state you need to track to check "is
    this column/diagonal safe" in O(1) instead of re-scanning the board on
    every placement.

---

## Phase 6 — Graphs

Graphs generalize trees (a tree is just a connected acyclic graph), so this phase
directly builds on Phase 3's DFS/BFS and Phase 5's backtracking instincts.

### 6.1 Graphs — DFS, BFS, Topological Sort, Union-Find
- **What it is:**
  - DFS/BFS on graphs: same as trees, but you need a `visited` set since cycles exist.
  - Topological sort: ordering nodes so dependencies come first (only on DAGs).
  - Union-Find (Disjoint Set Union): efficiently track connected components /
    merge groups.
- **Recognize it when:** Grid problems ("number of islands"), "course
  prerequisites" / dependency ordering, "connected components," "can you get
  from A to B," cycle detection.
- **Depends on:** Trees' DFS/BFS (Phase 3) + Backtracking's "track state and
  undo" instinct for path problems.
- **Anchor problems:** Number of Islands, Clone Graph, Course Schedule (topo
  sort), Pacific Atlantic Water Flow, Graph Valid Tree (union-find), Number of
  Connected Components.
- **Exercises:**
  - Explain the difference between Kahn's algorithm (BFS-based topo sort) and
    DFS-based topo sort — when would you prefer one over the other (e.g.,
    detecting *which* nodes are in a cycle)?
  - Solve: Rotting Oranges (multi-source BFS), Course Schedule II (return the
    actual order), Redundant Connection (union-find).
  - Twist: implement Union-Find with both path compression and union by rank,
    then explain why the combination gives near-O(1) amortized operations.

### 6.2 Advanced Graphs (do this after everything else feels solid)
- **What it is:** Shortest path algorithms (Dijkstra), minimum spanning tree
  (Prim's/Kruskal's), more complex weighted-graph reasoning.
- **Recognize it when:** "shortest path with weights," "minimum cost to connect
  all points," "cheapest flights within K stops."
- **Depends on:** Graphs (11) + Heap (9) — Dijkstra is literally BFS + a
  priority queue.
- **Anchor problems:** Network Delay Time (Dijkstra), Cheapest Flights Within K
  Stops, Min Cost to Connect All Points (MST).
- **Exercises:**
  - Explain why Dijkstra fails on graphs with negative edge weights, and what
    Bellman-Ford does differently to handle them.
  - Solve: Path with Maximum Probability (Dijkstra variant), Swim in Rising
    Water.
  - Twist: implement both Prim's and Kruskal's for Min Cost to Connect All
    Points, and compare why Kruskal's needs Union-Find while Prim's needs a
    heap.

---

## Phase 7 — Intervals & Greedy

### 7.1 Intervals
- **What it is:** Sort by start (or end) time, then sweep left to right merging/
  comparing overlaps.
- **Recognize it when:** Anything with `[start, end]` pairs — meetings, ranges,
  scheduling.
- **Depends on:** Sorting (Phase 0) + the greedy instinct below.
- **Anchor problems:** Insert Interval, Merge Intervals, Non-overlapping
  Intervals, Meeting Rooms II.
- **Exercises:**
  - Explain why sorting by *start* time works for Merge Intervals but sorting
    by *end* time is what you need for Non-overlapping Intervals — what's the
    difference in what you're optimizing for?
  - Solve: Interval List Intersections, My Calendar I.
  - Twist: Meeting Rooms II can be solved with a heap or with a sorted
    "events" sweep (start = +1, end = -1) — implement both and compare.

### 7.2 Greedy
- **What it is:** Make the locally optimal choice at each step and prove (or
  trust the pattern) that it leads to a globally optimal solution. The hard part
  isn't coding it — it's recognizing *when* greedy actually works vs. when you
  need DP.
- **Recognize it when:** "maximum/minimum number of ___," problems where sorting
  first + one pass gives the answer, and you can articulate why a local choice
  never hurts the global outcome.
- **Depends on:** Intervals is the cleanest greedy sub-case, so do it first.
- **Anchor problems:** Jump Game, Jump Game II, Gas Station, Hand of Straights,
  Merge Triplets to Form Target Triplet.
- **Exercises:**
  - For Jump Game, explain in one sentence why the greedy "track the farthest
    reachable index" approach is provably optimal (what would break if it
    weren't?).
  - Solve: Gas Station (explain why a global negative-sum check works),
    Partition Labels, Valid Parenthesis String.
  - Twist: take one greedy problem and argue why DP would also work but be
    worse (higher complexity) — that's the skill greedy problems are actually
    testing.

---

## Phase 8 — Dynamic Programming (the capstone)

DP is backtracking/recursion (Phase 5) **plus memoization** — you're still
exploring a decision tree, but you cache overlapping subproblems instead of
recomputing them. This is why DP should come *after* backtracking, not before:
if recursion over choices doesn't feel natural yet, DP will feel like magic
instead of mechanics.

### 8.1 1-D Dynamic Programming
- **What it is:** State depends on a single changing parameter (index i).
  Build up `dp[i]` from `dp[i-1]`, `dp[i-2]`, etc.
- **Recognize it when:** "number of ways to ___," "min/max cost to reach ___,"
  Fibonacci-shaped recurrences, decisions along a single sequence.
- **Depends on:** Backtracking (write the brute-force recursion first, then add
  a memo table — this progression matters more than memorizing formulas).
- **Anchor problems:** Climbing Stairs, House Robber, Coin Change, Longest
  Increasing Subsequence, Word Break, Decode Ways.
- **Exercises:**
  - For House Robber, write the brute-force recursion first (no memo),
    identify the overlapping subproblems, add memoization, then convert to
    bottom-up with O(1) space.
  - Solve: Maximum Subarray (Kadane's), Longest Palindromic Subsequence,
    Delete and Earn.
  - Twist: for Word Break, explain why naive recursion is exponential here
    specifically (what's being recomputed?) and how memoizing on the string
    index fixes it.

### 8.2 2-D Dynamic Programming
- **What it is:** State depends on two changing parameters — usually two indices
  into two sequences, or one index + one capacity/budget (knapsack).
- **Recognize it when:** Comparing two strings/sequences ("longest common
  subsequence/substring," edit distance), grid path problems, knapsack-style
  "items with capacity constraint."
- **Depends on:** 1-D DP fully internalized first — 2-D DP is the same
  brute-force-then-memoize progression with one more dimension.
- **Anchor problems:** Unique Paths, Longest Common Subsequence, Edit Distance,
  0/1 Knapsack (via Partition Equal Subset Sum), Longest Palindromic
  Substring, Interleaving String.
- **Exercises:**
  - For Longest Common Subsequence, draw the 2-D dp table by hand for two
    short strings (e.g. `"abcde"` and `"ace"`) and fill it in manually before
    coding.
  - Solve: Coin Change II, Target Sum, Distinct Subsequences.
  - Twist: in 0/1 Knapsack (via Partition Equal Subset Sum), explain why
    iterating the capacity dimension *backwards* matters when you compress to
    a 1-D array, and what bug you'd introduce by iterating forwards instead.

---

## Phase 9 — Odds and ends (light, high ROI, do in parallel with anything)

### 9.1 Bit Manipulation
- **What it is:** XOR/AND/OR/shift tricks — small standalone pattern, doesn't
  depend on anything else.
- **Recognize it when:** "without using extra memory," "find the single/missing
  number," problems explicitly about bits.
- **Anchor problems:** Single Number, Number of 1 Bits, Counting Bits, Missing
  Number, Reverse Bits.
- **Exercises:**
  - Explain why `x & (x - 1)` clears the lowest set bit, and use that fact to
    solve Number of 1 Bits and Counting Bits without a lookup table.
  - Solve: Sum of Two Integers (bit-level addition, no `+`), Single Number II
    (element appears 3x, others once).
  - Twist: Single Number works via XOR when everything else appears twice —
    explain why XOR fails once "everything else" appears three times, and
    what different trick Single Number II needs.

### 9.2 Math / Geometry
- Lowest priority for interview ROI unless a specific company's loop leans on
  it. A handful of problems (e.g., Rotate Image, Spiral Matrix) is enough
  unless you have signal that a target company emphasizes it.
- **Exercises:**
  - Solve: Rotate Image, Spiral Matrix, Set Matrix Zeroes.
  - Twist: for Rotate Image in-place, explain the transpose-then-reverse
    trick as two separate geometric operations rather than memorizing index
    formulas.

---

## How the phases connect (the actual dependency graph)

```
Phase 0 (Big-O, recursion, arrays)
   │
   ▼
Phase 1: Arrays&Hashing → Two Pointers → Sliding Window
                                 │
                                 └──► Stack / Monotonic Stack
   │
   ▼
Phase 2: Binary Search  (parallel track, reuses "ordering" instinct)
   │
   ▼
Phase 3: Linked List (fast/slow, reversal) → Trees (DFS/BFS) → Tries
   │
   ▼
Phase 4: Heap / Priority Queue  (heap = constrained tree)
   │
   ▼
Phase 5: Backtracking  (DFS over decision trees)
   │
   ▼
Phase 6: Graphs (DFS/BFS/topo/union-find) → Advanced Graphs (Dijkstra, MST)
   │                                              (needs Heap too)
   ▼
Phase 7: Intervals → Greedy
   │
   ▼
Phase 8: 1-D DP → 2-D DP   (DP = backtracking + memoization)

Phase 9: Bit Manipulation, Math/Geometry — standalone, slot in anytime
```

## Suggested cadence

- **Weeks 1-2:** Phase 0-1 (arrays, hashing, two pointers, sliding window, stack).
- **Week 3:** Phase 2-3 (binary search, linked list, trees, tries).
- **Week 4:** Phase 4-5 (heaps, backtracking).
- **Weeks 5-6:** Phase 6 (graphs — this phase is dense, don't rush it).
- **Week 7:** Phase 7 (intervals, greedy).
- **Weeks 8-9:** Phase 8 (DP — the capstone; budget the most time here).
- Ongoing, in parallel: Phase 9, and a weekly "mixed review" day doing 3-4
  problems pulled randomly from earlier patterns so recognition speed doesn't decay.

Once all patterns feel automatic, switch from "study by pattern" to timed mixed
practice (e.g., a curated 150-problem list, randomized, no labels) — that's what
actually simulates the interview, where nobody tells you which pattern applies.
