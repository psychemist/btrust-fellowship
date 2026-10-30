# Beyond HTLCs: Point Time-Locked Contracts and Cycle Jamming in Lightning

## 1. Introduction

The Lightning Network enables Bitcoin payments via a network of interconnected payment channels. Each payment channel is a connection between two Lightning nodes so the network is formed from thousands of these connections arranged in a graph-like structure.

Lightning currently uses Hash Time-Locked Contracts (HTLCs) to ensure that intermediary hops along payment routes can forward payments from a seder to the receiver without taking on the risk of paying their next hop and failing to collect from their previous one.

HTLCs work fine but their design has some limitations, particularly around payment privacy and the conditions that can be attached to a payment. Point Time-Locked Contracts (PTLCs) offer an alternative by replacing hash-timed locks with elliptic-curve point locks and adaptor signatures.

PTLCs improves on payment privacy as it makes it harder to link different payments along the route or construct the entire route using the single hop payments. They also allow for more expressive conditional payments. and change the information that routing nodes can use to identify malicious payments.

One interesting consequence of this is **cycle jamming**, where an attacker deliberately routes a payment through the same channels repeatedly to immobilize liquidity. preventing the funds from being used to settle payments and leading to perceived network outages.

## 2. How Lightning Payments Work

Let us consider a payment routed through these nodes:

```text
Alice → Bob → Carol → Dave
```

With an HTLC, Dave generates a secret preimage `r` and its hash:

$$
h = H(r)
$$

Every hop presents the same hash as the condition for receiving its funds. Once Dave reveals `r` to Carol, Carol can claim her incoming payment from Bob, who can then claim from Alice. The secret travels backward along the route. 

The timed locks are staggered to ensure that the intermediaries (Bob and Carol) have time to claim their incoming payments, or recover funds if the payment fails. In a nutshell, Alice's HTLC expires after Bob's and Carol's expires before Bob's.

The shared hash is useful for instant, atomic settlement, but it also acts as an identifier for all the hops along the same route. Two intermediaries that observe the same hash can infer that they may be forwarding the same payment.

## 3. From Hashes to Elliptic-Curve Points

PTLCs use a property of elliptic-curve cryptography that ordinary hashes do not have.

Given a private scalar `a` and the curve's generator point `G`, the corresponding public point is:

$$
A = aG
$$

Points can be added:

$$
aG + bG = (a+b)G
$$

Ordinary cryptographic hashes do not have an equivalent additive relationship. In general, $H(a+b)$ cannot be derived by adding $H(a)$ and $H(b)$.

This is relevant here because we can construct related payment conditions by adding secret offsets. For example, Bob might receive a payment locked to $P+yG$ while forwarding one locked to $P$. Once he learns the scalar corresponding to $P$, he adds `y` to obtain the scalar for his own contract.

These are point locks and while they differ, the contracts remain linked through the underlying mathematics. This is the basis of PTLCs' per-hop randomization and many other applications.

## 4. Adaptor Signatures

Point arithmetic gives us secrets related by scalar addition. **Adaptor signatures** let us enforce a payment condition using those secrets.

An adaptor signature is an incomplete signature that can be completed with a particular secret. More interestingly, someone who holds the incomplete signature can extract that secret after seeing the completed signature.

In a simplified Schnorr adaptor construction, the incomplete signature scalar $\tilde{s}$ and completed signature scalar $s$ are related by this equation:

$$
s = \tilde{s} + t \pmod n
$$

where `t` is the secret used to complete the signature. 

Someone holding both values can calculate:

$$
t = s - \tilde{s} \pmod n
$$

In a multi-hop PTLC payment, the receiver completes the downstream conditional signature. The preceding node extracts the secret, adjusts it using its own offset, and completes the upstream payment. This continues backward through the route till the first intermediary after the sender.

Actual adaptor-signature protocols require careful nonce handling and verification of the adaptor data, but I hope the main idea is clear: completing a conditional payment reveals the information needed to settle the previous hop.

## 5. Payment Privacy and Unlinkability

With an ordinary HTLC, the same payment hash appears across the route:

$$
h,\ h,\ h
$$

Randomized PTLCs allow different hops to use distinct but cryptographically related points:

$$
P_1,\ P_2,\ P_3
$$

Intermediaries working together can compare HTLC hashes to link payments they have both seen. With randomized PTLCs, comparing point locks no longer works in the same way.

This makes it harder to reconstruct full payment paths or associate senders with recipients. It does not eliminate other privacy leaks. Therefore, data like amounts, timing, network topology, and adjacent-hop information can still be useful to an observer.

The privacy improvement has an interesting security consequence which we will discuss next.

## 6. Cycle Jamming: The Availability Attack

A pending Lightning payment reserves outgoing channel liquidity until it either settles or fails. While that transfer is unresolved, the reserved funds cannot be used for another payment in the same direction. Channels also have limits on how many pending HTLCs they can hold.

Attackers can exploit both resources:

- **Liquidity jamming** reserves large amounts of directional balance, preventing other payments from using it.

- **HTLC-slot jamming** fills pending-contract slots with many small unresolved payments, even when the channel still has liquidity, preventing other payments from passing via this channel.

Cycle jamming amplifies the first problem. Instead of crossing a target channel once, an attacker constructs a payment route that crosses it repeatedly:

```text
Attacker → A → B → C → A → B → C → A → B → Attacker
```

Here, the channel `A → B` is traversed three times. If the payment amount is 100,000 sats, each traversal creates a separate pending conditional transfer. Without accounting for fees, approximately **300,000 sats** are reserved in the `A → B` direction. The other channels in the cycle are affected too: `B → C` and `C → A` each have approximately 200,000 sats resserved.

For this to work, the attacker would also control or work with the receiving node of the payment and withhold settlement, so those funds are reserved until the contracts fail, settle, or time out. The attack relies on a known payment route, sufficient directional liquidity in the channel, and valid hop & time lock limits.

The important distinction is between the **payment amount** and the **total liquidity reserved across traversals**. A single payment can immobilize several times its own amount on a channel. That does not necessarily mean the attacker's entire capital is equal to the payment amount. In fact, this typeof attack allows the attacker to reserve far more liquidity than they own.

## 8. Why PTLCs Make Cycle Detection Harder

Cycle jamming has existed before PTLCs. The PTLC-specific problem is that the point randomization removes a simple way to recognize repeated appearances of the same payment. A shared payment identifier like the hash `h` used by HTLCs is useful for recognizing when the same payment comes back through a node.

With HTLCs, a node might observe:

```text
First visit: h

Second visit: h

Third visit: h
```

Comparing the hashes gives it evidence that the payment has returned.

With randomized PTLCs, it would instead observe:

```text
First visit: P₁

Second visit: P₄

Third visit: P₇
```

The point locks are different. Any equality checks of the hash no longer identify the repeated payment.

Immediate reversals or one-step reversals such as `A → B → A` are easier to detect because node B knows that its next hop is also its previous hop. Longer cycles such as `A → B → C → A` are harder: each intermediary sees different neighboring nodes, while onion routing conceals the full path.

The same unlinkability that improves payment privacy removes a useful signal for detecting malicious routing cycles. That is the core problem described in the Bitcoin Problems entry on PTLC cycle jamming.

## 9. Proposed Mitigations

There are several approaches, each with tradeoffs. A couple are discussed below:

- **Reject immediate reversals.** A node can refuse to forward a payment straight back to the peer it came from. This is simple and works with PTLCs, but longer cycles remain possible.

- **Restrict route lengths and time-locks.** Shorter routes reduce how many times a payment can traverse a target channel. Tighter expiry policies can limit how long liquidity remains locked. Both measures can also reduce legitimate payment reliability.

- **Price reserved resources.** If forwarding nodes could charge for the liquidity and time reserved for unresolved payments, jamming would become more expensive. The difficult part is handling punishment when a payment fails for legitimate reasons.

- **Use reputation and resource allocation.** Nodes could allocate scarce forwarding capacity depending on how reliably neighboring peers resolve payments. Hybrid channel-jamming mitigation research explores this approach but it faces attribution and reputation-manipulation problems.

- **Prove that a route is cycle-free.** The Bitcoin Problems entry proposes investigating randomizable commitments and cryptographic proofs that confirm a route contains no repeated nodes without revealing the route itself. For a route $R=(N_0,N_1,\ldots,N_k)$, the desired property is:

$$
\forall i \ne j,\quad N_i \ne N_j
$$

	A zero-knowledge proof might establish that property while hiding the underlying node identities. But a valid construction must also bind the proof to the *actual onion route being executed*. Otherwise, an attacker could prove that a harmless route is cycle-free and send the payment through a different route. The commitments must also avoid becoming new identifiers which could be used to correlate payments along the route.
	
	This remains an open design direction in the referenced problem statement, rather than a complete deployed feature. Even a successful cycle-free proof would not eliminate ordinary jamming by payments that never loop.

## 10. What Else PTLCs Enable

The interesting thing about point time-locks is that we can combine secrets algebraically. That permits payment conditions beyond merely knowing a hash pre-image.

### Stuckless Payments

A payment can become stuck while the sender waits to learn whether a route will complete. Retrying through another route can risk duplicate payment if both attempts eventually settle.

A proposed PTLC construction lets the sender withhold part of the unlocking scalar until the receiver acknowledges that an attempt has arrived. For example, a payment might be locked to:

$$
(x+y+z)G
$$

The receiver initially knows only `z`. After acknowledgment, the sender releases `x + y`, allowing the receiver to complete the conditional payment. An upstream intermediary holding offset `y` can then recover the scalar it needs by subtracting `y`.

With the right protocol safeguards, this gives the sender greater control over which of several attempted routes actually settles. The algebra enables the construction but preventing duplicate settlement still depends on the complete protocol.

### Escrow Contracts

Let's say Alice is buying something from Bob, with Carol available as a referee if they dispute delivery.

Alice and Carol can derive a shared scalar `u` through Elliptic-Curve Diffie-Hellman (ECDH). Bob has private scalar `b` and public point $B=bG$. Alice creates a conditional payment locked to:

$$
P=B+uG
$$

Bob needs `b + u` to settle. Alice can provide `u` when satisfied, or Erin can provide it according to the agreed dispute procedure. The referee does not need to participate when the transaction goes smoothly.

A real escrow protocol still needs clear rules for timeout scenarios and an enforceable agreement about when the referee should release the secret.

### Selling Signatures

PTLCs can make payments valid only when a valid Schnorr signature is supplied for a particular message.

In the a Schnorr signature scalar:

$$
s=k+ep
$$

with public nonce point $R=kG$ and public key $P=pG$, the verification equation is:

$$
sG=R+eP
$$

The buyer can construct the payment lock from the public expression $R+eP$. To satisfy it, the seller must provide the scalar `s`, which is the signature component the buyer wants.

This makes it possible to exchange a Lightning payment for specific cryptographic information, with potential applications in eCash and other credential systems.

## 12. Conclusion

PTLCs improve Lightning's conditional-payment mechanism by replacing hash-timed locks with elliptic-curve points. The algebra makes the different hop payments along a route unlinkable possible and supports constructions like stuckless payments, escrow contracts, and selling signatures.

Cycle jamming shows why the security implications of PTLCs deserve just as much attention. An HTLC's shared hash can help a node recognize that a payment has returned through a cycle. Randomized PTLCs remove that check, making longer cycles harder to detect with local information alone.

Routing restrictions, resource pricing, and reputation systems can reduce the damage. A private proof that a route contains no cycles addresses the detection problem more directly, but the construction still has to preserve unlinkability and prove that the route actually executed.

What I find most interesting is how one low-level mathematical change expands the kinds of contracts Lightning can support while forcing us to recheck assumptions built into its routing security. However, the cryptography is only part of the design problem; its consequences for the network also matter.

As of October 2026, PTLCs are in the research and development phase and remain an undeployed upgrade to standard Lightning payments, while adaptor signatures are already used in specialized Bitcoin protocols. The Taproot upgrade, which went live on Bitcoin in November 2021, provides important foundations like channels and Musig2, but the required PTLC-specific tooling, specifications, network primitives, and security design remain incomplete.

## References

1. Pérez-Solà et al., [*LockDown: Balance Availability Attack Against Lightning Network Channels*](https://eprint.iacr.org/2019/1149.pdf)
2. Bitcoin Problems, [*PTLC Cycle Jamming*](https://bitcoinproblems.org/problems/ptlc-cycle-jamming.html)
3. Bitcoin Optech, [*Point Time-Locked Contracts*](https://bitcoinops.org/en/topics/ptlc/)
4. Bitcoin Optech, [*Adaptor Signatures*](https://bitcoinops.org/en/topics/adaptor-signatures/
5. Blockstream Research, [*Multi-Hop Locks from Scriptless Scripts*](https://github.com/BlockstreamResearch/scriptless-scripts/blob/master/md/multi-hop-locks.md)
6. Nadav Kohen, [*Replacing Payment Hashes with Payment Points*](https://www.kouloumos.com/bitcointranscripts.github.io/lightning-conference/2019/2019-10-20-nadav-kohen-payment-points/)
7. Bitcoin Optech, [*Channel Jamming Attacks*](https://bitcoinops.org/en/topics/channel-jamming-attacks/)
8. Lightning BOLTs, [*Hybrid Channel Jamming Mitigation*](https://github.com/lightning/bolts/issues/1218)
