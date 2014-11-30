---
layout: post
title:  "Deterministic Parallelism with Relational Monad"
---

Introduction
============

Deterministic-by-construction parallel programming is largely an open
problem. One popular approach is to introduce constrained shared
memory abstractions such as TVars of STM, IVars and LVars in haskell.
All of these solutions require programmers to reason about shared
memory, require significant effort from programmers to write programs
in the programming model, or incur heavy run-time penalty. 


Another popular approach is automatic parallelization. This approach
is heavily influenced by third homomorphism theorem, and 

1. We have a uniform way to represent any data structure as a set of
relations.
2. Merging relations is trivial - it is simply union. It is
   commutative, associative, and idempotent.
3. If we have an operation that monotonically grows the data structure
   - like for example building a webgraph or a social network graph,
     then we can perform the operation in parallel, while lazily
     merging results. 
4. Eventual consistency - updates from one thread are lazily
   propagated to other threads.
4. Once the operation concludes, we can reconstruct the data structure
   from relations, and slide it smoothly into the program.
We want to take fully functional implementations, and use them to
build data-parallel computations.
No encapsulated state hiding under LVars.

constructing the friendship graph among friends of a person:

    friendshipNetwork (p:Person) = 
      let pfs = friendsOf p in
        foldl (map pfs (fn pf => (pf,friendsOf pf))) emptyGraph 
          (fn (g,(pf,pffs)) => foldl pffs g (fn (g',pff) => addEdge g' (pf,pff))


nary-tree traversal:

    traverse (t:'a tree) = case t of
      Branch (x,subts) => [x] `concat` foldl (map subts traverse) [] concat
    | Leaf => []


calculating connected component in graph g
    connectedComponent (v : Vertex) (g : Graph) (seen : Vertex Set) =
      let nbrs = neighbours(v,g) in
        foldl nbrs (seen `union` v)
          (fn (acc,v') => if v' `notIn` acc
            then connectedComponent v' g acc
            else acc)
