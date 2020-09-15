# Register Allocation: Graph Coloring via Sudoku

* Goal: map each variable to a register such that no two interfering
  variables get mapped to the same register. 
  
* Secondary goal: minimize the number of stack locations that need
  to be used.

* In the interference graph, this means that adjacent vertices must be
  mapped to different registers. 

* If we think of registers and stack locations as colors, then this
  becomes an instance of the *graph coloring problem*.

If you aren't familiar with graph coloring, you might instead be
familiar with another instance of it, *Sudoku*.

Review Sudoku and relate it to graph coloring.

    -------------------
    |  9  |5 3 7|  1 4|
    |  3 8|  4 6|  9  |
    |4    |1    |2    |
    -------------------
    |     |     |    2|
    |7   9|8   2|1   5|
    |6    |     |     |
    -------------------
    |    4|    8|    6|
    |  6  |4 9  |7 2  |
    |8 7  |6 2 3|  4  |
    -------------------

* Squares on the board corresponds to vertices in the graph.
* The vertices for squares in the same *row* are connected by edges.
* The vertices for squares in the same *column* are connected by edges.
* The vertices for squares in the same *3x3 region* are connected by edges.
* The numbers 1-9 are corresponds to 9 different colors. 

What strategies do you use to play Sudoku?

* Pencil Marks? (most-constrained-first)
  We'll record the colors that cannot be used,
  i.e. the *saturation* of the vertex.
* Backtracking?
    * Register allocation is easier than Sudoku in
      that we can spill to the stack, i.e., we can always add more colors.
    * Also, it is important for a register allocator to be
	  efficient, and backtracking is exponential time.
	* So it's better to *not* use backtracking.

We'll use the DSATUR algorithm of Brelaz (1979).
Use natural numbers for colors.
The set W is our worklist, that is, the vertices that still need to be
colored.

    W <- vertices(G)
    while W /= {} do
      pick a vertex u from W with maximum saturation
      find the lowest color c not in { color[v] | v in adjacent(u) }.
      color[u] <- c
      W <- W - {u}

Initial state:

    {}     {}     {}    {}
    v ---- w ---- x     t.1
	       |\___     ___/|
		   |    \   /    |
		   |     \ /     |
		   y ---- z     t.2
		   {}     {}    {}

There's a tie amogst all vertices. Color v 0. Update saturation of adjacent.

    {}    {0}     {}    {}
    v:0--- w ---- x     t.1
	       |\___     ___/|
		   |    \   /    |
		   |     \ /     |
	       y ---- z     t.2
		   {}    {}     {}

Vertex w is the most saturated. Color w 1. Update saturation of adjacent.

    {1}   {0}    {1}    {}
    v:0--- w:1--- x     t.1
	       |\___     ___/|
		   |    \   /    |
		   |     \ /     |
	       y ---- z     t.2
		  {1}    {1}    {}


There is a tie between x, y, and z. Color x 0. 

    {1}   {0}    {1}    {}
    v:0--- w:1--- x:0   t.1
	       |\___     ___/|
		   |    \   /    |
		   |     \ /     |
	       y ---- z     t.2
		  {1}    {1}    {}

There is a tie between y and z. Color z 0.

    {1}   {0}    {1}    {0}
    v:0--- w:1--- x:0   t.1
	       |\___     ___/|
		   |    \   /    |
		   |     \ /     |
	       y ---- z:0   t.2
		  {0,1}   {1}    {}

Vertex y is the most saturated. Color y 2.

    {1}   {0,2}   {1}    {0}
    v:0--- w:1--- x:0   t.1
	       |\___     ___/|
		   |    \   /    |
		   |     \ /     |
	      y:2--- z:0    t.2
		  {0,1}  {1,2}   {}

Vertex t.1 is the most saturated. Color t.1 1.

    {1}   {0,2}   {1}    {0}
    v:0--- w:1--- x:0   t.1:1
	       |\___     ___/|
		   |    \   /    |
		   |     \ /     |
	      y:2--- z:0    t.2
		  {0,1}  {1,2}   {1}

Vertex t.2 is the only one left. Color t.2 0.

    {1}   {0,2}   {1}    {0}
    v:0--- w:1--- x:0   t.1:1
	       |\___     ___/|
		   |    \   /    |
		   |     \ /     |
	      y:2--- z:0    t.2:0
		  {0,1}  {1,2}   {1}

* Create variable to register/stack location mapping:
  We're going to reserve `rax` and `r15` for other purposes,
  and we use `rsp` and `rbp` for maintaining the stack,
  so we have 12 remaining registers to use:

        rbx rcx rdx rsi rdi r8 r9 r10 r11 r12 r13 r14
	
  Map the first 12 colors to the above registers, and map the rest of
  the colors to stack locations (starting with a -8 offset from `rbp`
  and going down in increments of 8 bytes.

		0 -> rbx
		1 -> rcx
		2 -> rdx
        3 -> rsi
        ...
        12 -> -8(%rbp)
        13 -> -16(%rbp)
        14 -> -24(%rbp)
        ...

  So we have the following variable-to-home mapping

		v -> rbx
		w -> rcx
		x -> rbx
		y -> rdx
		z -> rbx
		t.1 -> rcx
		t.2 -> rbx

* Update the program, replacing variables according to the variable-to-home
    mapping. We also record the number of bytes needed of stack space
    for the local variables, which in this case is 0.

    Recall the example program after instruction selection:

        locals: (v w x y z t.1 t.2)
        start:
          movq $1, v
          movq $46, w
          movq v, x
          addq $7, x
          movq x, y
          movq x, z
          addq w, z
          movq y, t.1
          negq t.1
          movq z, t.2
          addq t.1, t.2
          movq t.2, %rax
          jmp conclusion

    Here's the output of register allocation, after applying
    the variable-to-home mapping.

		stack-space: 0
		start:
          movq $1, %rbx
		  movq $46, %rcx
		  movq %rbx, %rbx
		  addq $7, %rbx
		  movq %rbx, %rdx
		  movq %rbx, %rbx
		  addq %rcx, %rbx
		  movq %rdx, %rcx
		  negq %rcx
		  movq %rbx, %rbx
		  addq %rcx, %rbx
		  movq %rbx, %rax
          jmp conclusion
