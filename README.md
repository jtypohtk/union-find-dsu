# union find python: Build a Working DSU Class With Path Compression, Dodge the Recursion Crash, and Know Which Problems Actually Need It

If you searched "union find python," you probably already know what a disjoint set is. What you want is a class you can paste into a solution file, a clear answer on whether `find` is O(1) or O(log n), and some sense of which LeetCode problems are secretly union find problems in a costume.

That's what this is. Code first, then the details people usually get wrong, then where the pattern shows up — including the connectivity questions where DFS is slower or simply won't work.

## The short version of what you're building

Union find (also called disjoint set union, DSU, or merge-find set) keeps a collection of non-overlapping sets and supports three things:

- `find(x)` — return the representative (root) of the set containing `x`
- `union(a, b)` — merge the two sets containing `a` and `b`
- `connected(a, b)` — a convenience wrapper around `find(a) == find(b)`

Everything else — path compression, union by rank, union by size — exists to keep the trees shallow so those three calls stay cheap.

## The implementation most interviews will accept

python
class DSU:
    __slots__ = ("parent", "size", "components")

    def __init__(self, n: int):
        self.parent = list(range(n))   # every node starts as its own root
        self.size = [1] * n
        self.components = n

    def find(self, x: int) -> int:
        # walk to the root
        root = x
        while self.parent[root] != root:
            root = self.parent[root]
        # path compression: re-point everything on the path at the root
        while self.parent[x] != root:
            self.parent[x], x = root, self.parent[x]
        return root

    def union(self, a: int, b: int) -> bool:
        ra, rb = self.find(a), self.find(b)
        if ra == rb:
            return False               # already in the same set
        if self.size[ra] < self.size[rb]:
            ra, rb = rb, ra            # attach the smaller tree under the bigger one
        self.parent[rb] = ra
        self.size[ra] += self.size[rb]
        self.components -= 1
        return True

    def connected(self, a: int, b: int) -> bool:
        return self.find(a) == self.find(b)


Two loops in `find` looks redundant the first time you read it, and it isn't. The first loop finds the root; the second walks the same path again and flattens it, so the next `find` on those nodes is one hop. If you only ever call `find` once per node, compression never pays off — but you almost never call it once.

A shorter variant, path halving, is common in competitive programming because it's five lines and gets most of the benefit:

python
def find(self, x: int) -> int:
    while self.parent[x] != x:
        self.parent[x] = self.parent[self.parent[x]]
        x = self.parent[x]
    return x


If you want the recursive one-liner — `return x if self.parent[x] == x else self.find(self.parent[x])` — read the next section before you paste it.

## The bugs that actually bite people

**1. Recursion depth.** A recursive `find` on a long chain hits Python's default recursion limit of 1000 and raises `RecursionError`. This is exactly the input that appears in test suites designed to punish uncompressed finds. Use the iterative version and the problem disappears.

**2. Skipping path compression because "union by rank is enough."** Without compression, `find` degrades toward O(n) per call on adversarial inputs. With it, the amortized cost is O(α(n)), the inverse Ackermann function — under 5 for any input size you will ever type into a terminal.

**3. Misunderstanding rank.** Rank is an *upper bound* on tree height, not the height itself. It only increases when you merge two trees of equal rank. If you try to maintain the exact height after path compression, you'll spend more time bookkeeping than you save.

**4. Reading `size[x]` on a non-root.** Sizes are only meaningful at roots. `self.size[x]` for a child node is a stale leftover from before it got attached, and it will silently give you a wrong answer when you use it to count members of a component.

**5. Forgetting the grid index math.** On a 2D grid, you have to flatten coordinates yourself: `index = r * cols + c`. Getting `r * rows + c` wrong is a genuinely popular way to spend an hour debugging.

**6. Counting components by scanning at the end.** Scanning for `parent[i] == i` is O(n) and fine, but if you're maintaining connectivity across queries, keep a counter and decrement it on every successful union. That's what the `components` field above is for.

## Which variant are you actually writing?

| Variant | `find` | `union` | Notes |
| --- | --- | --- | --- |
| Quick find (array of set ids) | O(1) | O(n) | Every union rewrites the whole array. Teaching material. |
| Quick union (parent pointers only) | O(n) worst case | O(n) worst case | Simple, but a bad sequence of merges creates a linked list in disguise. |
| Union by size or rank | O(log n) | O(log n) | Height is capped at log n. |
| Union by size/rank + path compression | O(α(n)) amortized | O(α(n)) amortized | Effectively constant. Use this. |

The practical takeaway: O(α(n)) is not just "fast in theory." For n = 10⁶ you can treat both operations as constant time and move on.

## Union find for string and non-integer nodes

Interviews love problems where the elements are emails, accounts, or words rather than `0..n-1`. Two options.

Cast to integers with a dictionary:

python
idx = {}
def get_id(key):
    if key not in idx:
        idx[key] = len(idx)
    return idx[key]


Or make the structure lazy, registering unknown nodes on first contact:

python
class DSU:
    def __init__(self):
        self.parent = {}
        self.size = {}
        self.components = 0

    def find(self, x):
        if x not in self.parent:
            self.parent[x] = x
            self.size[x] = 1
            self.components += 1
        root = x
        while self.parent[root] != root:
            root = self.parent[root]
        while self.parent[x] != root:
            self.parent[x], x = root, self.parent[x]
        return root


The lazy version is the one that saves you time in "Accounts Merge"-style problems, where you don't know the element set until you've read the input.

## Where union find earns its keep

**Connected components.** Number of Provinces (547), Number of Connected Components in an Undirected Graph (323). Feed each edge to `union` and read `components` at the end.

**Cycle detection in undirected graphs.** If `find(a) == find(b)` and you're about to union them, that edge closes a cycle. Redundant Connection (684) and Graph Valid Tree (261) are both this check with extra phrasing.

**Kruskal's minimum spanning tree.** Sort edges by weight, walk them in order, and keep an edge only when its endpoints aren't already connected. That's the entire algorithm:

python
def kruskal(n, edges):
    dsu = DSU(n)
    total = 0
    used = 0
    for w, u, v in sorted(edges):
        if dsu.union(u, v):
            total += w
            used += 1
            if used == n - 1:
                break
    return total if used == n - 1 else None


Min Cost to Connect All Points (1584) is this pattern with Manhattan distances generated on the fly.

**Equivalence constraints.** Satisfiability of Equality Equations (990) is pure union find: process every `a == b`, then verify every `a != b` contradicts nothing.

**Grids and dynamic connectivity.** Number of Islands II (305) — the version where land is added cell by cell — is awkward with BFS and trivial with union find. That "add edges over time" shape is the real signal.

**Almost-union-find problems.** Longest Consecutive Sequence (128) can be solved with a set walk in O(n); union find works too, but it's the heavier tool. Recognize when you're over-engineering.

AlgoMonster's own routing flowchart makes the same call explicit: when a graph problem is a connectivity question, the flowchart sends you to Disjoint Set Union rather than a traversal. The platform keeps this under an Advanced Data Structures / Disjoint Set Union module with a fundamentals lesson, an introductory problem, and companion practice like Number of Connected Components and Number of Unique Categories (a LiteProblem solved with a parent array and path compression).

## Union find vs. DFS/BFS — a decision you can make in ten seconds

Pick union find when:

- edges are added incrementally and you need connectivity at each step
- the question is "are these two things in the same group?" asked many times
- you're sorting edges and want a cycle check while you go (Kruskal)

Pick DFS/BFS when:

- you need to traverse, order, or label a component fully
- edges are directed or weighted and you care about paths, not membership
- you'd have to rebuild the structure from scratch after each query anyway

A rule of thumb that holds up: "are these two connected?" points at union find, and "what can I reach from here?" points at graph traversal.

## If you'd rather not write it yourself

Python has two ready-made options.

**networkx** ships a `UnionFind` class (`networkx/utils/union_find.py`) originally based on Josiah Carlson's 2004 ActiveState recipe, with later changes by David Eppstein. The API is different from what you'd write by hand: `X[item]` returns the name of the set containing `item`, `X.union(a, b)` merges them, and `to_sets()` yields the current partition. It works on hashable objects, so strings are fine.

**The `disjoint-set` package on PyPI** requires Python 3.10+ and gives you `DisjointSet`, with `find`, `union`, `connected`, and `from_iterable`. Small, readable, no dependencies.

Neither is what an interviewer wants to see, though. The hand-written version is roughly twenty lines, and it's one of the few data structures where writing it from memory is genuinely faster than reasoning about a library's API under pressure.

## Practicing the pattern instead of re-deriving it

The gap most people hit isn't understanding union find. It's recognizing, in the first two minutes of a problem, that a connectivity question is hiding under the story about couples at a party or accounts sharing an email. That recognition is a pattern-matching skill, and it's built by seeing the pattern mapped to problems explicitly rather than by grinding randomly.

That's the pitch behind AlgoMonster: 48 named patterns, each with a template and mapped practice problems, plus decision flowcharts for the "which pattern is this?" moment. Its DSU material sits where you'd expect — under Advanced Data Structures — and the union find lessons use Python alongside other languages.

Worth knowing before you buy: the pricing is promotion-driven and the numbers move around. Here's what the subscription page showed on this check, alongside the plan structure:

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems — enough to judge whether the text-and-diagram format works for you | $0 | — | [ Start with the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI Assistant; unlimited Speedrun; company questions updated monthly; 500+ lessons across 6 courses; 1,000+ illustrations; member discount on live bootcamps | $99/year (listed at $198, shown as $8.25/mo billed annually) | 12 months, cancel or upgrade anytime | [ Check the current yearly price](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan plus lifetime updates on new content | $189 one-time (from $378) | One payment, yours forever | [ See the lifetime plan](https://algo.monster?fpr=li69) |

Two things to keep in mind. First, the price you see at checkout is the only one that counts — during another sales window the same page displayed $69.99/year and an $89 lifetime, so the discount depth is not fixed. Second, AlgoMonster includes bonus Foundation and "Build an LLM" courses with a subscription, while some partner-developed courses (Quant Interview Prep, for example) are sold separately.

If you want to stack another 10% on top of the site's sale, partner pages have listed a code — one programmer-focused discounts page, updated in September 2026, names `SCHELLEY` as verified working on top of the 50% sale. Codes like this come and go, so treat it as worth trying at checkout rather than guaranteed.

There's no monthly option and no separate team or enterprise plan; it's annual or lifetime, aimed at individual candidates.

## The 30-second version

Write the parent array, compress paths on every find, union by size, and keep a component counter. Use the iterative `find` so you never meet `RecursionError`. Reach for it when edges arrive over time or when you're sorting them anyway — Kruskal, cycle checks, province counting, incremental grids. For everything else, DFS is usually shorter and just as fast.

Get the recognition part down and the twenty lines of code will stop being the hard part of the problem.
