---
layout: post
title:  "High Availability"
---

Hi Suresh,

We have an observation about our contract classification, which we think impacts how we argue our
case in the paper. The question we consider is this: Is our current classification of contracts over
replicated data type operations as Highly Available (HA), Sticky Available (SA), and Unavailable
(UA), universally applicable? For instance, if we classify an operation as, say, SA but not HA, does
it mean that on any eventually consistent replicated data store: (a). the operation cannot be
provided with high availability, and (b). the operation can be provided with sticky availability?
Recall that we classify contracts according to their availability with respect to our model of the
replicated data store; if we classify as SA, it means that the operation cannot be executed with
high availability in our model of the replicated store. It follows that for our classification to be
universally applicable, we require our model to be the universal model of eventually consistent
replicated data stores. 

Is our model universal? The answer is no. In fact, there does not exist a universal model (M) of
replicated data stores such that an operation that is highly available on M is highly available
everywhere. In practice, off-the-shelf eventually consistent stores (eg: Cassandra) impose different
constraints on the kind of operations that can be performed. Therefore, an operation that can be
performed with high availability on Cassandra may not be performed with high availability on a
different store.  Nevertheless, there does exist a most general model of replicated data stores -
the model of the asynchronous distributed system. It is this model against which Gilbert and Lynch
proved CAP theorem, and Bailis et al introduced the notion of highly available transactions. This
model is most general in the sense that it imposes no other constraints on the behaviour of the
store other than requiring that it be asynchronous. Consequently, if an operation cannot be offered
with high availability on this model, it cannot be highly available on any eventually consistent
replicated data store. However, the converse is not true; if an operation is highly available on
this model, it need not be highly available on every model of the replicated data store.

Is the model that we used for our classification most general? Again, the answer is no. There are
certain operations that are not HA in our model, but which would be classified HA by Bailis et al.
An example is an operation (call it n) with contract:

∀(a,b). vis(a,b) /\ so(b,n) => vis(a,n)

Whereas this operation can be peformed with high availability in the most general model, this is
necessarily sticky available (SA) in our model. Another example is a transaction that requires
Monotonic Atomic View (MAV) isolation. While Bailis et al can offer support such transactions with
high availability, our model classifies them as sticky available.

Why don't we classify contracts against the most general model of a replicated data store? As a
matter of fact, we have only recently realized that our model is not the most general model. 
Nevertheless, we argue that it is more practically relevant model in the sense that it captures
the semantics of an application implemented on the top of an off-the-shelf data store. Such stores
aim to provide a transparent interface to their data by concealing replication from their clients.
Consequently, it is not possible for clients to query a certain replica or insist that their
operations be directed to a certain replica. Both the HA examples mentioned in previous paragraph
require applications to be replication-aware to be highly available; so, they cannot be offered with
high availability if the application uses an off-the-shelf data store.

Nevertheless, it should be noted that although contract classification is tied to the semantics of
the underlying store, the idea of classification itself is applicable to any model of the store.
Therefore, we can always present a classification (C') for the most general model such that our
classification coincides with that of Bailis et al's classification. However, the purpose of the
contract classification is to choose appropriate implementation strategy for the corresponding
operation, and, at this point of time, the implementation strategy for each class of contracts in C'
is not clear.

In the light of these new observations, I think we need to reconsider how we should present our
work. In the extreme case we might have to change our narrative to approach from the implementation
perspective, but I think there are alternatives. Can we meet sometime today/tomorrow to discuss
this?

