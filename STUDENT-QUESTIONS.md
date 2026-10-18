# STUDENT-QUESTIONS.md — Discussion questions (goes inside your submission zip)

Answer directly under each question. 150–300 words each — **reasoning over length**.

The sections follow the four planks of the bridge. There are no standard answers; these are the
real point of the lab.

---

## A. Custody — where the asset actually is (plank a)

**A1.** In Lab 1, `totalCollateral()` read the vault's own ERC-20 balance, and the chain could
prove the invariant. Here, `MockTBillCustodian.realHoldings()` is just a number that a permissioned
address can move. Write the equivalent of `totalCollateral() == totalSupply()` for *this* system.
What does it actually assert, and what does it no longer prove?

> Your answer:

With no redemption requests or fees, and ignoring rounding, the expected reported-backing relation is:

```text
vault.reserveBalance() + custodian.realHoldings() == vault.totalClaimValue()
```

All terms use six-decimal USDC units; share value is `shares * NAV / 1e20`. Ex2's zero custody balance does not alone prove missing backing: subscription cash can remain in the vault.

With redemptions, the reported solvency comparison becomes:

```text
vault cash + queue cash + reported T-Bill value >= vault.assetsForShares(totalSupply - pendingShares) + pendingAssets + totalClaimable
```

Here `totalSupply` is tBILL supply; the other counters belong to the queue. Pending shares remain unburned but carry fixed payouts, so subtract them before valuing shares at current NAV. Settled shares are burned, but unclaimed cash remains owed.

For example, if 100 shares are enqueued at NAV 1 and NAV later becomes 1.10, those tickets still owe 100 USDC, not 110. Valuing the entire supply at the new NAV and then adding pending payouts would both reprice the fixed claims and count them twice.

Both comparisons assume synchronized records without double-counting sale proceeds. They are accounting checks, not contract-guaranteed invariants. A custodian can change reported holdings without moving cash; NAV can also be wrong. Matching numbers cannot prove securities exist, are unencumbered, belong to the SPV, or can promptly be sold.

**A2.** The T-Bills are held by an SPV, a legal entity set up so the fund's assets are bankruptcy-
remote from the issuer. Explain in your own words why a *legal* structure is part of a *technical*
design. If the SPV's custodian goes bankrupt, what does the on-chain token entitle its holder to?

> Your answer:

The SPV connects token records to enforceable rights over securities. Separating fund assets from the issuer requires effective ownership documents and custody arrangements; Solidity cannot provide bankruptcy remoteness. The technical design therefore needs a legal answer to the same question as the ledger: whose claim is being recorded, against whom, and over which assets? Without that connection, a perfectly functioning token contract can record balances that nobody is obliged to honour.

Issuer failure and custodian failure also need separate consideration. Isolating the fund from the issuer's creditors does not, by itself, establish how assets held through the custodian will be recovered.

If the custodian fails, properly segregated securities may be recoverable by the SPV or its representative, but recovery can take time. A holder receives only the rights established by fund documents and applicable law, potentially an interest in the fund or its distributions. The token alone guarantees neither direct title to particular T-Bills nor immediate repayment. UNIDROIT Principle 4 likewise leaves the legal effect of an underlying-asset link to applicable law. This lab supplies no documents establishing that entitlement.

---

## B. Attestation — how the chain learns the truth (plank b)

**B1.** `attest()` is the only function that moves the NAV, and it is gated by `REPORTER_ROLE`.
Everything downstream — subscription pricing, redemption payouts, both invariants — believes that
number. Name the failure modes of a single trusted reporter, and one mechanism (technical,
financial, or legal) that mitigates each.

> Your answer:

A single reporter creates several failure modes:

- A stolen key: hardware protection and approval by independent signers.
- Bribery or fabricated valuations: independent custody reconciliation, accountable audits and legal responsibility for reports.
- Decimal or input errors: scale checks and limits that send exceptional changes for review.
- Outage or delayed reporting: genuine update timestamps, maximum quote age and suspension of new pricing.
- Trading around privileged updates: staff trading restrictions, monitoring and published dealing cutoffs.

Multiple signatures cannot correct a shared false data source. These controls address different risks rather than proving the NAV is true.

The code also qualifies the question's premise: `attest()` requires `REPORTER_ROLE`, but anyone can call `MockPriceFeed.setPrice()` directly. A production feed must restrict every update path. Ex6 checks share escrow and funding of settled claims; neither invariant establishes fair valuation.

Authorization establishes who submitted a value. Reconciliation checks the evidence behind it. A valid signature from an authorized but compromised reporter still produces an economically false price, so role checks must be combined with independent valuation controls and an incident response process.

**B2.** A production feed would check whether `updatedAt` is stale; this lab's mock does not. A
**frozen** NAV is a different attack from a **wrong** NAV. Describe how a stale-but-honest NAV can
be exploited by someone who knows the true value has moved.

> Your answer:

Suppose the last honest NAV is 1 USDC, but current value is 1.10. Paying 1,000 USDC before the update buys 1,000 shares instead of about 909, diluting existing holders.

After a fall to 0.90, the same stale NAV lets a holder enqueue 1,000 shares for 1,000 USDC instead of 900. That payout stays locked even after NAV is corrected, leaving the overpayment with remaining holders. FIFO only orders payment; it cannot repair the price.

The attacker exploits outdated information without forging a report. I would require a genuine observation timestamp, an age limit and a disclosed dealing cutoff. Here `navPerShare()` ignores timestamps, and `latestRoundData()` returns the current block time on every read. Checking that synthetic timestamp alone would incorrectly treat an old price as fresh.

My policy would stop subscriptions and new redemption pricing when data is stale, while allowing already-funded claims to be collected. That avoids turning an oracle outage into a complete cash freeze. A maximum age only limits exposure: even a recent quote may be outdated after a sudden market move.

---

## C. Redemption — T+1 does not settle on-chain (plank c)

**C1.** The queue locks the payout at the NAV of **enqueue** time, not **settle** time. If the NAV
rises between the two, who gains and who loses — the redeemer, or the holders who stayed? Is that
the right allocation of risk, and how would you change it?

> Your answer:

Enqueuing 1,000 shares at NAV 1 locks 1,000 USDC. If NAV rises to 1.10, the redeemer forgoes 100 USDC of upside, which economically benefits remaining holders. If NAV falls, those holders bear the loss behind the fixed payout. The mock does not automatically calculate this redistribution.

Certainty helps redeemers, but immediate pricing rewards trading before NAV updates. I would price new requests at the next independently checked NAV after a published cutoff, then settle when cash arrives. Existing locked tickets should retain their terms. Requests received before the same cutoff should use the same valuation, and the settler should have no discretion to choose a more favourable pricing time.

This would make the allocation of market risk easier to explain: investors bear price movements until the scheduled valuation, then hold a fixed cash claim while settlement is pending. The tradeoff is that they no longer know their exact proceeds when submitting the request. I consider that preferable to rewarding whichever investor acts fastest against an outdated quote. The rule and expected settlement delay should be disclosed before investors subscribe.

`vault.reserveBalance() >= queue.pendingAssets()` is not a valid invariant: appreciation can remain in T-Bills rather than available cash. Ex6 instead requires queue cash to cover `totalClaimable`, the settled but unclaimed amount.

**C2.** In a 2008 money-market fund breaking the buck, and in USDC's 2023 depeg, redemptions were
handled very differently. Compare the two. For a tokenized T-Bill fund, what does it mean to "close
the redemption channel", and what should happen to the queue when it does? Then look at the lever
this lab actually ships: `pause()` (`PAUSER_ROLE`) freezes transfers, minting **and** redemption
together — one switch, no partial setting. Why is that the wrong tool for a liquidity squeeze, and
what would you rather have?

> Your answer:

In 2008, Reserve Primary Fund's NAV fell to $0.97 and redemptions were suspended with [SEC permission](https://www.sec.gov/divisions/investment/guidance/reservefundmmffaq.htm); recovery proceeded through [liquidation distributions](https://www.sec.gov/newsroom/press-releases/2010-16-reserve-primary-fund-distributes-assets-investors). During USDC's 2023 depeg, Circle maintained [1:1 redeemability after depositor protection was announced](https://www.circle.com/pressroom/3-3-billion-of-usdc-reserve-risk-removed-dollar-de-peg-closes), while banking disruption delayed processing. Circle reported [substantially clearing the backlog by March 15](https://www.circle.com/blog/march-15-update-on-usdc-operations). Market trading below par did not imply uninterrupted dollar withdrawals.

For a T-Bill fund, closing redemption stops conversion of claims into cash. Preserve existing tickets, locked amounts and FIFO order; publish reopening conditions. Stop new requests separately if pricing is unreliable. A temporary liquidity shortage should not silently cancel tickets or let later investors jump ahead. If the problem is an actual asset shortfall, resuming the queue alone cannot repair it; the fund needs a disclosed loss-allocation process consistent with its governing documents. Reporting available cash, outstanding requests and the reason for suspension would help holders distinguish waiting for cash from bearing a loss.

Token `pause()` is too broad for a cash shortage: it blocks transfers, subscription minting, enqueue transfers and settlement burns without creating liquidity. Already-settled `claim()` still transfers USDC because it does not touch tBILL. I would use separate subscription, request-intake and settlement controls, keeping funded claims available where lawful. A global pause should address wider contract or key compromise.

---

## D. Admission — the guest list and the backdoor (plank d)

**D1.** Every balance change in `TBillToken._update` checks **both** endpoints against the
whitelist. That is what stops a sanctioned address from receiving shares — and also what stops a
completely ordinary user who has not finished KYC. Where is the right line for a regulated fund,
and who should be able to move it?

> Your answer:

Eligibility should follow the fund's documented identity, sanctions and investor rules. Paying USDC does not establish eligibility, but unfinished KYC is not wrongdoing. Applicants need clear procedures, correction of errors and an appeal route where lawful; personal documents should remain off-chain.

An accountable compliance team should manage routine whitelist changes. Policy changes belong to fund governance with legal review, and removals should require independent approval and an auditable reason. Ex4 proves enforcement of recorded eligibility, not the accuracy of the identity checks. Removing a holder leaves their balance intact but blocks subsequent share movements. Operational addresses such as the queue also need approval for transfers and burns.

I would distinguish an application's pending status from an emergency restriction on an existing holder, even if both currently map to the same Boolean. The compliance system should record the reason and review status outside the token contract, with access limited to appropriate staff. Restrictions should be tied to the product's stated requirements rather than personal disagreement with a holder. Periodic review and a documented restoration process would reduce the risk that an erroneous freeze becomes permanent.

**D2.** A regulated fund asks for two levers: freeze an address (`setWhitelisted(addr, false)`) and
burn a balance (`MINTER_ROLE`) — both censorship tools, both justified by the product. But Ex4.3
shows the two **collide**: once an address is off the list, `burn()` reverts as well, because
`_update` guards both endpoints. So a freeze also blocks a seizure. Argue **both** sides of handing
an issuer these powers, then say what safeguards you would add and who should hold each key.

> Your answer:

Freeze and burn powers can support lawful restrictions, recovery from theft and correction of invalid issuance. The same powers enable censorship, mistaken exclusion or destruction of balances after key compromise. Burning a token does not itself transfer the underlying securities or establish lawful seizure.

Ex4.3 exposes the collision: de-listing a holder blocks both transfer and burn in `_update`. Restoring eligibility in a separate transaction just to burn reopens transfers meanwhile. I would specify a narrow enforcement mechanism with documented authority, events, notice and review rather than use that workaround.

A compliance multisig should control freezes; separate governance and compliance signers should authorize exceptional burns. Routine issuance and redemption should be confined to vault and queue contracts. An independent emergency multisig should hold pause authority, while role administration needs a governance multisig and delay. These are proposed safeguards. Role grants and revocations should be logged and reviewed, and emergency actions should receive prompt independent scrutiny. After enqueue, shares belong to the queue, so freezing the user does not stop an already-funded USDC claim; any required cash restriction needs its own explicit policy.

---

## E. Tests (Tier 1 required — this is Ex2, Ex3 and Ex4)

Turn the red tests green in `test/exercises/01_BridgeTasks.t.sol` to cover the scenarios below, and
write your test function names here:

| Plank | Scenario                                                             | Your test function name                             |
| ----- | -------------------------------------------------------------------- | --------------------------------------------------- |
| a     | Shares exist while the custodian holds nothing                       | `test_Ex2_SharesExistWhileTheCustodianHoldsNothing` |
| a     | `realHoldings()` moves with no cash moving                           | `test_Ex2_RealHoldingsIsJustANumber`                |
| b     | The claim value floats while the share count stays                   | `test_Ex3_TheClaimFloats_TheShareCountDoesNot`      |
| b     | One `attest` re-prices every holder at once                          | `test_Ex3_OneCallMovesTheWholeBook`                 |
| d     | A non-whitelisted address cannot receive shares, even by paying      | `test_Ex4_NoKyc_NoShares_EvenIfYouPay`              |
| d     | A whitelisted holder cannot send shares to a non-whitelisted address | `test_Ex4_TransferChecksBothEndpoints`              |
| d     | Removing an address freezes that holder — and blocks a burn          | `test_Ex4_FreezeBeatsConfiscation`                  |

Then write one more scenario you consider **most likely to be exploited** on a tokenized fund, and
say which plank it breaks:

> Your answer:

I would prioritize manipulation of the NAV used to lock redemption tickets, breaking attestation, plank (b). Every subscription and new redemption trusts that input; admission and FIFO cannot establish its truth.

In this mock, an ordinary whitelisted investor can exploit unrestricted `setPrice()`. Suppose an honest investor and the attacker each deposit 1,000 USDC at NAV 1, leaving 2,000 USDC in the vault and 1,000 shares each. The attacker changes NAV to 2, enqueues their shares to lock 2,000 USDC, and restores NAV 1. If an authorized settler processes that ticket, the attacker can claim all vault cash, leaving the honest investor's shares without backing.

This assumes sufficient vault cash, an operational queue and normal settlement; the attacker need not control the settler. Restoring NAV does not change `assetsLocked`. I would test that unauthorized feed updates revert and that accepted quotes satisfy authorization and freshness checks before creating tickets. In production, a compromised reporter is the corresponding risk even after the public setter is removed.
