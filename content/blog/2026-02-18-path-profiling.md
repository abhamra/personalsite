+++
title = "Understanding Whole Program Paths and Path Profiling"
[taxonomies]
  tags = ["llvm", "algorithms"]

[extra]
comment = true
+++
As a part of Georgia Tech's Compiler Design course, the last *sliver* of HW1 was to implement part of the trace and compression algorithm from James Larus' ["Whole Program Paths"](https://www.cs.cmu.edu/afs/cs/academic/class/15745-f09/www/papers/p259-larus.pdf) paper. I wanted to take some time today to go through the paper's ideas, my implementation, and some realizations/jumps I had to make along the way, partly because it's interesting and partly because I had fun thinking about some of the problems and techniques introduced to me in this paper. Before this, I had never really conceptualized what actual profiling infrastructure would look like, and in ~2.5 weeks, I'd only gone and written code to do it myself! I will say, this post will probably not deal with the LLVM-isms as much as my thought process and algorithm design, save a few details.

## What is path profiling?
Before we begin learning how to path profile, we must begin by understanding what it is. I'm trying and failing not to sound like a wise old sensei.

Anyway.

From one of Larus' other papers, ["Efficient Path Profiling"](https://faculty.cc.gatech.edu/~harrold/6340/cs6340_fall2009/Readings/micro96.pdf), we get the following succinct definition:
> A path profile determines how many times each acyclic
> path in a routine executes

Simple enough, and the abstract of the EPP paper goes on to say that this is better than basic block or edge profiling because those are "mere approximations of path frequencies" (sarcasm added). It makes sense - one surely has a better understanding of the dynamic behavior of a program if they have a full trace of the program at their fingertips, rather than frequency data on a per-{block, edge} basis.

## ho(W) do we (P)rofile (P)aths, then? (WPP)

The idea of Whole Program Paths begins with the humble acyclic path itself. The procedure itself works in two phases (from §1.1):
1. Produce a trace of acyclic paths executed by a program (at runtime)
2. Convert the trace into a more compact and usable form by finding its inherent regularity (i.e. repeated code).

The result of this procedure is a lossless, compressed DAG representation of the original CFG (control flow graph), that also lends itself nicely to analysis. All in all, this is a really fantastic deal, and even at the outset is a very cool idea that piqued my interest.

Part 1, the acyclic path trace generation, is largely handled by the EPP paper, with some crucial hurdles to overcome that we'll discuss soon. Part 2 relies on the [SEQUITUR compression algorithm](http://www.sequitur.info/), which is a similarly fascinating algorithm that I didn't have time to implement from scratch. The high level idea of this algorithm is to take the whole program path trace and convert it into a Context Free Grammar (the other CFG), from whence we can create our DAG.

While actually doing this project, our concrete goal was just to return the final compression ratio $$\frac{\text{old # of symbols in original trace}}{\text{new # of symbols in context free grammar}}$$
and so we didn't actually need to implement the DAG-ification procedure, instead using the SEQUITUR website's implementation largely wholesale to get our final results. This, of course, means that Part 1 of our problem is really all we have to focus on, so let's get crackin'.

### Efficient Path Profiling (suboptimal is bad)
In §2 of the WPP paper, "Producing an Acyclic Path Trace", we learn how to produce an acyclic path trace. Except, we really don't, because they say "hey just instrument the code like the EPP paper, with some slight modifications", while making **no attempt to delineate what modifcations those are**.

Ironically my first thought was not realizing that they had never actually specified the procedure for generating the path traces in the paper (or its inevitable modifications), but also that they hadn't defined instrumenting. To *instrument* a program is to insert code at certain points at compile time in a pass (such as an LLVM Analysis), such that at runtime the code runs correctly ***and also*** stores the trace data in a buffer of sorts, either in memory, by printing it out, or in some secret other way.

All that being said, lettuce now dive into the EPP paper, where our joys begin.

---

The Efficient Path Profiling paper works in a few key stages, and works on a function's CFG:
1. Convert your function's CFG into a DAG
2. Assign each acyclic path in the function to a unique natural number via path sums
3. Use a Max-Weight Spanning Tree to identify chords in the graph, then compute chord increments to maintain path sums when *only chords are incremented*
4. Do the actual code instrumentation!

We leave step 4 for later and focus on the first three steps, explicitly as written in the EPP paper.

#### (1) $\text{CFG} \rightarrow \text{DAG} = \text{joy}$
Blissfully unawares of any looming complications, we dive headfirst into the CFG to DAG conversion. The paper is largely clear about this, and for that we all say "thank you".

> For each vertex v that is the target of one or more
backedges, add a dummy edge $ENTRY \rightarrow v$. For
each vertex $w$ that is the source of one (or more)
backedges, add a dummy edge $w \rightarrow EXIT$. If one
of these edges is not in the spanning tree, it will be instrumented, which is efficient as the edge’s increment
can be combined with the code always added to a loop
backedge to record a path.

So yeah, split the backedge $(u, v)$ into $ENTRY \to v$ and $u \to EXIT$, which un-loops our CFG and makes it a DAG. Problem solved!

If this is all we were doing, we could use a `FunctionAnalysisManager` to get a `DominatorTree` from LLVM's `DominatorTreeAnalysis`. From here, finding a backedge is as simple as checking if $B$ dominates $A$ for edge $(A, B)$. HOWEVER. Because of unforseen circumstances (~foreshadowing~), we instead can also do this with a simple DFS traversal, to see if we encounter a block we've already seen on our path at a certain point.

> Also add a backedge $ENTRY \to EXIT$, which is useful for step 3.
Yeah, we'll see this later.

#### (2) Mapping paths to UIDs via path sums
<img src="https://github.com/abhamra/personalsite/blob/master/content/Images/pathsum.png?raw=true" alt="Pathsum demonstration" align="right" width="40%" /> 

This is one of the cooler parts of the paper, and I may take a stab at formally verifying it with Lean soon. The core idea is that given some $NumPaths(r)$ paths in a given DAG rooted at $r$, each unique acyclic path is given a path identifier in $[0, NumPaths(r) - 1]$, and all we have to do is assign each edge to an inicrement value such that, at $EXIT$, we get the correct results! When I first saw this and walked through the algorithm, I was pretty excited, because it's such an elegant idea. Here's an example we can walk through from the paper, Fig 6 from EPP.

This is currently in its "final state", with the path increments applied to each edge - check for yourself that there are indeed 6 paths with numberings in $[0, 5]$.

The algorithm, given in Fig 5, is as follows:
```
foreach vertex v in reverse topological order {
  if v is a leaf vertex {
    NumPaths(v) = 1;
  } else {
    NumPaths(v) = 0;
    for each edge e = v->w {
      Val(e) = NumPaths(v);
      NumPaths(v) = NumPaths(v) + NumPaths(w);
    }
  }
}
```
If we first convert the given CFG to a DAG, then find the reverse topological order (perhaps using Kahn's Algorithm, see my [Advent of Code blog](https://abhamra.com/blog/aoc24day5/) for an example), we get the node list `F, E, D, C, B, A`. Running the algorithm, we do the following:
1. For `F`, since it is a leaf, let $NumPaths(F) = 1$.
2. `E` is not a leaf, so let $Numpaths(E) = 0$ to start; there is only one edge ($E \to F$), thus $Numpaths(E) = 1$ and $Val(E \to F) = 0$.
3. `D` is the first interesting case, with two edges coming out of it. We initialize $NumPaths(D) = 0$ as usual, then notice that depending on which order we iterate over the edges $\{D \to E, ~D \to F\}$, the edges can have different valuations. To stay consistent with the figure, we first handle $D \to F$, setting $Val(D \to F) = 0$, then updating $NumPaths$ and setting $Val(D \to E) = 1$. After this, $NumPaths(D) = 2$ as desired, and we're done.

It may be helpful to manually follow through with the last three basic blocks, but the above should be sufficient to understand the path sum algorithm.

#### (3) Reducing instrumentation via Spanning Trees and Chords
After we have our DAG *with* paths weighted according to the increment along them (such that each acyclic path has a unique path sum) (repetition!), we come to the perhaps obvious realization that if we instrmented (read: added extra profiling code) to each edge in the DAG, we would bloat the crap out of our program. This has been widely regarded as a bad move, so can we do better?

Duh, you say, having read the title of this section, to which I respond that you, stand-in reader, are a figment of my imagination and this is unhelpful for pedagogical purposes. Onward we go.

Our goal here is to go from "every edge is instrumented" to "only a few are instrumented", and one way we can do that is with spanning trees. A spanning tree of a graph $G$ is a subgraph of $G$ that contains every vertex in $G$ and is a tree. It guarantees connectivity, acyclicity, and minimality (using the fewest # of edges to satisfy the previous two conditions, namely $n-1$ edges). All the edges that are in the DAG but not in the spanning tree are called *chords*.

Chords have a very key property that, when you add any chord to the spanning tree, it creates a unique *fundamental* cycle; this is a property that we will exploit later to get chordal increment values.

For now, we want to get a Max-Weight Spanning Tree (perhaps with a Modified Kruskal's algorithm that biases for larger weights rather than the typical min-weight formulation), such that we have min-weight chord edges. **We are creating this spanning tree on an undirected version of the DAG. This will be important in approximately one minute**.

From there, for any chord $c, ~Inc(c)$ is presented as the sum of the edges in the fundamental cycle in Fig 7 of the EPP paper ([linked again](https://faculty.cc.gatech.edu/~harrold/6340/cs6340_fall2009/Readings/micro96.pdf) for convenience), but there is a little more nuance than that. In writing this, I have realized that my attempts at implementing this were wrong, and while my hack works, it is not as elegant as the actual solution so I'll try to do it justice below.

---
Let's take the simple diamond graph $A \to B, ~A \to  C, ~B \to D, ~C \to D$ with backedge $D \to A$. This backedge is akin to our algorithm's $ENTRY \to EXIT$ backedge. Then, say our original path sum algorithm assigns weights of 0 to every edge except $A\to C$, for which the weight is 1. This is a perfectly valid edge valuation; the two acyclic paths are uniquely identified as 0 and 1.

From here, let's say our spanning tree is the following edges: $\{A\to C, ~B\to D, ~C\to D\}$. Then our chords are $\{A \to B, ~D \to A\}$. The algorithm for finding the increment of chord with spanning tree is as follows:
1. Find the (**<u>undirected</u>**) fundamental cycle in the graph induced by the spanning tree and the chord
2. Sum the edges in the fundamental cycle, taking care to *invert the sign* when we are going against the direction of the flow.
3. Bob's your uncle. Well, probably, for some of you. I have a few nice friends named Bob, no uncles though.

For our diamond-graph-with-backedge, let's concretely check both cases:
- For $D\to A$, $Inc(D\to A) = Val(D\to A)+Val(A\to C)+Val(C\to D) = 0 + 1 + 0 = 1$
- For $A\to B$, the cycle in the graph is actually $A \to B \to D \to C \to A$, where the $D \to C$ and $C \to A$ edges are *backwards*. Thus, $Inc(A\to B) = Val(A\to B)+Val(B\to D)-Val(D\to C)-Val(C \to A) = 0+0-0-1 = -1$.

We can treat the increment on the backedge as "always happening at $EXIT$", i.e. there's a constant +1 at the end before storing the path identifier into the trace. Thus, we have increments for each chord, and we can set the increments for all other paths to 0, i.e. we can leave them uninstrumented! Hooray!

For additional reference, the paper cited in the EPP paper is ["Efficiently Counting Program Events with Support for On-Line Queries"](https://dl.acm.org/doi/pdf/10.1145/186025.186027).

### Some problems with naively using the EPP algorithm
1. When using the LLVM framework, and working with an LLVM CFG, one needs to be careful to handle the overall control flow with care. As such, instead of directly converting the CFG into a DAG (which can ruin the control flow unless you use an `IRBuilder` to change the branching logic directly), we just create our own `DAGNode` and `DAG` structs to build and modify our DAG. We then use that DAG for steps 2 and 3 of the EPP paper, before applying our instrumentation logic "from the DAG to the CFG", as we'll discuss later.
2. The EPP paper describes instrumentation for counting the frequency with which each path is encountered (with something like `count[r]++`, where `r` represents the path identifier. We need to modify this to instead create a trace buffer, aka a huge array of ints that stores the path identifiers as they arrive in sequence. Paired with this trace buffer will be a trace index that needs to be incremented correctly at the $EXIT$ node so we keep track accurately.
3. In order to convert the path traces generated by the EPP algorithm on a per function basis into one unified Whole Program Path, you need to somehow distinguish each function's path identifiers. The way I chose to do this was by using a "base + offset" idea; the first function's base is 0, and its offset is the path identifier itself. Then, the second function's base is $0 + NumPaths(\text{first function})$, and the offset is the same idea. You can see where we're going with this - in general, since every path identifier is in the range $[0, ~NumPaths(\text{entry of func}) - 1]$, each range is cordoned off correctly. If function A has 2 paths, its paths are $\{0, ~1\}  $. Then, if function B has 3 paths, its paths will end up saved as $\{2, ~3, ~4\}  $. The base will be the sum of all of the previous functions.
4. Finally, we have the main disconnect between the paper and implementation. In the WPP paper §2, they mention the following:
> Whole program profiling requires a slight redefinition of a path,
so edges leading into a basic block containing a procedure call
terminate acyclic paths

As it turns out, they never really quite say how they did this. Dang. 

### Terminating acyclic paths at calls
I spent a couple days of my free time drawing flowgraphs in contrived scenarios before a conversation with my TA initiated a eureka moment of sorts. At that point, I had been working on a very specific test case (Fig 2 in the WPP paper):
<img src="https://github.com/abhamra/personalsite/blob/master/content/Images/initial_with_entry.png?raw=true" alt="initial cfg" align="right" width="23%" /> 
```c
int bar(int j) {
  if (j < 5)
    return j;
  else
    return 0;
}

int main(void) {
  for (int i = 0; i < 9; i++) {
    bar(i);
  }
  return 0;
}
```
which admits the above original CFG for `main`, where block `C` has the call.
The CFG for `bar` is simple, just a diamond graph to start. As such, we'll focus on `main`'s CFG for the call handling.

Before we discuss how to do the call handling, let's first discuss our output expectations. For `main`'s CFG, we expect the following 3 paths to be output by the trace:
1. $ENTRY\to A \to B\to C\to EXIT$ for the initial call to `bar` (somehow)
2. $ENTRY\to B\to C\to EXIT$ for the loop
3. $ENTRY\to B\to E\to EXIT$ for when we break out of the loop

The first key realization was that we had to handle calls before handling backedges, as this made the resulting acyclic paths easier to handle while also removing some backedges from the equation during call handling.

The second (and more important) realization was that if we wanted a call to terminate an acyclic path, we needed a way to go from a block ending in a call directly to the $EXIT$ node. From a block *ending in a call*...

---
ah.

---

The idea is simply to split blocks, such that you have an ordered list of blocks that corresponds to the original block, except each block either ends in a call or (in the case of the last one) may not have a call at all. In the above case, we can split block $C$ into the sequence $C_1;C_2$, where $C_1$ contains the call and $C_2$ contains the branching/bookkeeping, as well as the for loop's increment. Then, we can apply this algorithm of mine.
Given an ordered list of blocks $[C_{1}, C_{2}, \ldots , C_{n}]$, as well as $preds(C)$ and $succs(C)$:
1. For all $i \in [0, n-2]$, add the edge $C_i \to EXIT$ and $ENTRY\to C_{i+1}$. This terminates the path at callsite $C_i$ while allowing the new path to be handled with "fresh" instrumentation from $ENTRY$ to the post-call code at $C_{i+1}$.
2. For $C_{n-1}$ (the last block), there are two cases:
    1. If last block also ends in a call, then add $C_{n-1} \to EXIT$ and $ENTRY \to S ~\forall S \in succs(C)$. This connects the last block to $EXIT$ (as it has a call, so the path needs to be terminated) and connects all of $C$'s original successors to $ENTRY$ so the instrumentation works correctly.
    2. If the last block *doesn't* have a call, then add $ENTRY\to C_{n-1}$ and $C_{n-1} \to S ~\forall S \in succs(C)$. The last block is now a clean successor of the previous block with call. Itdoesn't need to go to an $EXIT$, so it flows from $ENTRY$. Then, as before, all of the successors of the original block $C$ are connected to it, and we are done!

<img src="https://github.com/abhamra/personalsite/blob/master/content/Images/graph_after_transform.png?raw=true" alt="transformed cfg" align="right" width="23%" /> 

To the right, you can see what the final DAG looks like. The call handling takes care of the backedge that was present originally! Just as a quick sanity check, let's also list out the paths we expect to see, and check if they line up with your hopes and dreams from before.
1. $ENTRY\to A\to B\to C_{1}\to EXIT$ (the initial)
2. $ENTRY\to C_{2}\to B\to C_{1}\to EXIT$ (the loop)
3. $ENTRY\to C_{2}\to B\to E\to EXIT$ (the end condition)

It seems like we've done a good job, give yourself a POTB.

It's also important to remember that this complicates the backedge calculation in the DAG; now we can't make use of the original CFG's flow relationships, and need to traverse manually to find our pesky backedges.

## Instrumenting our CFG (injecting stuff to trace stuff)
The last step in the arduous process of procuring but *one* trace of a silly little program is to write the code that actually, y'know, stores the trace. After figuring all the previous stuff out, this was way less fun, but here's generally how the instrumentation procedure works:
1. As a precursor to the actual instrumentation, create module level global variables for the trace buffer, trace index (a pointer into the buffer), and the path identifier.
2. $ENTRY$ Instrumentation: For each successor to the $ENTRY$ node, add a store to the `PathId` global variable, (re)initializing it at 0.
3. Chord Instrumentation: For every chord edge $(u, v)$, add a basic block in between with `llvm::SplitEdge` to get something like $(u, new), ~(new, v)$. In the $new$ block, load in the `PathId` then add the chord's increment to it.
4. $EXIT$ Instrumentation and overflow handling and printing at the end: For this, check if the trace index has reached the length you set your buffer. If not, store the `PathId` in the trace buffer with a [GEP](https://blog.yossarian.net/2020/09/19/LLVMs-getelementptr-by-example), then increment your trace index and (optionally) reset your `PathId` after storing (although it will likely be done by $ENTRY$ handling anyway).


## Heading to the $EXIT$
And that's pretty much it! We learned about the Whole Program Paths paper, Efficient Path Profiling, and some ideas necessary to jump from one to the other. I'm planning on trying to verify the path sum stuff, but beyond that I'm just glad I was exposed to this cool new idea and had the chance to think about it deeply.

Thank you for reading!
