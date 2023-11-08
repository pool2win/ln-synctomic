# Introduction

LN was designed before Schnorr signatures and multi party signature
protocols like MuSig were available on bitcoin.

How can we design LN with these new tools available to us?

What if we can do away with HTLCs completely by using signature
aggregation? If a protocol was possible what would look like and what
will be advantages and disadvantages of such a protocol over the
current HTLC based approach?

This document presents one possible construction that uses MuSig2
between multiple channels across parties to make reliable atomic
payments.

# Goals for any new construction

1. Payments should be *atomic* - either all parties update channel
   state, or none do.
2. Payments should be *reliable* - either the sender receives a success
   or a failure. That is, a sender is never left hanging about the
   status of the payment.
3. Payments should be *synchronous* - payment processing should
   complete in under seven seconds for the sender, or it should fail.
4. Payment should not leak information - i.e. payments should help
   retain sender and receiver privacy - even from parties that forward
   the payment.
   
# Does LN currently provide the above goals?

1. We do get atomicity, but the time bounds are large due to inherent
   time delays in HTLCs.
2. There are some corner cases where payments get stuck, so the
   protocol misses providing reliability. This is caused
   by timing dependencies HTLCs and need to broadcast transactions on
   chain to unroll them in worse case.
3. Failure cases can not finish under seven seconds - again due to
   timelocks.
4. LN can meet privacy challenge, as we move to taproot channels and
   PTLC instead of HTLCs.

The above unsolved problems in lightning come from the use of HTLCs to
forward payments along a path.

Going back to the question we posed at the start: Can we come up with
a construction using MuSig2 that meets our goals above?

# Protocol overview

1. Gossip and path finding algorithms are used as normal to get a list
   of parties a payment will be forwarded through.
2. All parties along the channel along in the path then construct
   transactions spending the channel commitment transaction outputs
   into a construction that guarantees the following:
   1. All parties will update their channel outputs, or no parties
      will.
   2. If a party tries to cheat by publishing an old state, the other
      parties can take all the input amount of that party. [This can
      be later replaced with any other covenant bitcoin adopts in the
      future.]
2. The sender initiates a key and signature algorithm to generate
   signatures for the transactions generated in the last step.
3. Once all parties have signed the transactions, all parties can
   update their channel state.

The protocol does not require any changes to path finding, we still
need to know which nodes can help forward a payment from source to
destination.

Once this path is available, the protocol runs a key and signature
aggregation algorithms to provide atomic state change across all
channels. That is, all parties either change state to capture the
payment along the path or no party changes any state.

In contrast to HTLC based approach, this atomic state change does not
traverse the path unlocking commitment transactions one hop at a
time. Instead all parties immediately have access to latest commitment
transaction states as soon as the MuSig2 signature aggregation
protocol finishes across all parties.

The communication protocol used to run the MuSig2 signature
aggregation algorithm only requires communication using onion packets
and therefore, each party along the path only knows about one upstream
and one downstream party. No party knows who are the senders and
receivers of the payment based on looking at the payment communication
messages.

# Payment processing using signature aggregation

We describe the protocol using a simple two hop example.

Say Alice wants to pay Carol 1 bitcoin through Bob. Say the initial
channel balances are:

```
Alice (10) <-> Bob (5)
Bob (10) <-> Carol (5)
```

We will use this example to describe the protocol.

## Transactions Tree

Before we describe the protocol in the next sections, these are the
types of transactions we will be constructing and signing:

1. FT - Funding Transaction - the usual two party channel funding
   transaction.
2. GCT - Group Commitment Transaction - a commitment transaction that
   can be spent only if signed by all parties in a payment path using
   MuSig2 protocol. The transaction uses all the FTs as inputs.
3. CT - Channel Commitment Transaction - a commitment transaction
   generating outputs for all participants with updated balances.
4. PT - Punishment Transaction - a multi output transaction that
   punishes a cheating party by taking all their funds on their
   balance and distributing them to all other parties in the payment
   path.
5. RK - Revocation Key - The revocation keys of all the rest of the
   parties along a path to punish a party that broadcasts an old GCT.
   
The figure below shows how the transactions are constructed as and we
explain these in the next sections.

<img src="./commit-transactions.png" width="600px" />

## Setup

1. Channels are constructed using the funding transactions as per LN
   BOLTs.
2. A payment route is found using any of the path finding
   algorithms. Say the path is Alice -> Bob -> Carol.
3. MuSig2 nonce setup: Without going into details, we simply say here
   that all the required public keys and nonces are shared between all
   participants in the payment path during this setup phase. We
   describe which combinations of keys and nonces are required in the
   following steps.

## Execution

1. **Key Aggregation**:
   1. Alice initiates the MuSig2 key aggregation protocol between the
      three parties on the payment path. Let's call the resulting key
      the "group key".
   2. All parties also generate MuSig2 aggregate keys for all
      C(n,n-1), i.e. n combinations of parties. These are used for
      discouraging parties from broadcasting old commitment
      transactions as we will see next. We call these keys the
      **revocation keys**.
	  1. The revocation key on party's branch (say A's) is essentially
         the aggregated key of all parties bar A's.
	  2. This combinatorial number of keys limits how big n can be.
2. **Commitment transactions**
   1. All parties create commitment transactions that spend the
      funding transaction output of all the channels. We call these
      transactions the "group commitment transactions" (GCT). This GCT
      is built using the punishment approach for providing covenants
      and this can be simplified once another covenant op code is
      enabled for bitcoin. For the current state of affairs on
      bitcoin, the GCT has the following two outputs:
	  1. One spendable by the group key encumbered by a CSV and a hash
	     that only the party creating the commitment transaction knows
	     the pre-image of.
	  2. The other one is spendable by a revocation key that allows
	     all the other parties in the channel to spend this group
	     commitment transaction if they know the hash in
	     output 1. There is no CSV encumbrance here.
   2. The parties also create a "channel commitment transaction" (CT)
      that spends the first output of the GCT with two outputs similar
      to the LN BOLT commitment transactions, where `to_local` is
      encumbered with a CSV delay and outputs for all other parties
      are not encumbered with CSV.
3. **Signing Commitment Transactions**
   1. All parties share their commitment transaction with all other
	  parties. We follow the broadcast protocol described in the next
	  section for all parties to share GCT and to generate aggregate
	  signatures.
   2. When a party receives a commitment transaction of another party,
	  it verifies the structure out of the outputs, generates its
	  share of the MuSig2 signature follow the broadcast protocol to
	  share their share of the signature.
4. Once a commitment transaction is signed, any party that broadcasts
   an older commitment transaction will lose all their funds to all
   the other members participating in the payment. The distribution of
   the cheating party's funds is in proportion to the channel capacity
   used by each funding party. We use this simple approach in the
   transaction tree diagram above.
   1. With covenant innovation, such a punishment scheme won't be
      required.
   
# Broadcast Protocol Over Onion Routes

The sequence diagram next shows how the parties interact to collect
required nonces and signature contributions. The sequence diagram
shows the simplicity of the protocol.

![object-interaction](./object-interaction.png)

The highlevel protocol can be described as below:

1. Setup A -> B: A sends setup message with it's pubnonces and pubkey material in an
   onion packet to B.
2. Setup B -> C: B sends setup message to C with it's key and nonce
   materials and includes the onion packet it received from A.
3. Setup C -> B: C adds its materials and forwards the rest of the
   onion back to B.
4. Setup B -> A: B finally sends the onion back to A, which has all
   the materials now available.

# Scalability

TODO: Explain better and verify proposed solution

## Multiplex funding outputs

TODO: Explain better and verify proposed solution

FT has multiple A/B outputs for small amounts. Payments consume on a
single output at a time. Therefore a channel can participate in a
number of payments simultaneously.

## Combining funding outputs for large payment amounts

TODO: Explain better and verify proposed solution

Multiple FT A/B outputs can be combined to pay for larger amounts.

# Analysis

## Happy path - All Parties Sign

All parties receive the aggregate signature for the group
key commitment transaction.

In this case, all parties know they can receive their correct amount
by broadcasting the new commitment transaction and the group
commitment transaction for their channel.

Knowing they are assured the can collect their correct balances
anytime by broadcasting the GCT and the CT. The parties can then
settle these commitment transactions by using the updated channel
balances to create new commitment transactions for new payments.

## Happy path - All Parties Sign - But One Cheats

All parties sign the group commitment transaction, but a party tries
to broadcast an older channel commitment transaction. In this case,
any participant can broadcast the punishment transaction for the
CT. This will distribute the cheating party's balance to all other
parties in proportion to their balance.

## Unhappy path - Group Signing Fails or Party Broadcast Old Group Commitment 

In this case nothing changes and parties can create new group
commitment transactions for new payments.

If a party tries broadcasting an older group commitment transaction,
then all other parties can immediately take all the balance of the
cheating party by broadcasting the punishment transaction.


# Advantages

1. We reduce the attack surface where HTLCs are forwarded and unwound
   back along the path.
2. Payments are atomic without additional protocol to confirm payments
   along a path by revealing HTLCs. Either all parties agree or they
   don't agree, no party can stall progress by choosing to losing
   money.
3. Simplicity of commitment transaction construction. There are no
   HTLCs and therefore the complexity of the commitment transactions
   is reduced.

# Disadvantages

1. Gathering signature for commitment transactions requires O(n^2)
   communication messages. The question here is how long are payment
   paths in practice in LN at the moment? This protocol will
   encourages higher connectivity between LN nodes so that payment
   paths remain short.
2. The protocol requires all parties in the path to know about all the
   participants included in the path.

# Questions and Required Work

1. What is the payment length in real world usage? How long a path can
   we support given a target transaction size.
2. Is there a smarter way to replace "revocation keys"?
3. What is the latency to run MuSig2 for a N parties over a p2p
   network? We should assume all parties along a path can talk to each
   other, even if they don't channels to each other.
4. How open are people to an alternative solution that does away with
   onion routing for increase reliability of payments?
5. How much worse will the latency be compared to current protocol?
6. How do we scale this construction so that each channel can
   simultaneously participate in more than one payment? The payments
   will succeed or fail using this construction within much shorter
   time periods - but these latencies have to be worked - and will
   they be enough?
