
# Problem

In current implementations of LN, a multi-hop payment requires that
senders and payment forwarding nodes wait for confirmations from
downstream hops before confirming the payment. The management of
confirmations over hops is vulnerable how miners pick transactions for
confirmations.

Transaction selection is part of the environment that LN protocols
don't have control over, and this results in complicated work arounds
in how we forward and confirm HTLCs.

We need to be resilient to how transactions are confirmed by
miners. As long as miners are free to choose which transactions they
confirm, we can not depend on the policy of transaction selection.

We propose a multi-party interactive protocol that eliminates the
dependence on transaction selection by miners. Transactions of all
hops become confirmable or none can ever be confirmed. There is no
need for HTLCs to rollback one hop at a time.

# Context

LN punishment payments depend on mempool for two things:

1. Punishment transactions to be confirmed before a breach transaction
   is confirmed.
2. HTLCs to be confirmed downstream from a hop before the upstream hop
   HTLC can be confirmed.
   
The problem is that without solutions like eltoo that change the
behaviour of mempool, we can't depend on what transactions miners will
pick to confirm first.

We don't focus on solving all mempool issues. However, we try and
reduce the exposure to mempool for multi-hop payments.

# Proposal Overview

We focus on fixing the multi-hop HTLC problem and the solution is to
force the multihop payment to be atomic. Currently a HTLC
confirmations are passed back along a path, one hop at a time. This
leaves the payment path vulnerable during this "peel back" of the
HTLC.

We propose that all parties along a path co-operate in an interactive
protocol to simultaneously change their channel hop balances. That is,
either all parties have the new states for their channel commitment
transactions if the payment succeeds or the state of no party is
changed.

## Commitment transaction states

As per the LN Penalty channel construction, we create a commitment
transaction that is asymmetric for attribution, delayed and revocable
to allow a party to broadcast a breach remedy transaction. We need to
keep the breach remedy transaction approach for two peers of a channel
to keep each other honest. What we want avoid is creating a multi-hop
dependence on commitment transactions.

When forwarding payments, we diverge from the approach of HTLC
forwarding. The forwarding of HTLCs creaties dependencies on timing
and mempool transaction confirmation. We suggest an alternative
protocol that forces all parties along a payment to commit to new
states corresponding to a payment succeeding or failing along the
entire path. In case of failure, the new state is abandoned by all
parties. If any party does broadcast and confirm the abandoned state,
we propose a punishment protocol that splits the cheating party's
balance with all the parties involved in the payment.

## Protocol

### Setup

1. Funding tx is created as normal for each channel.
2. Commitment transaction between channel parties is created as normal
   with outputs that encumber payment to self with CSV and leave the
   remote output unencumbered.

### Payment

Say we have a two hop payment path Alice -> Bob -> Carol for 1
bitcoin.

Say the amounts in the commitment transactions are as follows:

```
Alice (10) == Bob (0)
Bob (10) == Carol (0)
```

When Carol requests a payment from Alice, we assume Alice has the
routing information and knows the path to pay along is through Bob.

We introduce the notion of a locked amount for the payment. Each hope
locks a certain amount of liquidity while a payment is being
processed. We write the locked amount after the party's balance. For
example, below all parties have not locked any liquidity.

```
Alice (10, 0) == Bob (0, 0)
Bob (10, 0) == Carol (0, 0)
```

To make the payment:

1. Alice sends a request to Bob that Alice needs access to Bob ->
   Carol liquidity of 1 bitcoin. 
2. In response to receiving the above request Bob locks 1 bitcoin
   worth of liquidity on Bob -> Carol for a small time period (say 10
   seconds). Note the emphasis on short lock periods.
3. The balances along the channel with the latest commitment transactions look like:
	```
	Alice (10, 1) == Bob (0, 0)
	Bob (10, 1) == Carol (0, 0)
	```
4. Alice, Bob and Carol then run a key aggregation algorithm to generate a
   public group key K_{ABC}.
5. All channel participants then generate new commitment transactions
   to pay to this group key K_{ABC}. This payment is made asymmetric,
   delayed and revocable just as normal LN commitment transactions.
   1. We need to evaluate the limits on the path length based on the number of
      combinations we need to support here. A path length limit of 3-5 for a
      connected graph will also encourage increased network connectivity
6. All nodes create new commitment transactions spending into a 
   
   All the nodes run a signature aggregation protocol.
7. Once the signature aggregation protocol has completed all nodes now
   have new


### Communication and privacy

In the above we do not delve into how the communication between nodes
is achieved.

It can be over unicast messages wrapped in onion layers and forwarded
one hop at a time, or it can be direct p2p messages from the sender to
all other nodes on the path.

The protocol outlined above does break the onion routing and privacy
goals of the current LN protocol goals. These are traded off for
reliable and atomic payments.
