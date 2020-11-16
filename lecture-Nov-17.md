# Dataflow Analysis

References:

  * A Unified Appraoch to Global Program Optimization
    by Gary A. Kildall, POPL 1973.
    Gary's Ph.D. thesis at U.W.
    He is also known for the CP/M operating system that was an
    early competitor with MS-DOS (Microsoft).

Analyses and Optimizations:

  * constant propagation and folding
  * common subexpression elimination
  * live expression analysis 

## Constant Propagation and Folding

    A:  a = 1
    B:  c = 0
    for i = 1...10 {
      C: b = 2
      D: d = a + b
      E: e = b + c
      F: c = 4
    }

Consider every possible execution path, propagating constants along
each path.  Then the analysis information P_N for a node N is the
*intersection* of the information from all paths to that node.

Let P^i_N be the information for the ith path from the entry node 
up to but not including node N.
Then P_N = Inter_i P^i_N.

    A                       {}
    A->B                    {(a,1)}
    A->B->C                 {(a,1),(c,0)}
    A->B->C->D             *{(a,1),(b,2),(c,0)}
    A->B->C->D->E           {(a,1),(b,2),(c,0),(d,3)}
    A->B->C->D->E->F        {(a,1),(b,2),(c,0),(d,3),(e,2)}
    A->B->C->D->E->F->C     {(a,1),(b,2),(c,4),(d,3),(e,2)}
    A->B->C->D->E->F->C->D *{(a,1),(b,2),(c,4),(d,3),(e,2)}
    ...

P_D = {(a,1),(b,2)}

This approach is known as meet-over-all-paths (MOP).

This approach is not practical because the number of paths are often
infinite.

Instead of waiting to the end to take the intersection, we could do it
eagerly, intersecting along the way with prior results.

    A                       {}
    A->B                    {(a,1)}
    A->B->C                 {(a,1),(c,0)}
    A->B->C->D              {(a,1),(b,2),(c,0)}
    A->B->C->D->E           {(a,1),(b,2),(c,0),(d,3)}
    A->B->C->D->E->F        {(a,1),(b,2),(c,0),(d,3),(e,2)}
    A->B->C->D->E->F->C     {(a,1)}
    A->B->C->D->E->F->C->D  {(a,1),(b,2)}
    ...C->D->E->F->C->D->E  {(a,1),(b,2),(d,3))}
    ...D->E->F->C->D->E->F  {(a,1),(b,2),(d,3))}
    ...E->F->C->D->E->F->C  {(a,1)}

Informal Global Analysis Algorithm For Constant Propagation & Folding

1. For entry node E, set P_E = {}
2. Process node E, send info to the successor nodes.
3. Intersect the incoming pools for each node with the prior pool
   for the node, if there is one. If not, take the incoming pool.
4. If the result for a node changed, then send updated info to the
   successor nodes.

## Generalization to other Static Analysis Problems

* An "transfer" function f that maps a node and an input pool to an
  output pool. For example,

        f(A,S) = {(a,1)} U (S - a)
        f(B,S) = {(c,0)} U (S - c)
        f(C,S) = {(b,2)} U (S - b)
        f(D,S) = if (a,n1) in S and (b,n2) in S then
                   {(d,n1+n2)} U (S - d)
                 else
                   S - d

* Initial value for the entry node.

* The information must form a meet-semilattice.

   - The meet operation /\ is commutative and associative.
   - The semilattice has a "bottom" element (e.g. empty set).
     (It may also have a "top" element, in which case it is a lattice.)
   - The elements are partially ordered in a way that
     jives with the meet operation.

        a <= b iff a /\ b = a

* The transfer functions must be homomorphisms. (Kildall)

        f(N, X /\ Y) = f(N, X) /\ (N, Y)

  Also, we say f is distributive.

  We can relax this requirement slightly:

        f(N, X /\ Y) <= f(N, X) /\ f(N, Y)

  and also restate it as monotonicity:

        x <= y --> f(x) <= f(y)

  When distributive, applying the meet "early" doesn't lose precision.

* For termination, need lattice to have finite descending chains.

## Common Subexpression Elimination

Example:

    T: r = a + b
    U: s = r + x
    V: t = (a + b) + x
    W: return t

    ==>

    r = a + b
    s = r + x
    return s

Need to keep track of which expressions are equivalent,
i.e., we need to track equivalence classes of expressions.

    P_T = {}
    P_U = { a | b | a+b,r }
    P_V = { a | b | a+b, r | x | r+x, s }
    P_W = { a | b | a+b, r | x | r+x, s, (a + b) + x, t }

The transfer function for common subexpression elimination:

For each subexpression e in the node N:

   * If e is already in an equiv. class, then it is redundant.
   * If not, create a new class in P with singleton e.
   * If node N is an assignment d = e, remove from P
     all expressions containing d. For each expression e' in P  
     that contains e, create e'' by replacing e with d, and place e''
     in the class of e'.

The meet operation P1 /\ P2 forms a partition over the expressions
common to both P1 and P2, with

    (P1 /\ P2)(e) = P1(e) /\ P2(e)

## Live Expression Analysis

Generalizes liveness analysis from variables to expressions.
An *expression is live* if it may be used in the future.

This analysis works backwards, from the end of the program to the
beginning. Flip edges to have it work forwards.

The transfer function is:

    If d := e in N,
       P <- P - {e| d in e }
    For each e in N,
       P <- P U {e} 

The meet operation is set union.

Initial pools are {}.

## Implementation

Worklist Algorithm:

    Place entry node in the worklist.
    While the worklist is not empty:
      Pop a none from the worklist
      Apply the transfer function.
      Place its successors who changed in the worklist.

Speeding up the ordering:

Form the condensation graph by collapsing strongly-connected components.

Process the condensation graph in topological order.
For each component, run the worklist algorithm to a fixed point.

