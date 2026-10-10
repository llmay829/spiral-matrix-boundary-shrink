# Spiral Matrix: The Boundary-Shrinking Walkthrough That Fixes the Two Loops Everyone Gets Wrong

You've written the four loops. You ran it on the 3×3 example, it printed `[1,2,3,6,9,8,7,4,5]`, you felt good. Then you submitted it and it failed on `[[1,2,3,4]]`. Or on a single-column matrix. Or it returned duplicate values in the last ring.

Spiral matrix is one of those problems that looks like a simulation exercise and is actually a bookkeeping exercise. The traversal logic is obvious to everyone. What trips people up is knowing exactly when the remaining unvisited region has become empty, and stopping before a loop re-reads a row or column it already consumed. Get that right and the problem is ten lines of code.

Here's the full walkthrough: the boundary approach, a trace you can follow by hand, the two guards that fix the edge cases, the trade-offs against other approaches, and how to drill it so it sticks.

## What the problem is actually asking

Given an `m × n` matrix, return all its elements in clockwise spiral order: left to right along the top row, down the right column, right to left along the bottom row, up the left column, then inward one layer and repeat.

Two canonical examples:


Input:            Output:
[[1, 2, 3],       [1,2,3,6,9,8,7,4,5]
 [4, 5, 6],
 [7, 8, 9]]



Input:                  Output:
[[1, 2, 3, 4],          [1,2,3,4,8,12,11,10,9,5,6,7]
 [5, 6, 7, 8],
 [9,10,11,12]]


It's LeetCode 54, rated Medium. The rating is a little generous if you already think in boundaries, and a little optimistic if you're trying to track a position and a direction index by hand.

Note the second example: the matrix is 3×4, not square. Almost every bug in this problem shows up in the rectangular case, not the square one.

## The mental model: four shrinking walls, not a moving cursor

Instead of tracking *where you are*, track *what's left*.

Four integers describe the unvisited region at all times:

- `top`: index of the topmost row not yet read
- `bottom`: index of the bottommost row not yet read
- `left`: index of the leftmost column not yet read
- `right`: index of the rightmost column not yet read

The invariant is that everything outside `top <= r <= bottom` and `left <= c <= right` has already been added to the result. Every sweep consumes exactly one side of that box and then pulls the corresponding wall inward.

One full cycle looks like this:

1. Walk `matrix[top][left..right]` left to right. Then `top += 1`.
2. Walk `matrix[top..bottom][right]` top to bottom. Then `right -= 1`.
3. Walk `matrix[bottom][right..left]` right to left. Then `bottom -= 1`.
4. Walk `matrix[bottom..top][left]` bottom to top. Then `left += 1`.

Repeat while `top <= bottom` and `left <= right`.

A quick trace on the 3×4 matrix:

| Cycle | Sweep | Cells read | Wall moved |
| --- | --- | --- | --- |
| 1 | East | 1, 2, 3, 4 | top: 0 → 1 |
| 1 | South | 8, 12 | right: 3 → 2 |
| 1 | West | 11, 10, 9 | bottom: 2 → 1 |
| 1 | North | 5 | left: 0 → 1 |
| 2 | East | 6, 7 | top: 1 → 2 |
| 2 | South | (nothing, `top > bottom`) | — |

Result: `[1,2,3,4,8,12,11,10,9,5,6,7]`. That matches the expected output, and the walls tell you why: after cycle 1 the unvisited box is rows 1–1, columns 1–2. Two cells left, one short sweep, done.

## The part most explanations skip: the two guards

Here's the code, with the guards that matter:

python
def spiralOrder(matrix):
    if not matrix or not matrix[0]:
        return []

    top, bottom = 0, len(matrix) - 1
    left, right = 0, len(matrix[0]) - 1
    res = []

    while top <= bottom and left <= right:
        # East: top row, left -> right
        for c in range(left, right + 1):
            res.append(matrix[top][c])
        top += 1

        # South: right column, top -> bottom
        for r in range(top, bottom + 1):
            res.append(matrix[r][right])
        right -= 1

        # West: only if a row still exists
        if top <= bottom:
            for c in range(right, left - 1, -1):
                res.append(matrix[bottom][c])
            bottom -= 1

        # North: only if a column still exists
        if left <= right:
            for r in range(bottom, top - 1, -1):
                res.append(matrix[r][left])
            left += 1

    return res


Those two `if` checks before the West and North sweeps are not defensive padding. They are load-bearing, and removing them breaks two specific inputs.

**Single row.** Take `[[1,2,3,4]]`. `top = bottom = 0`. The East sweep reads all four values and sets `top = 1`. Without a guard, the West sweep would now run across `matrix[bottom]` (still row 0) from right to left and append `4,3,2,1` a second time. With the guard `top <= bottom` (1 <= 0 is false), the sweep is skipped entirely.

**Single column.** Take `[[1],[2],[3]]`. The South sweep reads 1, 2, 3 and sets `right = -1`. Without a guard, the North sweep would walk back up the same column and duplicate everything. The guard `left <= right` (0 <= -1 is false) stops it.

This is why the "four clean loops inside a while" version works only on square matrices of odd size, which happens to be the test case everyone tries first.

## A second way to think about it: direction vectors

The other common solution keeps a cursor `(r, c)` and a direction index, moving one cell at a time and turning clockwise when the next cell is out of bounds or already visited. AlgoMonster's editorial for this problem uses exactly that shape, with the direction array `{0, 1, 0, -1, 0}` where adjacent pairs give the row and column deltas for right, down, left, up.

python
directions = [(0, 1), (1, 0), (0, -1), (-1, 0)]
visited = [[False] * cols for _ in range(rows)]


It's easier to reason about mentally and harder to get wrong at the boundaries, because you only ever ask one question: "can I step there?" The cost is a `visited` grid, which pushes extra space from O(1) to O(m·n).

Trade-off summary:

| Approach | Extra space (beyond output) | Risk |
| --- | --- | --- |
| Four boundaries | O(1) | Missing guards on rectangular / single-line inputs |
| Direction vectors + visited | O(m·n) | Low; but you allocate per call |
| Recursive rotate trick | O(min(m,n)) stack | Cute, rarely what an interviewer wants |

That recursive trick is worth knowing exists, if only for the grin:

python
def spiralOrder(self, matrix):
    if not matrix:
        return []
    return list(matrix.pop(0)) + self.spiralOrder(list(zip(*matrix))[::-1])


It pops the first row, rotates the remainder counter-clockwise, and recurses. Fine as a curiosity, awkward to explain under pressure, and it mutates the input.

## Complexity, stated plainly

Time is **O(m·n)** for all of these. Every cell is appended exactly once, and the boundary variables do constant work per sweep.

Extra space is **O(1)** for the boundary version, ignoring the output array, which itself needs O(m·n) since it holds every element. If someone asks "is this O(1) space?", the precise answer is: O(1) auxiliary, O(m·n) for the result you were asked to return.

## Mistakes that cost you the submission

The list of ways this goes wrong is short and consistent:

- Skipping the empty-matrix check, or writing `matrix[0]` before checking `len(matrix)`. Order matters: test `not matrix` first, then `not matrix[0]`, or you'll index into nothing on an empty input.
- Forgetting the West and North guards. Covered above.
- Off-by-one in the reverse ranges. `range(right, left - 1, -1)` includes `left`. `range(right, left, -1)` doesn't, and it silently drops a cell.
- Incrementing `top` in the wrong order relative to the South sweep. `top += 1` must happen before the South loop starts from the new `top`, otherwise you re-read the corner you just appended.
- Assuming square input because the example is 3×3.
- Mutating the input matrix. The rotate trick does this, which is usually disqualifying unless the interviewer says otherwise.

The corner cells are where all the double-counting happens. Every sweep should leave the shared corner to exactly one loop.

## What interviewers actually follow up with

Getting the traversal right is the entry fee. Expect one of these next:

- **Spiral Matrix II (LeetCode 59)** — same traversal, but you're *writing* `1..n²` into an `n × n` matrix instead of reading out. The boundary code transfers almost unchanged; the difference is you assign instead of append.
- **Spiral Matrix III / IV** — moving from a fixed grid to a walk that can leave the matrix, or generating a lexicographically smallest spiral. Same ring intuition, different termination conditions.
- **Print Matrix Clockwise (JianZhi Offer 29)** — an identical problem with different branding, common in Chinese-language interview loops.
- **Rotate Image (LeetCode 48)** — layer-by-layer again, but with in-place swaps. Candidates who understand rings usually see the connection immediately.
- **"What if the matrix is a stream?"** — you can't hold it in memory, so the boundary approach needs rethinking. This is where the discussion moves from coding to design.

The reason this problem shows up so often isn't the spiral. It's that it exposes whether you can maintain an invariant and state it out loud. The candidates who do well say something like "`top` is the first unvisited row" rather than "so then I increment `i`."

## How to actually drill it

Solving spiral matrix once teaches you spiral matrix. Solving it as a member of a pattern is what transfers.

The pattern here is matrix traversal by shrinking boundary. It's the same pattern behind rotate image, set matrix zeroes done in place, and a pile of grid problems where "which cells are still in play" is the real question. If you can name the pattern on a new problem in ten seconds, you've saved yourself the twenty minutes of re-deriving it under interview pressure.

That's the gap structured pattern courses are built for. AlgoMonster organizes its curriculum around named patterns with templates and decision flowcharts rather than a wall of problems, and its write-up of this exact question sits in the public problem library:

👉 [Read AlgoMonster's Spiral Matrix editorial and solution](https://algo.monster/liteproblems/54?fpr=li69)

If you want to go a level up from the single problem, the matrix traversal pattern itself is covered in the paid course material, along with Speedrun mode for timed review before an onsite.

## What AlgoMonster costs right now

AlgoMonster runs two paid tiers plus a free tier, and it has no month-to-month option. Prices on the site move with promotions, and published third-party comparisons quote figures all over the map, so the checkout page is the only number worth trusting.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of lessons and problems, enough to judge whether the text-and-diagram teaching style works for you | $0 | No card needed | [Start with the free tier](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company questions updated monthly; 500+ lessons across 6 courses; 1,000+ illustrations; 3,000+ editorials; member discount on live bootcamps | $5.84/mo, billed annually at $69.99/year | 12 months, cancel or upgrade anytime | [Check the current yearly price](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates, one payment, no renewals | $89 | Billed once | [See the lifetime plan](https://algo.monster/subscribe?fpr=li69) |

Two things worth saying about those numbers.

First, they're lower than the "$99/year, $189 lifetime" figures that most review sites list, which tells you either that the site is running a deep promotion right now or that those reviews are stale. Either way, verify before you buy.

Second, compare against what you'd otherwise spend. If your prep window is four to eight weeks and you already know which patterns you're weak on, a year of access is the sensible buy. Lifetime only pays off if you expect to interview again in two or three years, which is common enough among engineers who switch companies rather than stay put.

If your weak spot is specifically "I know the concept but I fumble the code under time pressure," that's the case for the structured version rather than free editorial hunting.

👉 [Compare all AlgoMonster plans on the official page](https://algo.monster?fpr=li69)

## FAQ

**What is a spiral matrix?**
Either the input or the output of a traversal problem: reading an `m × n` grid in clockwise order, moving inward ring by ring. The name also covers the inverse version, where you fill a grid in spiral order.

**What's the time complexity?**
O(m·n), and you can't do better without skipping cells, since the output contains every element.

**Is O(1) extra space possible?**
Yes, and the boundary method is how. You need four integers and no visited grid.

**Why does my solution duplicate the last row or column?**
Because the West or North sweep ran after the region it traverses became empty. Add the `top <= bottom` and `left <= right` guards.

**Is it really a Medium?**
The traversal is easy. The edge-case handling is what makes it Medium, and it's also what makes it a genuinely useful warm-up before harder grid problems.

**Do I need to memorize the code?**
No. Memorize the invariant instead: `top`, `bottom`, `left`, `right` bound the unvisited region, and each sweep consumes one side of that box. If you can restate that, you can rebuild the code in two minutes on a whiteboard.

The version of this problem worth being able to write is the version that survives `[[1,2,3,4]]` and `[[1],[2],[3]]` on the first try. Everything else is the same four sweeps with the walls moved in.
