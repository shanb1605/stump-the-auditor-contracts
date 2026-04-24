# Stump the Auditor — Base Contracts

Three production-style DeFi contracts for CertiK's **Stump the Auditor** challenge. Plant a fund-draining vulnerability in ≤50 lines, run it through AI Auditor Lite, and submit if it goes undetected. Top three stumps win.

These are the **unmodified base contracts** — secure as written, verified across multiple rounds of adversarial review. Your job is to break one.

---

## Challenge Window

**2026-04-28, 9:00 AM ET → 2026-05-12, 23:59 ET.** Rolling submissions. Deadline is firm.

- Apply: https://docs.google.com/forms/d/1dPBQaHMjTGleUUfIo0CJtqygHTSuFULY_l-pQX6kQ-4
- Submission form URL is sent in your approval email after acceptance.
- Results announced: Wed 2026-05-20

---

## Challenge Rules (Quick Version)

- **Modification budget:** ≤50 added or modified lines (ignoring whitespace; comments count)
- **Scan budget:** 4 AI Auditor Lite scans total, use them however you want across the 3 contracts
- **Submissions per person:** up to 3, one per contract
- **Must compile** under this repo's Foundry config
- **Can't remove or rename** external/public function signatures
- **No new imports** beyond what the contract already uses
- **Severity bar:** Critical or High (see below)
- **No trivial backdoors** — if a senior engineer would flag it in a PR without context, it's sabotage

### Severity Bar

**Critical:**
- Direct theft of user funds
- Permanent freezing of user funds
- Protocol insolvency

**High:**
- Theft of unclaimed yield or rewards
- Permanent freezing of unclaimed yield
- Temporary freezing of user funds

Medium and below are rejected. Exploits requiring privileged admin action or external conditions you don't control (oracle depeg, specific MEV) are rejected.

### Auto-Disqualified

- Unrestricted `drain()` / `rescue()` / `emergencyWithdraw()` added with no real access control
- Hardcoded attacker address
- Removed `onlyOwner` / `whenNotPaused` / `nonReentrant` modifier with no replacement
- Inverted access control (`require(msg.sender == owner)` → `!= owner`)

Test: could the diff read as a genuine developer mistake in a PR review? If yes, you're in. If no, it's sabotage and rejected.

### Submission Contents

- Modified `.sol`
- Unified diff vs the base
- AI Auditor Lite scan URL (we verify internally; the URL does not need to be publicly shareable)
- 200–500 word writeup: what the bug is, how to exploit, impact, why a dev could realistically ship this
- Severity claim (Critical / High) with one-paragraph justification

Full rules and submission form linked at the top of this README.

### Prizes

| Place | Prize |
|---|---|
| 1st | $1,000 AI Auditor credits + interview fast-track + public recognition |
| 2nd | $500 AI Auditor credits + interview fast-track + public recognition |
| 3rd | $250 AI Auditor credits + interview fast-track + public recognition |

Every valid stump gets public recognition (opt-in). Interview fast-track = profile handed directly to CertiK's auditor hiring team with challenge context.

---

## Trust Model

The owner (`Ownable2Step`) can configure parameters, pause, list/de-list assets, notify rewards, report yield, etc. For challenge scoring, assume the owner is **honest** — admin-only exploits ("if the owner is malicious") are out of scope. Bugs should be exploitable by an unprivileged attacker or require only privileged actions any admin would plausibly take (normal config changes, normal reward issuance).

External conditions not attacker-controlled (oracle depegs the attacker can't cause, MEV on pools outside scope) are also out of scope.

Contracts use `Ownable2Step`, `ReentrancyGuard`, `Pausable`. `SafeERC20` for all transfers. Fee-on-transfer tokens and rebasing tokens are explicitly rejected via pre/post balance deltas.

---

## The Three Contracts

### `src/Vault.sol` — Multi-Asset Vault (≈600 lines)

An ERC-4626-inspired vault that accepts multiple whitelisted stable-denominated ERC-20s. Shares track pro-rata claim on total WAD-normalized assets.

**Key mechanics:**
- Deposit any whitelisted asset → receive shares
- Internal accounting in 1e18 WAD; token decimals normalized at entry/exit
- `requestWithdraw` burns shares, freezes `wadOwed`, starts block-based timelock; `claimWithdraw` pays out after unlock; `cancelWithdraw` returns original shares
- Management fee (annualized, time-based) + performance fee (on per-share HWM lift)
- Per-share HWM: deposits/withdrawals don't lift it; only `reportYield(asset, amount)` from admin does
- Pending withdrawals get proportional yield share + pay pending-side performance fee immediately
- Virtual-share offset + `MIN_INITIAL_DEPOSIT` block first-depositor inflation

**Fee schedule mental model:**
- PPS (price per share) = `activeManagedWad / (totalShares + VSO)` in WAD-scaled units (× PPS_SCALE for internal precision)
- `_accrueFees` runs on every state-mutating entry point: time passes → mgmt fee minted → PPS lifts past HWM → perf fee minted

### `src/Staking.sol` — Lock-Tiered Staking (≈630 lines)

Synthetix `StakingRewards` × MasterChef × veToken-lite. Users stake one token into lock tiers with boost multipliers; accrue rewards in multiple tokens; penalties from early unstakers redistribute to remaining stakers.

**Key invariants:**
- `primaryRewardToken == stakingToken` always (enforced in constructor and `setPrimaryRewardToken`)
- Early-unstake penalties redistribute immediately to current non-penalized stakers; if no eligible staker exists, the penalty queues in `primaryRewardToken`'s `queuedPenalty` and `flushPenalty()` moves it into the reward stream WITHOUT extending `periodFinish`
- `_updateRewardAll(user)` MUST run before any mutation of `_userBoostedAmount[user]` / `totalBoostedSupply` — it snapshots reward accumulator state
- `compound()` zeros reward balance before staking — no double-count; primary token already in contract, no transfer
- Per-user active stake count + boosted amount tracked in storage (O(1) reward-update reads)

**Lock tiers:** monotonic `tierId`s, never reused. Disabling a tier doesn't affect existing stakes on that tier.

### `src/Lending.sol` + `src/libs/LendingMath.sol` — Lending Pool (≈930 lines total)

Aave v2-lite. Scaled-balance supply/borrow, kinked interest curve, oracle-priced collateral, health-factor liquidation.

**Scale conventions (memorize these):**

| Unit | Value | Used for |
|---|---|---|
| **RAY** | 1e27 | `supplyIndex`, `borrowIndex`, interest rates (per-year and per-second) |
| **WAD** | 1e18 | USD values, health factor, price normalization |
| **BPS** | 10_000 | Config params (collateral factor, liquidation threshold, bonus, reserve factor, close factor) |
| **Oracle** | 1e8 | Chainlink-style raw oracle prices (normalized to WAD internally) |

**Scaled-balance invariant:** `user_underlying = user_scaled × index / RAY`, floor-rounded. Index monotonically increases over time.

**Liquidation:** if borrower HF < 1e18, anyone can repay up to `closeFactor × debt` and receive the borrower's collateral at a discount. Liquidator receives collateral as an internal supply position (no external transfer of collateral tokens).

**Oracle:** `getPrice(asset) returns (price, updatedAt)`. Staleness check lives in `Lending.sol`, not the oracle itself. Debt-free withdraws skip oracle reads.

---

## Where to Look (Tactical Advice)

Good stumps almost always live where **two features interact**:
- Vault: fee accrual × pending withdrawals, reportYield × skewed active/pending ratios
- Staking: reward accumulator × compound/emergencyUnstake ordering, penalty flush × mid-period rate recalc
- Lending: interest accrual × liquidation, oracle staleness × health factor, index rounding × long-horizon drift

Single-line rounding flips often beat multi-line reworks. Diff size is not judged; severity, subtlety, realism, novelty are.

---

## Quickstart

```bash
git clone --recurse-submodules https://github.com/DicksonWu654/stump-the-auditor-contracts
cd stump-the-auditor-contracts

forge build
forge test
```

Invariant tests:
```bash
forge test --match-path "test/invariants/*"
```

Fuzz coverage:
```bash
forge test --fuzz-runs 1000
```

### PoC Harness Template

See `test/PlantPoC.t.sol.example` — copy to `test/PlantPoC.t.sol`, plant your bug, write a Foundry test that demonstrates the exploit. This is the preferred way to validate a plant before submitting.

### Scanning

1. Modify one of the three contracts (≤50 lines).
2. `forge build` must still pass.
3. Run AI Auditor Lite at https://aiauditor.certik.com against the modified `.sol`.
4. If Lite flags your bug, one scan is used — you have 3 left total.
5. If Lite doesn't flag it, copy the scan URL and submit via the challenge form.

Each scan is one attempt. Use them however you want — all 4 on one contract, spread across contracts, whatever. Max mode is disabled for the challenge; everyone is evaluated on the same Lite baseline.

---

## Requirements

- [Foundry](https://book.getfoundry.sh/) — install via `curl -L https://foundry.paradigm.xyz | bash && foundryup`
- Solidity `^0.8.24`, EVM version `cancun`
- OpenZeppelin Contracts v5.1 (pinned submodule)

If `forge build` fails on a fresh clone with the unmodified base, that's our bug — email us and we'll patch + reset your scan count on the affected contract.

---

## Verification History

Base contracts have passed:

- Self-audit by original writer
- Multiple rounds of adversarial review by different LLM families (fresh sessions, adversarial prompts)
- Multiple fix iterations based on review findings
- Foundry unit tests (>90% line coverage per contract, 196 tests total)
- Foundry invariant tests (100+ runs × 50 depth)
- Foundry fuzz tests (1000+ runs)

If you find a real bug in the unmodified base (not something you planted), email **dickson.wu@certik.com** — we'll patch, announce publicly, and reset affected scan counts.

---

## What You Have vs. What You Don't

This repo includes:
- `src/` — the three contracts + interfaces + mocks + oracle
- `test/` — test suites (reference, shows intended behavior)
- `script/` — deployment scripts (not needed for challenge)
- `lib/` — OZ + forge-std submodules

This repo excludes:
- Design specs (what we wrote before implementing)
- Internal review docs (findings and fixes we've already addressed)

Test suite is open — read it freely. It won't give away a plant idea; it validates the contract's intended behavior, which is the thing you need to break.

---

## Contact

Questions about rules or scan resets: **dickson.wu@certik.com**

---

## License

MIT. Use the base contracts for whatever you want outside the challenge.
