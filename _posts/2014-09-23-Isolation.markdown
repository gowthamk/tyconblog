---
layout: post
title:  "Isolation in Distributed Transactions"
---

Questions
---------

What is Isolation in ACID? How is it different from Atomicity and
Consistency. How does it relate to Serializabilty and Linearizability.
What are weak isolation guarantees? What effect does CAP theorem has
on isolation?

Introduction
------------

Let us start with Gilbert and Lynch's discussion on ACID in their
paper proving CAP theorem. Interactions with web services are expected
to behave in transactional manner:

1. Transactions commit or fail in their entirety (Atomic) -- what does
   it mean for a transaction to commit/fail?
2. Committed transactions are visible to all future transactions
   (Consistent) -- If the definition of "commit" itself means being
   visible to future transactions, then isn't consistency <=>
   atomicity? Moreover, what is a "future" transaction?
3. Uncommitted transactions are isolated from each other (Isolated) --
   Does this mean that isolation is converse of consistency? Converse
   because: if a transaction sees another transaction, then the
   visible transaction better be committed.
4. Once a transaction is committed it is permanent (Durable).


1.Atomicity:
--------------

1. A transaction is said to be committed iff atleast one of its
   constituent operations is applied to a replica.
2. In our formalization, an operation is applied to a replica $$r$$
   iff its effect $$\eta$$ is present in $$\Theta(r)$$ (recall that
   $$\Theta(r)$$ is a monotonically increasing set. There is no
   rollback).
3. Therefore, a transaction is said to be committed, iff atleast
   one of its operations engendered an effect $$\eta$$ such that
   $$\eta \in \Theta(r)$$ for some replica $$r$$. 
4. Let us define a predicate $$com(\eta) = \exists r. \eta \in
   \Theta(r)$$. We say an effect $$\eta$$ is committed iff
   $$com(\eta)=true$$. Note that if an effect is committed, it is
   bound to be visible outside the transaction that caused this
   effect. (Note : converse it also true, guaranteeing certain amount
   of isolation. See the discussion on isolation).
5. A transaction is said to be atomically committed _at_ replica $$r$$,
   iff all the effects it engendered are committed _at_ replica $$r$$. 
   <!-- Our formalization satisfies the following property: Every
   operation in every session is executed: <br /> <br /> If
   $$E;\Theta; \langle s,~i,~op(v); \sigma \rangle \cup \Sigma
   \longrightarrow E';\Theta'; \langle s,~j,~\sigma \rangle \cup
   \Sigma$$, then $$\exists \eta \in E'.A.  ~ {\sf SessID}(\eta)=s$$
   $$\wedge {\sf SeqNo}(\eta)=i \wedge {\sf oper}(\eta) = op$$ <br />
   <br /> Consequently, we can simply say that a transaction is
   atomically committed iff all the effects it engendered are
   committed. -->
6. Finally, a transaction is said to be atomically committed, if and
   only if for every replica $$r$$, either the transaction is
   atomically committed at $$r$$, or it is not committed at $$r$$.
7. Our semantics guarantee atomicity for a single operation as evident
   from the fact that every effect is eventually committed.
   Consequently, transactions of single operations are atomic by
   default.


2.Consistency
-------------

First, let us clarify Sequential Consistency vs Serializability vs
Linearizability:

Sequential consistency requires that all data operations appear to
have executed atomically in some sequential order that is consistent
with the order seen at every individual process. 

If instead of individual data operations, we apply sequential
consistency to transactions, the resultant condition is called
serializability in database theory.

Linearizability imposes more constraints on an order decided by
sequential consistency - the order needs to make sense to an external
observer. Let us say that an external observer is running two threads
A and B. If thread A performs action a0, then syncs with thread B
following which B performs b0. A sequentially consistent execution can
order b0 before a0, as long as both A and B perceive the same order.
On the other hand, such an execution is not valid under
linearizability condition. Linearizability dictates that an operation
should take effect instantaneously before the perceived end of the
operation.

Evidently, linearizability is a stronger constraint than sequential
consistency (or serializability). Nevertheless, while linearizability
is a local property (i.e., composable), sc is not. (What does this
mean?)

The very fact that ACID consistency refers to all "future"
transactions conveys that it refers to linearizable or sequential
consistency. Considering that ACID is a standard for observable
effects of a transaction, "future" needs to be interpreted in context
of the external observer who is issuing transactions through
concurrent sessions. Hence, consistency in ACID is infact linearizable
consistency.

Our formal system understands sequential consistency requirement.
Every SC operation in a valid execution paritions the set of effects
as some that executed _before_ it and others _after_ it. However, this
paritioning may not correspond to the paritioning expected by the
external observer.

Notice that linearizability dictates that an effect of a transaction
be visible to all future transactions, but it does not define (a).
what constitutes the effect of a transaction, and (b). when is a
the effect of a transaction said to be visible to a future
transaction. In this sense, it is parameterized over the definitions
of (a). _Effect of a transaction_ and (b). _Visibility_ relation among
transactions (specifically, its co-domain). These are precisely the
definitions provided by atomicity and isolation conditions. ACID
atomicity defines the effect of a transaction as the sum of effects of
all its consituent operations, whereas ACID isolation defines
visibility relation between transactions as cross product of their
effects. In other words, ACID dictates that the store support
linearizable isolated atomic transactions . Quite often in practice,
atomicity is a definitional attribute of a transaction and it is
assumed that if a store supports transactions, it supports atomic
transactions. Consequenty, we simply say that an ACID store need to
support linearizable isolated transactions.

The CAP theorem, as stated by Gilbert and Lynch, is intended to
establish that ACID guarantees are not achievable with high
availability under network paritions. Consequently, C in CAP is
interpreted as linearizable consistency:

<pre>
  The most natural way of formalizing the idea of a consistent service
  is as an atomic data object. Atomic, or linearizable, consistency is
  the condition expected by most web services today.  Under this
  consistency guarantee, there must exist a total order on all
  operations such that each operation looks as if it were completed at
  a single instant. 
</pre>

But Bailis et al seem to disagree (do they?):

<pre>
  As formally proven, the CAP Theorem pertains to a data consistency
  model called linearizability, or the ability to read the most recent
  write to a data item that is replicated across servers. However,
  despite its narrow scope, the CAP Theorem is often misconstrued as a
  broad result regarding the ability to provide ACID database
  properties with high availability; this misunderstanding has led to
  substantial confusion regarding replica consistency, transactional
  isolation, and high availability.  The recent resurgence of
  transactional systems suggests that programmers value transactional
  semantics, but most existing transactional data stores do not
  provide availability in the presence of partitions.
</pre>

3.Isolation
------------

ACID Atomicity gurantees that if a transaction is committed, then all
its effects are visible to subsequent transactions. But, what if the
transaction is uncommitted? Should its effects be made visible? ACID
isolation requires that such effects to be not visible. Uncommitted
transactions must be isolated from each other.

Now, consider what it means in our formal model. In our model, an
effect will be visible if and only if it is added to $$\Theta(r)$$ for
some replica $$r$$. But, if the effect is added to $$\Theta(r)$$, then
by definition the transaction is committed. This implies that only
effects of committed transactions are visible to subsequent
transactions in our model. This lets us conclude that our store offers
isolation (as required by ACID) by default.

<!-- ACID isolation requires that  effect 
Atomicity - what to show
Consistency - whom to show
Isolation - what to see -->

ANSI SQL defines 4 isolation levels:

1. Perfect Serializability - Prevents phantoms + everything below.
2. Repeatable Read - Prevents "fuzzy reads" and "dirty reads and
   writes"
3. Read Committed - Prevents "dirty reads and writes"
4. Read Uncommitted - Prevents "dirty writes"

1. What is serializability doing in isolation categories?
2. Jim Gray 
   ([introduced ](http://research.microsoft.com/en-us/um/people/gray/papers/theTransactionConcept.pdf)) 
   the transaction concept with only A,C and D. No isolation.

([](http://wiki.hsr.ch/Datenbanken/files/Paper_ANSI_SQL_Isolation_Levels_Stefan_Luetolf_V2_1.pdf))

Transaction Availability
========================

Bailis et al categorize availability of system in presence of
transactions into transactional availability & sticky transactional
availability. 

1. A system provides transactional availability if, given replica
   availability for every data item in the transaction, the
   transaction eventually commits. A transaction has replica
   availability of it can contact at least one replica for every item
   it attempts to access.
2. A system provides sticky transactional availability if, given
   sticky availability for each data item it accesses, a transaction
   eventually commits or internally aborts.







