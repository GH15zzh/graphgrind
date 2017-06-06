GraphGrind
===========================
A NUMA-Aware Graph Processing Framework for Shared Memory
======================

Based on Ligra, a Lightweight Graph Processing Framework for Shared Memory
--------

Compilation
--------

Recommended environment

* Clang with CilkPlus and NUMA support: https://github.com/hvdieren/swan_clang
* Matching LLVM: https://github.com/hvdieren/swan_llvm
* Extended Cilkplus runtime: https://github.com/hvdieren/swan_runtime

Check out https://github.com/hvdieren/swan_tests for instructions on how
to set up the build environment.
 
After the appropriate environment variables are set, to compile,
simply run

```
$ make -j 16 
```

This is to compile and build with 16 threads in parallel. You can use the
number of your choice.

The remainder of the explanation is retained from Ligra.

Run Examples
-------
Example of running the code: An example unweighted graph
rMatGraph_J_5_100 and weighted graph rMatGraph_WJ_5_100 are
provided. Symmetric graphs should be called with the "-s"
flag for better performance. For example:

```
$ ./BFS -s rMatGraph_J_5_100
$ ./BellmanFord -s rMatGraph_WJ_5_100
``` 

For BFS, BC and BellmanFord, one can also pass the "-r" flag followed
by an integer to indicate the source vertex.  rMat graphs along with
other graphs can be generated with the graph generators at
www.cs.cmu.edu/~pbbs. By default, the applications are run four times,
with times reported for the last three runs. This can be changed by
passing the flag "-rounds" followed by an integer indicating the
number of timed runs.


Input Format
-----------
The input format of an unweighted graphs should be in one of two
formats.

1) The adjacency graph format from the Problem Based Benchmark Suite
 (http://www.cs.cmu.edu/~pbbs/benchmarks/graphIO.html). The adjacency
 graph format starts with a sequence of offsets one for each vertex,
 followed by a sequence of directed edges ordered by their source
 vertex. The offset for a vertex i refers to the location of the start
 of a contiguous block of out edges for vertex i in the sequence of
 edges. The block continues until the offset of the next vertex, or
 the end if i is the last vertex. All vertices and offsets are 0 based
 and represented in decimal. The specific format is as follows:

AdjacencyGraph  
&lt;n>  
&lt;m>  
&lt;o0>  
&lt;o1>  
...  
&lt;o(n-1)>  
&lt;e0>  
&lt;e1>  
...  
&lt;e(m-1)>  

This file is represented as plain text.

2) In binary format. This requires three files NAME.config, NAME.adj,
and NAME.idx, where NAME is chosen by the user. The .config file
stores the number of vertices in the graph in text format. The .idx
file stores in binary the offsets for the vertices in the CSR format
(the <o>'s above). The .adj file stores in binary the edge targets in
the CSR format (the <e>'s above).

Weighted graphs: For format (1), the weights are listed at the end of
the file (after &lt;e(m-1)>). Currently for format (2), the weights
are all set to 1.

By default, format (1) is used. To run an input with format (2), pass
the "-b" flag as a command line argument.

By default the offsets are stored as 32-bit integers, and to represent
them as 64-bit integers, compile with the variable LONG defined. By
default the vertex IDs (edge values) are stored as 32-bit integers,
and to represent them as 64-bit integers, compile with the variable
EDGELONG defined.


Ligra Graph Applications
---------
Currently, Ligra comes with 8 implementation files: BFS.C
(breadth-first search), BC.C (betweenness centrality), Radii.C (graph
radii estimation), Components.C (connected components), BellmanFord.C
(Bellman-Ford shortest paths), PageRank.C, PageRankDelta.C and
BFSCC.C (connected components based on BFS).


Ligra Data Structure and Functions
---------
### Data Structure

**vertexSubset**: represents a subset of vertices in the
graph. Various constructors are given in ligra.h

### Functions

**edgeMap**: takes as input 3 required arguments and 3 optional arguments:
a graph *G*, vertexSubset *V*, struct *F*, threshold argument
(optional, default threshold is *m*/20), an option in {DENSE,
DENSE_FORWARD} (optional, default value is DENSE), and a boolean
indiciating whether to remove duplicates (optional, default does not
remove duplicates). It returns as output a vertexSubset Out
(see section 4 of paper for how Out is computed).

The *F* struct contains three boolean functions: update, updateAtomic
and cond.  update and updateAtomic should take two integer arguments
(corresponding to source and destination vertex). In addition,
updateAtomic should be atomic with respect to the destination
vertex. cond takes one argument corresponding to a vertex.  For the
cond function which always returns true, cond_true can be called. 

```
struct F {
  inline bool update (intT s, intT d) {
  //fill in
  }
  inline bool updateAtomic (intT s, intT d){ 
  //fill in
  }
  inline bool cond (intT d) {
  //fill in 
  }
};
```

The threshold argument determines when edgeMap switches between
edgemapSparse and edgemapDense---for a threshold value *T*, edgeMap
calls edgemapSparse if the vertex subset size plus its number of
outgoing edges is less than *T*, and otherwise calls edgemapDense.

DENSE and is a read-based version where all vertices not satisfying
Cond loop over their incoming edges and DENSE_FORWARD is a write-based
version where each frontier vertex loops over its outgoing edges. This
optimization is described in Section 4 of the paper.

Note that duplicate removal can only be avoided if updateAtomic
returns true at most once for each vertex in a call to edgeMap.

**vertexMap**: takes as input 2 arguments: a vertexSubset *V* and a
function *F* which is applied to all vertices in *V*. It does not have
a return value.

**vertexFilter**: takes as input a vertexSubset *V* and a boolean
function *F* which is applied to all vertices in *V*. It returns a
vertexSubset containing all vertices *v* in *V* such that *F(v)*
returns true.

```
struct F {
  inline bool operator () (intT i) {
  //fill in
  }
};
```

To write your own Ligra code, it would be helpful to look at the code
for the provided applications as reference.

Currently the results of the computation are not used, but the code
can be easily modified to output the results to a file.

To develop a new implementation, simply include "ligra.h" in the
implementation files. When finished, one may add it to the ALL
variable in Makefile. The function that is passed to the Ligra driver
is the following Compute function, which is filled in by the user. The
first argument is the graph, and second argument specifies the
starting vertex for algorithms that require one.

```
template<class vertex>
void Compute(graph<vertex> GA, intT start){ 

}
```

For weighted graph applications, add "#define WEIGHTED 1" before
including ligra.h.

To write a parallel for loop in your code, simply use the parallel_for
construct in place of "for".

Resources  
-------- 

Jiawen Sun, Hans Vandierendonck and Dimitrios S. Nikolopoulos [*GraphGrind:
Addressing Load Imbalance of Graph Partitioning*]. To appear in Proceedings
of the International Conference on Supercomputing (ICS), 2017.

Based on: Julian Shun and Guy Blelloch. [*Ligra: A
Lightweight Graph Processing Framework for Shared
Memory*](http://www.cs.cmu.edu/~jshun/ligra.pdf). Proceedings of the
ACM SIGPLAN Symposium on Principles and Practice of Parallel
Programming (PPoPP), pp. 135-146, 2013.
