+++
title = "Proving bounds for the Randomized MaxCut Approximation algorithm in Lean4"
[taxonomies]
  tags = ["lean", "algorithms"]
+++

For a given graph `G = (V, E)`, a cut `C` is a set of edges such that there is a partition `V = (A, B)` where all edges `e ∈ C` have one vertex in `A` and the other in `B`.

MaxCut is a very famous combinatorial optimization problem wherein we want to find the largest such cut. This is useful in scheduling, partitioning, financial portfolio optimization, and more! While the problem itself is NP-Complete, there exist a host of approximation algorithms that allow us to do "pretty good" in practice. To see a careful treatment of deterministic MaxCut algorithms, check out these [two](https://pages.cs.wisc.edu/~jyc/02-810notes/lecture19.pdf) [lecture](https://courses.cs.cornell.edu/cs4820/2019sp/handouts/max-cut.pdf) notes.

Some quick nomenclature: an α-approximation for a (maximization/minimization) problem is when we:
1. (max) Do as good as 1/α * OPT
2. (min) Do as good as α * OPT

For our purposes, lets say we don't even care about doing good right now, just that we care about doing good in expectation. That is, while there exists a 1/2-approximation for MaxCut, we want to create an algorithm such that the expected size of our output cut satisfies the 1/2 approximation ratio.

### Our simple algorithm
For every vertex `v ∈ V = (A, B)`, choose with 50% probability whether it is in `A` or `B`. That's it, that creates the cut.

Good job! Take a well-deserved break now.

---

### Proving it informally
Now that you're back from break, let's get to proving this with words!

For any edge `e = (u, v) ∈ E`, we have `Pr[(u, v) ∈ C] = 1/2`. For each edge `e`, create a random variable `Xₑ` such that `Xₑ = 1 if e ∈ C else 0`. Notice now that the size of the cut, `|C| = ∑ Xₑ`.

The overall proof is then `E[|C|] = ∑ E[Xₑ] = ∑ Pr[e ∈ C] = |E|/2 ≥ |C*|/2`, for some optimal cut `C*`. Just throw linearity of expectation and use our previous facts - this, plus the fact that the size of the largest possible cut is bounded by the number of edges (that is, `|E| ≥ |C*|`), and we get the result we want. `|E| = |C*|` only when the graph `G` is bipartite; this follows basically by definition.

## Proving it with Lean
Having encountered this simple and elegant algorithm + proof in my algorithms course, I wanted to see if I could formalize it in Lean. I've been getting more interested in formal verification, so this seemed like a fun challenge, especially w.r.t the randomization aspect of it! For the uninitiated, Lean is a programming language and proof assistant similar to Rocq. There has been a big movement in recent times to formalize mathematics and CS concepts within the Lean MathLib and CSLib, and I'm eager to see where it goes :)

### Structures
The first step is always representing our key structures. Lean already has a `SimpleGraph`, so all we need to do is design a structure to store our `Cut`:
```haskell
-- Given some graph G = (V, E)
-- a cut is defined as a set of edges C ⊆ E such that for each edge c = (u, v) ∈ C,
-- we have that u ∈ V₁ and v ∈ V₂, for V₁ ⊕ V₂ (disjoint union) = V
structure Cut (V : Type*) (G : SimpleGraph V) [DecidableEq V] [Fintype V] where
  A : Finset V
  B : Finset V
  -- below are proof obligations
  -- propositions are types, proofs are values of those types!
  partition : A ∪ B = Finset.univ
  disjoint  : Disjoint A B
```

We also create some helper functions, namely for:
1. Creating cuts from an assignment, a mapping from vertices to booleans (true in A, false in B)
2. `size` of the cut
3. Whether a given edge is in the cut at all.
```haskell
-- determine whether an edge is in the cut
def edgeInCut {V : Type*} [DecidableEq V] [Fintype V] {G : SimpleGraph V}
  (C : Cut V G) (e : Sym2 V) : Bool :=
  e.lift ⟨fun u v => (u ∈ C.A ∧ v ∈ C.B) ∨ (u ∈ C.B ∧ v ∈ C.A), by simp [or_comm, and_comm]⟩

def size {V : Type*} [DecidableEq V] [Fintype V] {G : SimpleGraph V}
  [DecidableRel G.Adj] (C : Cut V G) : ℕ := 
  (G.edgeFinset.filter (fun e => C.edgeInCut e)).card

-- given some characteristic function, use char func to determine A, B
def ofAssignment {V : Type*} [DecidableEq V] [Fintype V] {G : SimpleGraph V}
  (f : V -> Bool) : Cut V G where
    A := Finset.univ.filter (fun e => f e)
    B := Finset.univ.filter (fun v => !f v)
    partition := by grind
    disjoint  := by
      simp only [Finset.disjoint_iff_inter_eq_empty, Bool.not_eq_eq_eq_not, Bool.not_true]
      ext v
      simp
```

**Some notable things**:
The backbone of our `Cut` is the `Finset`, which is a type representing a finite set of elements of some type α. Since graphs are finite structures, we get a lot of free convenient properties from leveraging `Finset` here. We also use typeclass declarations for `Fintype` and `DecidableEq`. `Fintype α` lets our function know that α is finite, and `DecidableEq` knows that propositional equality is decidable for all elements of type α.

Generally, we also use the idea of an assignment function quite a bit, i.e. some mapping from `V -> Bool` that defines our cut.

---

From here on out, I'll skimp on some of the proof details and go over more of the overarching approaches for some of my lemmas, since that seems like a more economical use of time and article space.

First, we finalize our randomized cut instances and the edge containment indicator function, which basically represents our random variable `Χₑ` from the proof.
```haskell
def randomizedMaxCut {V : Type*} [DecidableEq V] [Fintype V]
    {G : SimpleGraph V} : (V → Bool) → Cut V G :=
  fun assignment => Cut.ofAssignment (G := G) assignment

-- use edgeInCut to create a characteristic function returning in {0, 1}
def edgeIndicator {V : Type*} [DecidableEq V] [Fintype V] {G : SimpleGraph V}
  (C : Cut V G) (e : Sym2 V) : ℕ :=
  if Cut.edgeInCut C e then 1 else 0
```

### Proofs
Finally, in the order I proved them, here are the lemmas and theorems!

1. `cut_size_eq_sum_indicators`: `|C| = ∑ χₑ ∀ e ∈ E`
```haskell
lemma cut_size_eq_sum_indicators {V : Type*} [DecidableEq V] [Fintype V] (G : SimpleGraph V)
  [DecidableRel G.Adj] (assignment : V -> Bool) :
    (Cut.ofAssignment (G := G) assignment).size =
    ∑ e ∈ G.edgeFinset, edgeIndicator (Cut.ofAssignment (G := G) assignment) e := by ...
```
Here, we first unfold our relevant definitions. Then, we convert to using summation for both sides, and finally we simplify and show that our sets are equal; if the sets are equal, their cardinalities must be equal as well!

2. `count_diff_assignments`: half of the 2^N possible assignments have `f u ≠ f v`
```haskell
lemma count_diff_assignments {V : Type*} [Fintype V] [DecidableEq V]
  (u v : V) (huv : u ≠ v) : 2 * (Finset.univ.filter (fun f : V -> Bool => f u ≠ f v)).card =
    Fintype.card (V -> Bool) := by ...
```
This proof is a behemoth of sorts, unexpectedly. The key thread here is to codify the fact that `f u ≠ f v` is half the space by creating a bijection and showing it is an involution (its own inverse). Then, if we know that the sets `f u ≠ f v` and `f u = f v` have the same cardinality, and that they equal `2^|V|`, we get our result.

3. `prob_edge_in_cut`: `Pr[e ∈ C] = 1/2`
```haskell
lemma prob_edge_in_cut {V : Type*} [DecidableEq V] [Fintype V] (G : SimpleGraph V)
  (e : Sym2 V) (h : ¬e.IsDiag) :
  (∑ assignment : V -> Bool, (edgeIndicator (G := G) (randomizedMaxCut assignment) e : ℝ)) / 
  (Fintype.card (V -> Bool)) = 1/2 := by ...
```
For this proof we use induction on the edges to extract the vertices, then claim that the sum of the indicators when `u ≠ v` is equal to the cardinality of the `Finset` when `f u ≠ f v` (the count of "good" assignments, when not in same vertex set of partition). This, plus our result from the `count_diff_assignments` (counting lemma) plus some tiny polishing steps gives us our result.

4. `expected_cut_size`: `E[|C|] = |E|/2` (the important one!)
```haskell
theorem expected_cut_size {V : Type*} [DecidableEq V] [Fintype V] (G : SimpleGraph V)
  [DecidableRel G.Adj] :
    (∑ assignment : V -> Bool,
    ((randomizedMaxCut (G := G) assignment).size : ℝ)) / (Fintype.card (V -> Bool)) =
      (G.edgeFinset.card : ℝ) / 2 := by ...
```
This proof begins with our original expectation statement and chains together our previous proofs to give us our major result. First, we leverage `cut_size_eq_sum_indicators` to get our size in terms of the indicators. Next, we simulate our linearity of expectation via `Finset.sum_comm` and `Finset.sum_div` to move things around. Finally, we use the `calc` tactic to sequentially show equivalences with the `prob_edge_in_cut` lemma and summation results to give us our answer!

---
#### A brief interlude: finding *the* Max Cut
In order to wrap this up and prove things with respect to the actual maximum cut, we need to find a way to represent this. Today I learned about the `Finset.univ.sup`, meaning "supremum". A supremum is the smallest quantity greater than or equal to each element of a given (sub)set. In effect, we iterate over all `Finset`s, create `Cut`s based off of each possible assignment, and pick the best/max one!
```haskell
def maxCutValue {V : Type*} [DecidableEq V] [Fintype V] (G : SimpleGraph V)
  [DecidableRel G.Adj] : ℕ :=
  Finset.univ.sup (fun f : V -> Bool => (Cut.ofAssignment (G := G) f).size)
```
Back to the proofs.

---

5. `maxCut_le_edges`: Maximum cut is at most `|E|`. This proof is short enough to include in its entirety:
```haskell
lemma maxCut_le_edges {V : Type*} [Fintype V] [DecidableEq V]
    (G : SimpleGraph V) [DecidableRel G.Adj] :
    maxCutValue G ≤ G.edgeFinset.card := by
  -- notice that finset S's max size is |E|
  -- a Cut can contain at most |E| edges anyway
  unfold maxCutValue
  -- sup cut sizes <= num edges
  apply Finset.sup_le
  intro S _
  -- unfold to get finset and card
  unfold Cut.ofAssignment Cut.size
  -- filtered subset of edges, so cardinality must be less
  exact Finset.card_filter_le _ _
```

6. `rand_approx_guarantee`: `E[|C|] >= |maxCut|/2`. This proof is also short, because it stands tall on the backs of giants (the others).
```haskell
theorem rand_approx_guarantee {V : Type*} [DecidableEq V] [Fintype V]
  (G : SimpleGraph V) [DecidableRel G.Adj] :
    (∑ assignment : V -> Bool,
    ((randomizedMaxCut (G := G) assignment).size : ℝ)) / (Fintype.card (V -> Bool)) >=
    (maxCutValue G : ℝ) / 2 := by
      rw [expected_cut_size]
      field_simp
      norm_cast
      exact maxCut_le_edges (G := G)
```
We basically just use the `expected_cut_size` and then simplify the division by two on either side, then use our inequality. Note that the expression `(∑ assignment : V -> Bool, ((randomizedMaxCut (G := G) assignment).size : ℝ)) / (Fintype.card (V -> Bool))` accurately represents `E[|C|]`, because it is the cut size divided by the total number of possible assignments.

### Learnings
Lean is hard. Also, I have to learn to better speak the language; it was pretty challenging to learn how some tactics worked even with the reference (quite mathematically heavy, as is to be expected) and AI tools (more hand-wavy, imprecise at times), but this was a great learning experience and I'm happy I took the time to prove this! My original goal was to try proving something similar about randomized Quicksort, but that felt like too hard a challenge for a Lean novice.

The rest of my code (i.e. the full proofs and my silly comments) can be found at this Github repo: [verif-randomized-maxcut](https://github.com/abhamra/verif-randomized-maxcut).

Thank you for reading!
