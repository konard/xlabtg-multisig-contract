# Security Audit — TON Multisig Wallet (`multisig-code.fc`)

> Audit performed in response to xlabtg/multisig-contract#1
> ([TON Bug Bounty scope](https://github.com/ton-blockchain/bug-bounty)).

## Scope & artifacts

| Item | Value |
|------|-------|
| Audited file | `multisig-code.fc` |
| Repo commit | `0ac5314` |
| `sha256(multisig-code.fc)` | `efe263be776d04201443b4c8731c8e66139c65d07463e50e4af9a44dc9f263b1` |
| Supporting files | `scheme.tlb`, `stdlib.fc`, `*.fif` interaction scripts |

**Provenance note.** `multisig-code.fc` in this repository is **byte-for-byte identical** to the
canonical `ton-blockchain/multisig-contract` / TON monorepo
(`crypto/smartcont/multisig-code.fc`). This was verified by diffing against upstream `master`
(0 differences). Therefore this audit covers the genuine, widely-deployed TON multisig contract,
and the findings below are properties of that upstream code, not of a modified fork.

## Methodology

Following the phases requested in the issue:

1. **Code comprehension** — mapped storage layout (`scheme.tlb`), the two entry points
   (`recv_external`, `recv_internal`), all get-methods, and the order/voting lifecycle.
2. **Threat modeling** — trusted parties are the `n` key holders (any `k` of whom form a quorum);
   untrusted parties are everyone else (who cannot produce a valid owner signature). Critical assets
   are the contract's TON balance and its `pending_queries` / `owner_infos` state.
3. **Vulnerability analysis** — per-function review of authorization, signature verification,
   replay protection, arithmetic, gas, and state consistency.
4. **Validation** — each finding is tied to concrete line numbers and an attack scenario; claims that
   did **not** hold up under analysis are listed explicitly in
   [§ Verified-safe properties](#verified-safe-properties-non-findings) to avoid false positives.

---

## Summary of findings

| # | Severity | Title | Type |
|---|----------|-------|------|
| [F-1](#f-1) | **Medium–High** | A single compromised/malicious owner key can drain the wallet balance through fees and spam on-chain state | Economic DoS / griefing (CWE-400 / CWE-799) |
| [F-2](#f-2) | **Medium** | Unbounded `pending_queries` growth exhausts the external-message gas credit and bricks the wallet | DoS (CWE-400 / CWE-770) |
| [F-3](#f-3) | **Low** | Co-signatures are not bound to the contract address; cross-wallet signature reuse when two multisigs share `wallet_id` | Signature scope (CWE-347 / CWE-294) |
| [F-4](#f-4) | **Low / Info** | `recv_internal` silently ignores every internal message, including bounces | Bounce/error handling (CWE-703) |
| [F-5](#f-5) | **Low / Info** | No on-chain validation that owner indices are `< n`; a misconfigured deployment can permanently brick the wallet | Access/config (CWE-20) |
| [F-6](#f-6) | **Info** | Order expiry depends on `now()` (block timestamp), giving validators a small timing influence | Time dependence (CWE-829) |

> **No signature-bypass or direct fund-theft vulnerability was found.** The quorum is genuinely
> enforced: outgoing messages are emitted only when `cnt >= k` distinct valid owner signatures have
> been collected (`update_pending_queries`, line 86). Every path that reaches `send_raw_message`
> passes through that check. See [§ Verified-safe properties](#verified-safe-properties-non-findings).

---

<a name="f-1"></a>
## F-1 — A single owner key can drain the balance via fees / spam state

**Severity:** Medium–High
**CVSS 3.1:** 6.5 — `AV:N/AC:L/PR:H/UI:N/S:U/C:N/I:L/A:H`
**CWE:** CWE-400 (Uncontrolled Resource Consumption), CWE-799 (Improper Control of Interaction Frequency)
**Affected function:** `recv_external` (`multisig-code.fc:116`)
**Affected lines:** 148–171, 183
**Corresponds to:** [ton-blockchain/ton#168](https://github.com/ton-blockchain/ton/issues/168)

### Description

The multisig accepts and *pays for* an external message as soon as **one** valid owner signature is
present, long before a quorum is reached. On-chain voting is a deliberate feature (an under-quorum
order is stored so the remaining signatures can be added later), but it means the cost of the "voting"
is borne entirely by the multisig's own balance and can be triggered unilaterally by any single key
holder:

* `set_gas_limit(100000)` (line 164) accepts the message (equivalent to `accept_message` with a cap),
  so the contract pays for gas even when `cnt < k`.
* The under-quorum order is persisted to `pending_queries` (lines 95–100 via
  `update_pending_queries`) and `commit()`ed (line 173), consuming storage the contract also pays rent
  for.
* `accept_message()` at line 183 additionally pays for the cleanup pass.

An owner (or an attacker who has compromised **one** of the `n` keys) can therefore repeatedly submit
distinct never-to-be-completed orders, each of which costs the contract gas + storage, steadily
draining the balance without ever obtaining a quorum. This defeats the intuitive expectation that a
single leaked key is harmless in a `k`-of-`n` wallet.

### Root cause

The contract couples "collect a vote" with "spend the wallet's gas". Because a valid external message
requires only one owner signature (lines 130–133), the frequency of paid operations is controlled by
any single holder.

### Existing mitigations (why this is Medium, not Critical)

The upstream authors already added partial rate-limiting; a fully honest assessment must credit them:

* **Per-owner flood cap** — each owner may have at most 10 in-flight proposals
  (`flood += 1; throw_if(39, flood > 10)`, lines 148–151), decremented on completion/expiry.
* **Minimum lifetime for solo proposals** — a *new* order with `cnt < k` is rejected unless its expiry
  is more than one hour in the future (`throw_if(41, …)`, line 162; documented in `README.md` "Dev
  History" #5). This blocks the "infinitely re-submit the same cheap order" variant.

These bound the drain *rate* but do not remove it: slots free up as orders expire (~hours), so a
patient malicious key still bleeds the balance and pollutes state indefinitely. This is why it is a
real availability/economic issue rather than a non-issue.

### Impact

* **Financial:** gradual loss of the wallet's balance to network fees (no theft to an attacker address).
* **Operational:** `pending_queries` fills with junk, which directly feeds F-2 (gas-credit DoS).

### Proof-of-concept (scenario)

```
Preconditions: attacker controls exactly one owner key at index j (0 <= j < n), k >= 2.

Loop forever:
  1. Build 10 orders with distinct query_ids (distinct seqno / timestamp),
     each with a benign message, expiry > now + 1h  (to pass throw_if 41).
  2. Sign each only with key j (root signature), no co-signatures.
  3. Send all 10 as external messages.
     -> each is accepted (set_gas_limit at line 164), stored, committed;
        contract pays gas+rent; flood[j] reaches 10.
  4. Wait for the orders to expire; flood[j] is decremented during cleanup.
  5. Go to 1.

Result: the wallet's balance is continuously spent on fees for orders that
        can never reach quorum, and no other owner authorised any of it.
```

An automated sandbox PoC would deploy the contract with `create_init_state`, send the crafted external
messages via TON Sandbox / `toncli`, and assert the balance strictly decreases while no
`send_raw_message` is emitted.

### Remediation

This is an inherent trade-off of the on-chain-voting design. Options, in order of robustness:

1. **Prefer the audited successor.** Use
   [`ton-blockchain/multisig-contract-v2`](https://github.com/ton-blockchain/multisig-contract-v2),
   whose design routes proposals through signer wallets (internal messages), so the *proposer* pays
   for creating an order rather than the multisig.
2. **Tighten the flood cap / raise the minimum lifetime** relative to the expected legitimate order
   rate, and consider a global cap on `pending_queries` count, not just per-owner.
3. **Operational:** treat a single leaked key as an incident (rotate keys / migrate funds) — the `k`-of-`n`
   guarantee protects against *theft* but not against a griefing holder.

---

<a name="f-2"></a>
## F-2 — Unbounded `pending_queries` exhausts the gas credit and bricks the wallet

**Severity:** Medium
**CVSS 3.1:** 5.9 — `AV:N/AC:H/PR:H/UI:N/S:U/C:N/I:N/A:H`
**CWE:** CWE-400 / CWE-770 (Allocation of Resources Without Limits)
**Affected function:** `recv_external` (`unpack_state` line 79/127, cleanup loop lines 186–200)
**Affected lines:** 127, 143–144, 186–200

### Description

External messages on TON are processed under a **fixed gas credit** (~10k gas) *before*
`accept_message`/`set_gas_limit` is reached. `recv_external` must, before line 164, run `unpack_state`
(which parses the whole storage cell, including the `pending_queries` `HashmapE`) and several dict
lookups. As `pending_queries` grows, the cost of touching it grows with it. Once the pre-acceptance
work exceeds the gas credit, **every** external message aborts before acceptance — including
legitimate ones — and the wallet can no longer process any order. `README.md` documents this as the
~100-non-expired-orders limit ("out of gas credit exception").

The cleanup loop (lines 186–200) only removes records that expired **more than 64 seconds ago**
(`bound -= 64 << 32`), and removes them lazily one-per-message, so a burst of orders with long/staggered
expiries can outpace cleanup.

### Root cause

There is no hard upper bound on the number of simultaneously-stored orders; the per-message gas budget
is effectively bounded by the size of `pending_queries`, which is attacker-influenceable (see F-1).

### Impact

Temporary-to-persistent denial of service: while the dict is oversized the wallet is unusable. It
recovers only as records age out, which requires messages to be processed — the very thing that is
failing. Combined with F-1, one key holder can push it over the edge cheaply.

### Proof-of-concept (scenario)

Using F-1's loop, keep `pending_queries` near its maximum (≈100 live orders with far-future expiry).
While in that state, a legitimate owner's external message runs out of gas credit during
`unpack_state` and is rejected, i.e. the wallet is bricked until orders expire.

### Remediation

* Cap the number of live orders (reject new proposals when `pending_queries` is at capacity) so the
  pre-acceptance cost stays bounded.
* Store a small count/index alongside the dict so size checks don't require a full scan.
* Prefer `multisig-contract-v2`, which does not persist unbounded pending-order state on the multisig
  itself.
* **Operationally:** heed the README warning — keep well under ~100 non-expired orders.

---

<a name="f-3"></a>
## F-3 — Co-signatures are not bound to the contract; cross-wallet reuse under shared `wallet_id`

**Severity:** Low
**CVSS 3.1:** 3.7 — `AV:N/AC:H/PR:N/UI:N/S:U/C:N/I:L/A:N`
**CWE:** CWE-347 (Improper Verification of Cryptographic Signature) / CWE-294 (Replay)
**Affected function:** `recv_external` (lines 135–159), `check_signatures` (lines 31–53)
**Affected lines:** 137, 143

### Description

The message a **co-signer** signs is `hash = slice_hash(in_msg)` taken *after* the signature dict is
consumed (line 137), i.e. it covers only `(query_wallet_id, query_id, toSend)`. It does **not** include
the multisig's own address or its public-key set. Consequently, a co-signature collected for an order
on multisig **A** is also a cryptographically valid co-signature for a byte-identical order
(same `wallet_id`, same `query_id`, same messages) on a different multisig **B**, provided the same key
holder is an owner of both wallets.

The intended isolation control is `wallet_id` (checked at line 139), so this only bites when two
multisigs are deployed with the **same** `wallet_id` and share at least one owner — an unusual but not
impossible configuration.

Note the root signature *does* transitively bind more (its `root_hash`, line 124, is computed before
`root_i` and the signature dict are consumed, so it covers them), but it still does not include the
contract address either.

### Impact

Limited: a shared-`wallet_id` deployment could see a co-signer's vote for order X on wallet A be
replayed as a vote for the same order X on wallet B without the co-signer's intent. It cannot forge a
signature that was never produced, and it still cannot bypass the quorum on either wallet.

### Remediation

* Always deploy each multisig with a **unique** `wallet_id`.
* For a code-level fix, include the contract address (or the `owner_infos` hash) in the signed
  preimage, so signatures are non-transferable across wallets.

---

<a name="f-4"></a>
## F-4 — `recv_internal` ignores all internal messages, including bounces

**Severity:** Low / Informational
**CWE:** CWE-703 (Improper Check or Handling of Exceptional Conditions)
**Affected function:** `recv_internal` (`multisig-code.fc:55`)

### Description

```func
() recv_internal(slice in_msg) impure {
  ;; do nothing for internal messages
}
```

The contract accepts any incoming internal message and does nothing. In particular, a message the
multisig sent out (after an order reached quorum) that later **bounces** returns its value to the
contract but is silently dropped: the order is already marked processed (inactive), so it is neither
retried nor flagged. It also does not filter on the bounce flag.

### Impact

No fund loss (bounced value is credited back to the balance), but there is **no observability**: an
order that failed at the destination looks identical on-chain to one that succeeded. Off-chain tooling
must reconcile results independently. This is acceptable for a wallet but worth documenting.

### Remediation

Optionally short-circuit bounced messages explicitly (`in_msg` flags bit 0) and/or emit a log, so
failed sends are distinguishable. No change is required for safety.

---

<a name="f-5"></a>
## F-5 — No on-chain check that owner indices are `< n`

**Severity:** Low / Informational
**CWE:** CWE-20 (Improper Input Validation)
**Affected function:** `create_init_state` (line 229), `pack_state` / storage layout
**Affected lines:** 99 (`store_uint(cnt_bits, n)`), 229–231

### Description

`cnt_bits` is stored as an `n`-bit field (line 99) and votes set bit `1 << i` where `i` is the owner
index. The contract assumes owners occupy indices `0 … n-1` (which the `new-multisig.fif` deploy script
does enforce — it assigns indices sequentially from 0, and checks `k <= n`). But the *contract itself*
never validates this: `create_init_state` stores whatever `owners_info` dict it is handed.

If a deployment places an owner at index `i >= n` (a bespoke/buggy deploy tool), then the first time
that owner votes, `store_uint(cnt_bits, n)` is asked to store a value `>= 2^n` in an `n`-bit field,
which throws a cell-overflow exception in `pack_state` and makes the affected order permanently
unprocessable — potentially bricking the wallet for that index.

### Impact

Deployment-time footgun only; not remotely triggerable and not exploitable against a correctly
deployed wallet. Included for completeness.

### Remediation

Document the invariant "owner indices must be `0 … n-1`", and/or add a guard in `create_init_state`
that iterates `owners_info` and asserts every key is `< n`.

---

<a name="f-6"></a>
## F-6 — Order expiry relies on `now()` (block timestamp)

**Severity:** Informational
**CWE:** CWE-829 (Reliance on Untrusted Time Source, in the mild TVM sense)
**Affected lines:** 153–154, 162, 184

### Description

Expiry and replay windows are derived from `now()` (`bound = now() << 32`). Block producers have the
usual small latitude over the block timestamp. This is a standard, accepted TON pattern and is not a
practical vulnerability; noted only because the issue checklist asks about time/seqno handling. The
replay protection itself is sound (see below).

---

<a name="verified-safe-properties-non-findings"></a>
## Verified-safe properties (non-findings)

To avoid false positives, these were specifically checked and found **correct**:

* **Quorum is enforced.** `send_raw_message` is reached only inside `update_pending_queries` under
  `if (cnt >= k)` (lines 86–90). No other path emits outgoing messages. A single or sub-quorum order
  never sends funds.
* **No double-counting of signatures.** `check_signatures` de-duplicates by `cnt_bits`
  (`should_check = cnt_bits != old_cnt_bits`, lines 46–49), and the root signer's own bit is rejected
  if already set (`throw_if(34, cnt_bits & mask)`, line 158). Reaching `cnt >= k` genuinely requires
  `k` *distinct* valid owner signatures.
* **Every external message needs an owner signature.** Non-empty external messages must carry a valid
  root signature from a known owner (`throw_unless(31, found?)` / `throw_unless(32, check_signature…)`,
  lines 130–133). Anonymous attackers cannot even spam.
* **Replay protection is sound.** Processed orders are stored inactive and re-submission throws
  `35`; after cleanup, `query_id < now()<<32` throws `33` (lines 154, 60–64, 92). Executed orders
  cannot be replayed.
* **Signature-list size is bounded.** `calc_boc_size` caps the whole message at 8 cells / 2048 bits
  (`throw_if(40, …)`, lines 143–144), so `check_signatures` cannot be forced into an unbounded loop.
* **No arithmetic overflow.** `flood` (8-bit) is capped at 10; `cnt`, `k`, `n` are 8-bit and
  bounded by construction; `query_id` is 64-bit; `1 << i` stays within TVM's 257-bit integers for
  valid indices. No under/overflow leading to fund loss was found.
* **`try_init` is safe.** The empty-message initializer only sets `last_cleaned` once
  (`throw_if(37, last_cleaned)`, line 80); re-initialization is impossible.

---

## Deliverables checklist (from the issue)

- [x] Read all contracts and scripts thoroughly
- [x] Mapped entry points, state, and the order lifecycle
- [x] Threat model (trusted `k`-of-`n` holders vs. everyone else)
- [x] Per-function vulnerability analysis with line references
- [x] Findings reported with severity, CWE/CVSS, root cause, impact, PoC scenario, remediation
- [x] False positives excluded and documented as verified-safe properties
- [x] Confirmed in-scope items for the TON Bug Bounty (no mainnet/testnet exploitation performed)

## References

- TON Bug Bounty scope — https://github.com/ton-blockchain/bug-bounty
- Multisig vulnerability discussion — https://github.com/ton-blockchain/ton/issues/168
- Audited successor design — https://github.com/ton-blockchain/multisig-contract-v2
- Upstream source (identical to this file) — https://github.com/ton-blockchain/ton/blob/master/crypto/smartcont/multisig-code.fc
- "From Paradigm Shift to Audit Rift" (TON audit taxonomy) — https://arxiv.org/abs/2509.10823
- CWE-400, CWE-770, CWE-347, CWE-294, CWE-703, CWE-20 — https://cwe.mitre.org/
