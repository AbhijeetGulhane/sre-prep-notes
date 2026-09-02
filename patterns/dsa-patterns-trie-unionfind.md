# DS&A Patterns: Trie + Union-Find — Quick Reference

Two patterns that are easy to miss recognizing under pressure. Each: recognition trigger, cold-writable template, one canonical problem, complexity.

---

## Trie (prefix tree)

**Recognition trigger:** "prefix," "starts with," "search words with wildcards," "autocomplete," dictionary of strings with shared prefixes, or word-search-on-grid with many words.

**Template (Python):**
```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word):
        node = self.root
        for ch in word:
            if ch not in node.children:
                node.children[ch] = TrieNode()
            node = node.children[ch]
        node.is_end = True

    def search(self, word):
        node = self.root
        for ch in word:
            if ch not in node.children:
                return False
            node = node.children[ch]
        return node.is_end

    def starts_with(self, prefix):
        node = self.root
        for ch in prefix:
            if ch not in node.children:
                return False
            node = node.children[ch]
        return True
```

**Canonical problem:** #208 Implement Trie. Step up: #211 Add and Search Word (`.` wildcard → DFS at the wildcard), #212 Word Search II (Trie + grid DFS).

**Complexity — with WHY:** insert/search **O(L)** where L = word length — you walk one node per character, no scanning of other words. Space **O(total chars)** worst case, but shared prefixes collapse, which is the whole point vs a hash set of strings.

---

## Union-Find (Disjoint Set Union)

**Recognition trigger:** "connected components," "is A connected to B," "number of groups/islands via unions," "redundant edge / cycle in undirected graph," "accounts merge," dynamic connectivity. If you're tempted to BFS/DFS *repeatedly* for connectivity queries → Union-Find.

**Template (Python) — with path compression + union by rank:**
```python
class DSU:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n
        self.count = n                # number of components

    def find(self, x):
        while self.parent[x] != x:
            self.parent[x] = self.parent[self.parent[x]]  # path compression
            x = self.parent[x]
        return x

    def union(self, a, b):
        ra, rb = self.find(a), self.find(b)
        if ra == rb:
            return False              # already connected (→ cycle / redundant)
        if self.rank[ra] < self.rank[rb]:
            ra, rb = rb, ra
        self.parent[rb] = ra
        if self.rank[ra] == self.rank[rb]:
            self.rank[ra] += 1
        self.count -= 1
        return True
```

**Canonical problem:** #323 Number of Connected Components. Step up: #684 Redundant Connection (union returns False → that's the cycle edge), #547 Number of Provinces, #261 Graph Valid Tree (n-1 unions all succeed → tree).

**Complexity — with WHY:** near **O(α(n))** ≈ O(1) amortized per op with path compression + union by rank — α is the inverse Ackermann function, effectively constant. Without both optimizations it degrades toward O(n) per find on a degenerate chain, which is the trap.

**Cold-write checklist:** parent=range(n) · find with path compression (grandparent hop) · union by rank · `union` returns False on same-root (that False *is* the cycle/redundant signal) · track `count` if you need component count.

---

## Why these two matter for SysEng

Union-Find in particular shows up in infrastructure-flavored problems: network connectivity, host grouping, dependency clustering. Being fluent signals you think in the right data structure, not just BFS-everything.
