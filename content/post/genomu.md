---
Description: Attempt to document some design decissions made during genomu development.
Keywords:
- CAP theorm
- PACELC
Section: post
Slug: consistency-management-in-distributed-settings
Tags:
- DB
- consistency
- distribution
Title: Consistency management in distributed settings
Topics:
- Development
- DB
date: 2014-11-13
---


Consistency management in distributed settings
==============================================


Disclaimer
----------

I just participate in numerous design discussions related to genomu development. However all ideas and 99% of the implementation due to [@yrashk](http://github.com/yrashk). I decided to write this post to document the design of genomu inspite the fact that this great development didn't take off.


Introduction
------------

Being a consultancy agency we had a client who have ordered a development of a backend for a system
for doing sport events betting. The business idea behind the system is to sell paper tickets which contain a list of sport events along with three scratch areas for choices such as (team A wins, team B wins, tie). Customers scratch out their choices. The more areas they scratch the more expensive the bet.
Special hardware terminals installed at points of sales in order to scan the user's choices and validate winning tickets. Payout to a winner would depend on how many scratched areas she got correctly.


## Set of requirements for a backend

1. Backend should be `always` available for accepting tickets with users' choices
2. Backend must stop any payouts if there is any inconsistency present in the system
3. Payout approximation and jackpot calculation should be calculated real time and displayed on the web page of the customer


## Requirements analysis

At first sight we thought that we have a contradicting sets of requirements which would violate Dr. Eric Brewer's CAP theorem. However we noticed that we actually have different types of updates with different requirements.

* tickets - representing new bets
* events - information about sport event outcomes
* payouts - update which invalidates the ticket

Tickets and events require high availability. Payout require full consistency. We evaluated a number of existing databases which would give as guaranties we need. We were unable to find one. Therefore we decided to roll our own. Our client didn't mind to sponsor this development.


## Problem statement

Given set of the requirements it was obvious to us that in order to achieve high availability we would need to have a distributed system. Dealing with distributed systems is always a challenge due to a necessity to maintain an appropriate consistency level of the system's state. CAP theorem in it's most often used interpretation states that we cannot have all three (C,A,P) properties in a single system. However there is a more advanced interpretation of the theorem called [PACELC](http://dbmsmusings.blogspot.ca/2010/04/problems-with-cap-and-yahoos-little.html)[^1].

Using PACELC we can design a system where for some types of updates we would choose full consistency and for others we would choose high availability. In terms of PACELC we need a system which is

   * PA/EL for tickets and events
   * PC/EC for payouts

How do we configure the consistency options for different types of updates? The answer is NRW model.

  * N - how many replicas of the data we want
  * R - how many servers' replies we wait until we conclude that we have a consistent read
  * W - how many servers' acknowledgements we would wait until we consider data to be fully committed

[^1]: PACELC --- if there is a partition (P) how does the system tradeoff between availability and consistency (A and C); else (E) when the system is running as normal in the absence of partitions, how does the system tradeoff between latency (L) and consistency (C)?


Genomu
------

[Genomu](genomu.com) is eventually consistent key value storage database we have developed in order to implement betting system. This system follows the Event Sourcing model. Event Sourcing ensures that all changes to application state are stored as a sequence of events. In order to get current state all events need to be replayed.


## Conflict detection and resolution

In any system where concurrent updates are possible eventually it is going to be a case when two replicas would contain different values for the same key. There are just a few techniques to detect conflicts

  * timestamps
  * vector clocks
  * global sequence number
  * interval tree clocks[^2]

Resolution of conflicts is usually done using one of the following approaches

  * last writer wins
  * allow a user's application to resolve conflict

Allowing an application to resolve conflicts is more flexible because some of the data have a merge-able nature. For example non unique list of values (set). It is trivial to merge this kind of data.

Conflicts are usually resolved on

  * read
  * write
  * schedule

For genomu we decided to pick following properties

  * detect conflicts using ITC
  * resolve conflicts automatically (for some types of updates)
  * allow a user's application to resolve conflicts (for some types of updates)
  * let the user's application to decide when to merge conflicts (on reads or writes)
  * implement automatic resolution on writes for merge-able data (using user provided callback)

[^2]: Interval Tree Clocks (ITC) is a new clock mechanism that can be used in scenarios with a dynamic number of participants, allowing a completely decentralized creation of processes/replicas without need for global identifiers.


## Leader election or lack thereof

Since genomu needs full consistency for payouts we need ACID transactions. ACID transactions in distributed systems are implemented using a transaction coordinator (sometimes called transaction manager or a leader). Usually for leader election in distributed systems Paxos [^3] protocol is used. Leader election is needed if it is not clear which of the nodes would be responsible for coordination of a transaction. It turned out that leader election is not needed if the client is able to maintain simultaneous connections to multiple nodes. Here is how it works. We use round-robin load balancer in front of our backend. Client application creates a pool of connections to backend. Load balancer makes sure that connections from the same user are opened to different backend nodes. When the application wants to initiate a transaction it checkouts one of the connections from the pool and creates a `channel`. Every transaction executes in a `channel` context. When a transaction is committed or rolled back `channel` get destroyed. When an application creates a `channel` the node which receives the request would start a new process responsible for the `channel`. This process would become a `leader` for a transaction. If transaction fails for any reason application would need to try again and check out different connection from its pool. Which would be for a different node. Which would mean that transaction leader would be on different node (which might be on a different side of a split brain).

[^3]: Paxos is a family of protocols for solving consensus in a network of unreliable processors.


## Isolation

The isolation property means that the effects of the transaction shouldn't be visible to other concurrently executed transactions until the transaction is committed. We achieve isolation by fetching revision IDs for all keys which are read or updated inside the transaction and storing them in `ets` table owned by `channel`. Then we prepare a transaction. Prepared transaction contains a next revision id of the key we want to update as well as an operation which needs to be applied to a value. We choose nodes where we want to store N replicas of the data. We send prepared transaction to every replica and wait for W acknowledges. When node containing a replica receives a transaction it is able to tell if the revision id has been changed or not. If it does change node replies with NACK (negative acknowledgement) containing latest revision id it knows about. Updates for a key a placed under a composite key {key, revision} which provides an isolation. When a node commits transaction. It copies the value of {key, revision} under {key, nil} composite key.


## Implementation details

We used riak-core as our distribution framework. Instead of sharding, riak-core evenly distributes data across a cluster using consistent hashing. Riak core provides deterministic routing to replicas of a key. It handles automatic data handovers when a node leaves/joins the cluster. It uses the gossip protocol to ensure consistent topology changes. We implemented a riak-core vnode (virtual node) which receives commands from our transaction coordinator.

Genomu has a clean distinction between `set value` and `update value`. Set value overwrites the old value so it requires full consistency. We let a user's application to resolve conflicts in this case. For `update value` we only allow predefined set of operations. Due to a limited set of update operations, Event Sourcing and use of ITC we can merge conflicts automatically.
