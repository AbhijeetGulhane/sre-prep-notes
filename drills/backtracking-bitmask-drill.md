# Drill: Backtracking + Bitmask-DP — Write COLD Under Time

Escalating-reps drill. Goal: write both reference solutions **cold, correct, runnable** under time pressure. This closes a practically-tested surface. **Not** NP-hardness theory — this is about recognizing the pivot and executing the pattern.

## Recognition trigger (say this out loud when you see it)

> "Optimal is intractable at general scale → pivot to **backtracking with pruning** (find/count valid configs), **bitmask/subset DP** (small n, ≤ ~20), **pseudo-poly DP** (value bounded), or **greedy-approx** (only if approximation acceptable)."

The interviewer signal: constraints are *small* (n ≤ 12-20), or the problem says "all ways" / "is there a valid arrangement." That smallness is the license to do exponential work cleanly.

---

## Target 1 — N-Queens (backtracking with pruning)

**Recognition:** place items with constraints that conflict across rows/cols/diagonals; "all valid arrangements." Small board.

**Reference (Python):**
```python
def solveNQueens(n):
    res = []
    cols = set()
    diag = set()      # r - c  (constant along ╲)
    anti = set()      # r + c  (constant along ╱)
    board = [["."] * n for _ in range(n)]

    def backtrack(r):
        if r == n:
            res.append(["".join(row) for row in board])
            return
        for c in range(n):
            if c in cols or (r - c) in diag or (r + c) in anti:
                continue                     # PRUNE
            cols.add(c); diag.add(r - c); anti.add(r + c)
            board[r][c] = "Q"
            backtrack(r + 1)
            cols.remove(c); diag.remove(r - c); anti.remove(r + c)
            board[r][c] = "."                # UNDO

    backtrack(0)
    return res
```

**Complexity — with the WHY welded on:**
- Time: **O(n!)** — row by row, first row has n choices, next ≤ n-1 survive the column/diagonal prune, and so on. The pruning is what makes it n! instead of nⁿ; without the diagonal/col sets you'd explore invalid branches.
- Space: **O(n)** for the three sets + recursion depth n. Board is O(n²) but that's output-shaped.

**Cold-write checklist:** three sets (cols, r−c, r+c) · prune-before-place · place → recurse → **undo all three + board** · base case r==n.

---

## Target 2 — Held-Karp TSP (bitmask / subset DP, small n)

**Recognition:** "visit every node once, minimize cost," n small (≤ ~15-18). Brute force is n! ; bitmask DP is n²·2ⁿ — the pivot that makes it tractable.

**Reference (Python):**
```python
import math

def held_karp(dist):
    n = len(dist)
    # dp[mask][i] = min cost to start at 0, visit exactly the set `mask`, end at i
    dp = [[math.inf] * n for _ in range(1 << n)]
    dp[1][0] = 0                              # only node 0 visited, at node 0

    for mask in range(1 << n):
        for i in range(n):
            if dp[mask][i] == math.inf:
                continue
            if not (mask & (1 << i)):
                continue
            for j in range(n):
                if mask & (1 << j):
                    continue                  # j already visited
                nxt = mask | (1 << j)
                cost = dp[mask][i] + dist[i][j]
                if cost < dp[nxt][j]:
                    dp[nxt][j] = cost

    full = (1 << n) - 1
    return min(dp[full][i] + dist[i][0] for i in range(1, n))  # close the tour
```

**Complexity — with the WHY:**
- Time: **O(n²·2ⁿ)** — 2ⁿ subsets (the mask), × n end-nodes per subset, × n transitions. The 2ⁿ is the subset enumeration; that's why it only works for small n (2^18 ≈ 260k is fine, 2^25 is not).
- Space: **O(n·2ⁿ)** for the dp table — this is the binding constraint in practice; you run out of memory before time.

**Cold-write checklist:** dp[mask][end] · seed dp[1][0]=0 · iterate masks ascending · skip if i not in mask / j in mask · transition mask|(1<<j) · close tour back to 0.

---

## Rep prescription

- **2×/week**, cold, timed. Sit down, no reference, write both from the checklist.
- **Pass bar:** N-Queens runnable in < 8 min; Held-Karp runnable in < 12 min, both with correct undo/seed and no typos (apply RITUAL 1 — identifier pass).
- If either fails the bar, that one gets an extra mid-week rep.
- Log misses: the usual failure is forgetting to **undo all state** (N-Queens) or **seed dp[1][0]** / mis-ordering the mask loop (Held-Karp).
