---
layout: post
title:  "Paper Review - CALM"
---

Suresh mentioned this
[paper](http://db.cs.berkeley.edu/papers/cidr11-bloom.pdf) some days back:

    Consistency Analysis in Bloom: A CALM and Collected Approach

The paper relates the concepts of Consistency and Logical Monotonicity
in distributed programs via the CALM principle:

    Logically monotonic distributed programs are eventually consistent
    by nature.

A computation is logically monotonic if it never needs to retract its
previous output upon seeing more input. Relational operations like
selections, projections and joins are logically monotonic, as adding
more input only increases their output. In contrast, aggregations like
count, min etc are non-monotonic. Distributed programs that operate
over unordered collections using only monotonic operators (eg: a
replicated selection) are eventually consistent by default. However,
if the computation involves a non-monotonic operation, then there is a
need for coordination among replicas to ensure consistency. The points
in the program where there is a need to introduce coordination in
order to ensure convergence are called points of order. Since
coordination is expensive (high latency and low availability), a
distributed programmer has to carefully build his programs to ensure
convergence while minimizing non-monotonic code and points of order in
his programs.

Reasoning about high-level program properties, such as convergence, in
presence of loosely coordinated consistency is difficult for
programmers. This task is made even more difficult due to the
mismatch between the Von Neumann model of computation in imperative
languages, and actual reality of distributed programming. Von
Neumann's model captures state as an ordered array of addresses, and
computation as an ordered sequence of instructions. In contrast,
distributed programming model is "disorderly" by nature; inputs arrive
in disorderly fashion, and different parts of the program execute at
different times. 

To make it easier to write and reason about distributed programs,
there is a need to bridge the existing gap between programming models
and reality. The paper proposes a declarative programming language
called Bloom to bridges this gap by encouraging "disorderly"
programming. In Bloom, state is captured as unordered relations, and
computation is expressed in logic as unordered set of declarative
rules, each consisting of unordered set of predicates. Authors build a
temporal logic called Dedalus as theoretical foundation of Bloom.
Nevertheless, Bloom's programming model can be understood as a
straightforward combination of Dikjstra's guarded command language
(GCL), Lamport's TLA and relational algebra. Recall that in GCL, a
program is just an ordered collection of guarded commands, which get
executed whenever their corresponding guards become true. Further, we
have the assurance that in a single time step, only one of the guards
can be true. As such, GCL is an excellent model for event-driven
programming, where guards are predicates which come true on triggering
of an event. This is the reason why most distributed algorithms are
presented in GCL fashion (recall broadcast algorithms from Patrick's
course).

Lamport's TLA is temporal logic based specification language for
distributed algorithms. TLA encourages formal description of
distributed algorithms in GCL style, where guards are event predicates
(eg: onRecv), and commands are "actions". An action describes how to
compute the value of a state variable at next time step using values
at current time step (eg: k' = k+1). Therefore, a guarded action
describes how state is changed when an event (eg: receive of a
message) occurs. At the top-level, and algorithm in TLA is a predicate
asserting that atleast one of the event predicates is true at every
time step. (A TLA-style specification of reliable channel behaviour
written in Z3 is [here](https://github.com/pbanzal/DS_Proj/blob/master/ReliableChannel.z3)).

Bloom programs can be viewed as executable TLA specs, whose state is
only made of relations (modeled as unordered collections). Depending
on the application, some of these relations are persistent (called
tables), while some are non-persistent (called scratch). SQL-style
statements (like SELECT, GROUP BY etc) are used to read and write from
these relations. In Bloom, even network channels used for
communication are relations; so, I/O is also done through SQL
statements. In this model, reasoning about convergence, and
determining "points of order" becomes easy. A naive solution would be
to observe if the path from input channel to output channel contains
application of any non-monotonic operators (such as GROUP BY). Each
such application is a point of order, as there is a need for
coordination to ensure that the result of non-monotonic operation is
consistent across replicas. They say more intelligent analyses are
possible. They have a system that implements the naive solution and
use it to show that a state-based shopping cart RDT is has more points
of order (hence, requires more coordination) than an operation-based
"disorderly" shopping cart RDT. 

The Bloom model and our model are closely related in the following ways:

1. Both encourage _disorderly_ programming - Bloom's state is made up
   of unordered collections, while our _context_ (recall OPER rule) is an
   unordered set of effects. However, Bloom's model is more general in
   two major ways: 1. It allows the state to be modified (DELETEs and
   UPDATEs to relations), and 2. It allows replicas to be uniquely
   identified, and permits one-to-one communication among replicas.
   This generality is required in their case, as Bloom aims to be a
   general purpose language for distributed programming (their
   "getting started" example is a chat application). In contrast, we
   aim to provide a programming model that makes it easy to implement
   (and reason about) replicated data types over an off-the-shelf
   eventually consistent data store like Cassandra or RiakDB. As a
   result of our focused approach: 1. Our programming model is simple -
   Programmers never have to implement coordination/communication or
   I/O logic; they only need to implement RDT operations, which are
   always folds over the context, 2. Our implementation is effecient -
   we have shim layer caching, and we trigger summarization at
   appropriate time. We are also aided by the knowledge of consistency
   requirements of RDT operations.
2. Both models impose coordination only when needed - In Bloom, there
   is a _consistency analysis_ to identify points of order, where
   coordination is needed, whereas we expect programmers to declare
   consistency requirements of RDT operations. While Bloom treats
   consistency as an on/off switch, we recognize that different
   operations require different levels of consistency, which can have
   varied impact over availability of the system. Our contract
   language is expressive enough to capture these fine distinctions,
   and our implementation is sensitive to such distinctions. As our
   results (reference to the graph showing the latency when all ops are
   executed unavailable) indicate, distinguishing between levels of
   consistency needed by operations is of utmost practical relevance.


Shopping Cart Example
=====================

Datatypes and operations
-----------------------

Items table (Key: itemID)

- getItemFromStock
- removeItemFromStock (Qty)

Carts table (key: cartId)

- createCart 
- removeCart

CartItems table (Key: cartID)

- addItemToCart (itemID, Qty)
- getItemsFromCart
- removeItemFromCart (itemID, Qty)

Availability Contracts
----------------------

getItemsFromCart : ∀a. soo(a,n) => vis(a,n) -- Read my writes

Transactions
------------

1. User logs in
  - createCart
2. User searches an item
  - getItemFromStock
2. User adds q nos of item i to cart
  - addItemToCart (i,q)
  - getItemsFromCart
3. User removes q nos of item i from cart
  - removeItemFromCart (i,q)
  - getItemsFromCart
4. User checks out
  <!-- User clicks on checkout. Checkout 
       sequence gets initiated. -->
  - getItemsFromCart
  - getItemsFromStock 
  <!-- User takes time to review.  Then, clicks on "Order Now". -->
  <!-- If user takes too much time, the sequence gets aborted. He 
       will have to initiate new sequence by clicking on checkout
       again. -->
  <!-- Otherwise, the sequence continues. Note that he may connect to
       an arbitrary replica now. -->
  - getItemsFromCart
  - getItemsFromStock
  <!-- Third-party payments -->
  <!-- After payment succeeds, user may connect to a different replica -->
  - getItemsFromCart
  - removeItemFromStock (i,q)
  - removeCart

Repeatable read transaction has to involve a read to shared data item.

RUBiS Auction Example
=====================

Datatypes and operations
-----------------------

Items table (Key: itemId)
<!-- Contains item name, description, seller's userId, min price, and
     max bid (Initially zero. But minumum bidding amt is min price) -->
- addItem (itemId,itemDetails, sellerId, minPrice)
- updateMaxBid (itemId, amt)
- getItem (itemId)
- removeItem (itemId)

ItemBids table (Key: itemId)
<!-- Materialized view to help get all bids 
   for an item.-->
<!-- Why not store ItemBids in items table? -->
<!-- Item lookup is a very common operation. We don't want to fetch
    all bids for an item everytime someone looks it up. We just need
    its current price, which is also the max bid. We store that in
    items table -->
<!-- Can't we get all bids for an item by querying the bids table? --> 
<!-- Technically we can. However, bids table is indexed by bidId, and 
     cassandra doesn't support secondary indexes. So, querying based
     on itemId will have to be implemented by scanning the entire
     table. -->
- addItemBid (itemId,userId,amt)
- getBidsForItem (itemId)
- removeItemBid (itemId,userId)
- getMaxBid (itemId)

Users table (key: userId)
<!-- User's profile details and amount left in his wallet -->
- getUser (userId)
- withdrawFromWallet (userId, amt)
- depositToWallet (userId, amt)

UserBids (key:userId)
<!-- Materialized view to get all bids by a user -->
- addUserBid (userId, itemId, amt)
- getBidsByUser (userId)
- removeUserBid (userId, itemId)

<!-- we don't need this 
  Currently active bids
  Bids table (key: bidId)
  - doBid (bidId, itemId, userId, amt)
  - getBid (bidId)
  - unBid (bidId)
-->


Transactions
------------

1. User logs in
  <!-- Print his name and amt left in his wallet -->
  - getUser
2. User browses an item
<!-- Display item details, including min price and max bid on that item -->
  - getItem
3. UserId bids for an itemId with bidamt.
  <!-- Get item details to check if the bidamt > min-price -->
  - getItem (itemId)
  <!-- Check if user has sufficient balance (bidamt >= balance)-->
  - getUser (userId)
  - withdrawFromWallet (userId,bidamt)
  - addUserBid (userId, itemId, bidamt)
  - addItemBid (itemId, userId, bidamt)
  <!-- If bidamt > maxbid, then update maxbid -->
  <!-- Note: if there are concurrent updateMaxBids, then arbitration
       chooses max of them. -->
  - updateMaxBid (itemId, bidamt)
4. UserId deposits amt in his wallet
  - depositToWallet (userId,amt)
5. UserId wants to see all his bids
  - getBidsByUser (userId)
  <!-- for each bidId from the above set ... -->
6. After viewing his bids, UserId wants to cancel one of the bids.
  - getMaxBid (itemId) <!-- let this return maxbidder and maxbid -->
  - removeUserBid (userId, itemId)
  - removeItemBid (itemId, userId)
  <!-- If userId is maxbidder, we update max bid -->
  - updateMaxBid (itemId, maxbid)
5. Seller adds an item
  - addItem (new itemId, itemDetails, sellerUserId, minPrice)
  <!-- The max bid for itemId, when it is newly added, is zero.-->
6. Seller (with sellerUserId) concludes auction for itemId
  - getMaxBid (itemId) <!-- returns maxbidder and maxbid -->
  - depositToWallet (sellerUserId, maxbid)
  <!-- For max bidder, whose bids succeeded, just cleanup -->
  - removeUserBid (maxbidder, itemId)
  - removeItemBid (itemId, maxbidder)
  <!-- For every other user, give their money back -->
  - getBidsForItem (itemId) <!-- for every bidderUserId and bidamt returned ... -->
  - removeItemBid (itemId,bidderUserId)
  - removeUserBid (bidderUserId,itemId)
  - depositToWallet (bidderUserId,bidamt)
