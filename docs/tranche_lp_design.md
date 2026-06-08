# Tranched LP Position Design for Uniswap v4
## Senior Tranche (Fixed Yield) / Junior Tranche (Leveraged Yield)

> Derived from research and design discussion, June 2026.
> Context: Uniswap v4 hook-based structured LP product for ETH/USDC pools.

---

## Table of Contents

1. [Core Concept](#1-core-concept)
2. [Capital Stack](#2-capital-stack)
3. [Hook Architecture](#3-hook-architecture)
4. [Fixed Yield Rate Design](#4-fixed-yield-rate-design)
5. [Personal Tranche Ratio — Non-Binary Deposits](#5-personal-tranche-ratio--non-binary-deposits)
6. [Concrete Example: ETH/USDC Pool](#6-concrete-example-ethusdс-pool)
7. [Scenario Analysis](#7-scenario-analysis)
8. [Fee Tier Selection](#8-fee-tier-selection)
9. [Receipt Token Design](#9-receipt-token-design)
10. [Implementation Notes](#10-implementation-notes)
11. [Risk Parameters Summary](#11-risk-parameters-summary)

---

## 1. Core Concept

An LP position in a CPMM has two separable cash flow components:

| Component | Characteristic |
|---|---|
| **Fee income** | Positive, relatively stable, accrues continuously per swap |
| **Impermanent Loss (IL)** | Negative, volatile, path-dependent, always ≤ 0 |

The tranche design **splits these cash flows** between two investor classes:

- **Senior Tranche**: receives fee income first up to a fixed promised yield. Zero IL exposure as long as junior buffer is intact.
- **Junior Tranche**: receives residual fee income after senior is paid, absorbs 100% of IL. Compensated by leveraged upside in fees when the pool is active.

The junior tranche is structurally equivalent to **writing a put on the pool's value** to the senior tranche. This is a DeFi-native implementation of a structured finance CDO/CLO, implemented entirely via a Uniswap v4 hook with no external capital or derivatives required.

---

## 2. Capital Stack

```
┌─────────────────────────────────────┐
│         POOL TOTAL VALUE            │
├─────────────────────────────────────┤
│  SENIOR TRANCHE                     │
│  • Fixed APY (model-derived)        │
│  • First claim on fees              │
│  • No IL exposure                   │
│  • Redeemable at par (entry value)  │
├─────────────────────────────────────┤
│  JUNIOR TRANCHE                     │
│  • Residual fees after senior paid  │
│  • Absorbs ALL IL                   │
│  • Higher potential return          │
│  • First-loss capital               │
└─────────────────────────────────────┘
```

### Leverage Factor

$$\lambda = \frac{\text{Total Pool Capital}}{\text{Junior Capital}}$$

At 75% senior / 25% junior: $\lambda = 4\times$. Junior earns (or loses) at 4× leverage on the net spread between fee APY and IL APY.

### Junior Wipeout Threshold

The junior tranche is wiped out when cumulative IL exceeds junior capital:

$$IL_{\text{cumulative}} > \text{Junior Capital} + \text{Junior Fee Accrual}$$

For a 25% junior tranche on an ETH/USDC pool at $2000, this occurs at approximately a **3× price move** (ETH to $6000 or $667). Beyond this, senior begins taking principal losses.

---

## 3. Hook Architecture

### State Tracked Per Pool

```solidity
struct TranchePool {
    // Capital balances
    uint256 seniorCapital;       // total senior deposits at entry prices
    uint256 juniorCapital;       // total junior deposits at entry prices

    // Accrual tracking
    uint256 seniorYieldAccrued;  // fees owed to senior so far
    uint256 juniorFeeAccrued;    // residual fees owed to junior
    uint256 ilAccrued;           // cumulative IL since pool inception

    // Reference invariant for IL calculation
    uint256 k0;                  // invariant at pool inception / last reset
    uint256 seniorFixedRate;     // current epoch rate in WAD (e.g. 8e16 = 8%)
    uint256 lastAccrualTime;     // timestamp of last accrual
    uint256 epochStart;          // start of current rate epoch
}
```

### State Tracked Per LP Position

```solidity
struct LPPosition {
    uint256 seniorDeposit;   // LP's senior capital at entry
    uint256 juniorDeposit;   // LP's junior capital at entry
    uint256 phi;             // personal senior fraction in WAD (0 to 1e18)
    uint256 k0_personal;     // pool invariant at LP's deposit time
    uint256 depositTime;     // timestamp
    uint256 srtBalance;      // Senior Receipt Tokens minted
    uint256 jrtBalance;      // Junior Receipt Tokens minted
}
```

### Hook Callback Flow

#### `afterAddLiquidity` — Deposit

1. LP specifies total deposit amount $C$ and personal senior fraction $\phi \in [0,1]$.
2. Hook computes $C_s = \phi \cdot C$ and $C_j = (1-\phi) \cdot C$.
3. Enforce aggregate pool constraint: $\bar{\phi}_{\text{pool}} \leq 1 - \delta_{\min}$.
   - If breached: either reject excess senior allocation or reduce $r_s$ (see §4).
4. Record current invariant $k_0^{\text{personal}}$ as the IL reference for this LP.
5. Mint $C_s$ worth of **Senior Receipt Tokens (SRT)** and $C_j$ worth of **Junior Receipt Tokens (JRT)**.
6. Emit event with position metadata.

#### `afterSwap` — Continuous Accrual

Fires on every swap. Steps:

**Step A — Fee earned this swap:**
$$\text{feeEarned} = \Delta k = k_{\text{new}} - k_{\text{old}}$$
(growth in pool invariant represents fee revenue in reserve-denominated terms)

**Step B — Incremental IL since last accrual:**
$$IL_{\text{incremental}} = \left[2\sqrt{k \cdot p_{\text{now}}} - (x_0 p_{\text{now}} + y_0)\right] - IL_{\text{prev}}$$

Use TWAP price (not spot) to prevent manipulation.

**Step C — Accrue senior yield:**
$$\text{seniorDue} = \text{seniorCapital} \times r_s \times \Delta t / 365\text{ days}$$
$$\text{seniorAccrual} = \min(\text{feeEarned},\ \text{seniorDue})$$

**Step D — Route residual to junior:**
$$\text{juniorNet} = (\text{feeEarned} - \text{seniorAccrual}) - IL_{\text{incremental}}$$

Can be negative — junior's recorded claim shrinks when IL exceeds residual fees.

**Step E — Check junior buffer health:**
$$\text{juniorBuffer} = \frac{\text{juniorCapital} + \text{juniorFeeAccrued} - \text{ilAccrued}}{\text{totalPoolValue}}$$

If $\text{juniorBuffer} < \delta_{\min}$: pause new senior deposits, emit alert event.

#### `beforeRemoveLiquidity` — Withdrawal

**Senior payout** (first-out):
$$\text{seniorPayout} = \text{seniorCapital} + \text{seniorYieldAccrued}$$

Funded from pool reserves. If pool value is insufficient, junior capital is liquidated first.

Protection condition:
$$\text{seniorPayout} \leq V_{\text{pool}} - \max(0,\ \text{juniorNet})$$

**Junior payout** (residual):
$$\text{juniorPayout} = \max\!\left(0,\ V_{\text{pool}} - \text{seniorPayout}\right)$$

Burn SRT/JRT proportional to withdrawal amount.

---

## 4. Fixed Yield Rate Design

The senior rate **must not be pre-determined**. A hardcoded rate is fragile — too high causes insolvency in low-fee environments, too low attracts no senior capital. Three approaches, ranked by sophistication:

### Approach A: Fee-Derived Rate (Simplest)

$$r_s = \alpha \times \bar{F}_{30d}$$

where $\bar{F}_{30d}$ is the 30-day trailing fee APY (computed from invariant growth) and $\alpha \in (0,1)$ is a protection ratio (e.g. 0.5).

- Always fundable by construction: $r_s \leq \bar{F}$ guaranteed
- Rate floats with pool activity
- **Weakness:** rate is uncertain at deposit time

### Approach B: Market-Discovered Rate (Most Elegant)

Senior depositors bid minimum acceptable yields; junior depositors bid maximum fundable rates. Clearing rate $r_s^*$ where supply meets demand:

$$\sum_i S_i \cdot \mathbf{1}[r_i \leq r_s^*] = \frac{1-\delta}{\delta} \sum_j J_j \cdot \mathbf{1}[r_j \geq r_s^*]$$

Implemented as a periodic rate auction (e.g. every 30 days). Analogous to Notional Finance's fixed-rate CFMM.

- Rate reflects actual supply/demand for risk
- Seniors get committed rate for the epoch period — genuine fixed income
- **Weakness:** requires bootstrapping liquidity on both sides simultaneously

### Approach C: Model-Derived Rate (Most Principled) ✅ Recommended

$$\boxed{r_s = \bar{F}_{30d} - \frac{\hat{\sigma}^2}{8} - z_{0.95} \cdot \frac{\hat{\sigma}}{2\sqrt{2}} \cdot \sqrt{\delta}}$$

clipped to $[0,\ \alpha \cdot \bar{F}_{30d}]$.

Where:
- $\bar{F}_{30d}$: 30-day trailing fee APY from invariant growth
- $\hat{\sigma}^2 / 8$: expected IL APY (from realized vol $\hat{\sigma}$)
- $z_{0.95} \cdot \frac{\hat{\sigma}}{2\sqrt{2}} \cdot \sqrt{\delta}$: safety buffer at 95% confidence, scaled by junior fraction $\sqrt{\delta}$

**Volatility source** (in order of preference):
1. Rolling realized vol from Uniswap v4 TWAP price impacts stored in hook
2. Chainlink implied vol feed (if available for the pair)
3. Simple rolling log-return standard deviation stored in hook state

**Key behaviors:**
- Rate = 0 when $\bar{F}_{30d} \leq \hat{\sigma}^2/8 + \text{buffer}$ → hook pauses senior deposits automatically
- Rate adjusts weekly (or each epoch) — published 24 hours in advance
- Thinner junior buffer ($\delta$ smaller) → larger buffer term → lower $r_s$ → protects remaining senior

### Hybrid Implementation (Recommended)

```
r_s = clip(
    F_30d - (σ² / 8) - safety_buffer,   ← Approach C floor
    0,                                    ← never negative
    α × F_30d                             ← Approach A ceiling
)
```

### Rate Response to Senior/Junior Imbalance

When too many LPs choose high $\phi$ (senior-heavy pool), $r_s$ scales down:

$$r_s(\text{adjusted}) = r_s^{\text{model}} \times \frac{\delta_{\text{current}}}{\delta_{\min}}$$

This reduces senior incentive and attracts junior capital — a built-in market-clearing mechanism.

---

## 5. Personal Tranche Ratio — Non-Binary Deposits

Each LP specifies a **personal senior fraction** $\phi \in [0, 1]$ at deposit. This is not binary.

$$C_s = \phi \cdot C \qquad C_j = (1-\phi) \cdot C$$

### Payoff Formula

$$\text{Personal Return} = \phi \cdot r_s + (1-\phi) \cdot r_j$$

where $r_j$ is the realized junior APY (variable, can be negative).

### Personal Breakeven Ratio

In a loss scenario where junior APY = $r_j < 0$, the minimum $\phi$ to avoid losing money:

$$\phi_{\text{break}} = \frac{-r_j}{r_s - r_j}$$

Example: $r_s = 8\%$, $r_j = -40\%$ → $\phi_{\text{break}} = 40\% / 48\% = 83\%$

### Payoff at Different $\phi$ (Scenario A: Fee 15%, IL 2%)

| $\phi$ | Senior | Junior | Blended Return | Profile |
|---|---|---|---|---|
| 0.00 | $0 | $100,000 | **28%** | Pure risk taker |
| 0.25 | $25,000 | $75,000 | **23%** | Risk-leaning |
| 0.50 | $50,000 | $50,000 | **18%** | Balanced |
| 0.75 | $75,000 | $25,000 | **13%** | Conservative |
| 1.00 | $100,000 | $0 | **8%** | Pure fixed income |

### Aggregate Pool Constraint

The pool-wide average senior fraction must stay within bounds:

$$\bar{\phi}_{\text{pool}} = \frac{\sum_i \phi_i C_i}{\sum_i C_i} \leq 1 - \delta_{\min}$$

---

## 6. Concrete Example: ETH/USDC Pool

**Setup:**
- ETH price: $p_0 = \$2000$
- Total pool capital: $\$1{,}000{,}000$
- Senior tranche: $\$750{,}000$ (75%)
- Junior tranche: $\$250{,}000$ (25%)
- Leverage factor: $\lambda = 4\times$
- Target senior yield: 8% APY

**Initial reserves:**
$$x_0 = \frac{\$500{,}000}{\$2000} = 250 \text{ ETH}, \qquad y_0 = \$500{,}000 \text{ USDC}$$
$$k = 250 \times 500{,}000 = 125{,}000{,}000$$

### IL at Various Price Endpoints

$$IL(p) = \frac{2\sqrt{p/p_0}}{1 + p/p_0} - 1$$

| ETH Final Price | Price Ratio | IL % | IL on $1M Pool |
|---|---|---|---|
| $2000 | 1.00× | 0.00% | $0 |
| $2200 | 1.10× | −0.23% | −$2,300 |
| $2500 | 1.25× | −0.98% | −$9,800 |
| $3000 | 1.50× | −3.37% | −$33,700 |
| $4000 | 2.00× | −11.80% | −$118,000 |
| $6000 | 3.00× | −25.00% | −$250,000 ← Junior wipeout |
| $1600 | 0.80× | −0.55% | −$5,500 |
| $1333 | 0.67× | −3.37% | −$33,700 |
| $1000 | 0.50× | −11.80% | −$118,000 |
| $667 | 0.33× | −25.00% | −$250,000 ← Junior wipeout |

**Junior wipeout boundary:** ETH to $6000 or $667 (3× move in either direction).

---

## 7. Scenario Analysis

Senior rate = 8% APY ($60,000/year on $750,000). Junior APY formula:

$$r_j = 4 \times (F - IL) - 3 \times 8\%$$

### Scenario Matrix

| | IL APY 2% (±10%) | IL APY 5% (±20%) | IL APY 12% (±40%) | IL APY 25% (3× move) |
|---|---|---|---|---|
| **Fee 5%** | Sr ✅ 8% / Jr **−12%** | Sr ✅ 8% / Jr **−24%** | Sr ⚠️ 5% / Jr **−100%** | Sr ❌ loss / Jr wiped |
| **Fee 10%** | Sr ✅ 8% / Jr **+16%** | Sr ✅ 8% / Jr **+4%** | Sr ⚠️ 8% / Jr **−56%** | Sr ❌ loss / Jr wiped |
| **Fee 15%** | Sr ✅ 8% / Jr **+28%** | Sr ✅ 8% / Jr **+16%** | Sr ✅ 8% / Jr **−20%** | Sr ⚠️ 5% / Jr wiped |
| **Fee 20%** | Sr ✅ 8% / Jr **+40%** | Sr ✅ 8% / Jr **+28%** | Sr ✅ 8% / Jr **−4%** | Sr ❌ loss / Jr wiped |
| **Fee 30%** | Sr ✅ 8% / Jr **+64%** | Sr ✅ 8% / Jr **+52%** | Sr ✅ 8% / Jr **+26%** | Sr ⚠️ 8% / Jr **−12%** |

⚠️ = senior partially paid from fees; junior capital drawn down to cover the gap.

### Four Full Scenarios

#### Scenario A: Benign Market (Fee 15%, IL 2%, ETH ~$2,200)

| Item | Amount |
|---|---|
| Gross fee revenue (15% × $1M) | +$150,000 |
| IL loss (2% × $1M) | −$20,000 |
| Net pool return | +$130,000 |
| Senior payment (8% × $750K) | −$60,000 |
| Junior net | **+$70,000 (+28% APY)** |
| Senior APY | **+8% ✅** |

#### Scenario B: Volatile Bull (Fee 20%, IL 12%, ETH ~$3,200)

| Item | Amount |
|---|---|
| Gross fee revenue (20% × $1M) | +$200,000 |
| IL loss (12% × $1M) | −$120,000 |
| Net pool return | +$80,000 |
| Senior payment (8% × $750K) | −$60,000 |
| Junior net | **+$20,000 (+8% APY)** |
| Senior APY | **+8% ✅** |

Junior breakeven — high volatility pushed IL up to consume all excess fees.

#### Scenario C: Low Fee, High Volatility (Fee 10%, IL 12%, ETH crashes to ~$1,100)

| Item | Amount |
|---|---|
| Gross fee revenue (10% × $1M) | +$100,000 |
| IL loss (12% × $1M) | −$120,000 |
| Net pool return | −$20,000 |
| Senior payment (8% × $750K) | $60,000 (funded from fees) |
| Junior net | **−$100,000 (−40% APY)** |
| Senior APY | **+8% ✅** (junior absorbs loss) |

Senior protected; junior loses 40% of capital.

#### Scenario D: Catastrophic (Fee 10%, IL 28%, ETH to $6,500+)

| Item | Amount |
|---|---|
| Gross fee revenue (10% × $1M) | +$100,000 |
| IL loss (28% × $1M) | −$280,000 |
| Net pool return | −$180,000 |
| Junior capital remaining | ~$10,000 (nearly wiped) |
| Senior shortfall | ⚠️ ~$50,000 received vs $60,000 promised |
| Senior APY | **~5.8% ❌** — junior buffer breached |

Junior wipeout threshold crossed. Senior takes a haircut.

---

## 8. Fee Tier Selection

### Senior Protection Condition

$$F \geq r_s \times \frac{\text{Senior Capital}}{\text{Total Capital}} + IL$$

For 75/25 split, 8% senior rate:

$$F \geq 6\% + IL$$

### Fee Sufficiency Table

| IL Environment | Min Fee APY Needed | Recommended Tier |
|---|---|---|
| Stable ETH (±10%, IL ~2%) | 8% | 0.05% (high volume) or 0.30% |
| Normal vol (±20%, IL ~5%) | 11% | 0.30% with decent volume |
| High vol (±40%, IL ~12%) | 18% | 0.30% high volume, or 1.00% |
| Extreme vol (±3×, IL ~25%) | 31% | Not viable — junior needed as primary buffer |

### Recommendation: 0.30% Fee Tier

For ETH/USDC at 75/25 split:
- Most liquid tier — attracts maximum volume
- At 50%+ daily volume/TVL, generates 15–20% fee APY
- Comfortably covers 8% senior yield + moderate IL (up to ~12%)
- 0.05% tier: only viable in low-vol, high-volume conditions
- 1.00% tier: deters volume; pool sits idle accumulating IL with no fee offset

**Optimal enhancement:** combine with a dynamic fee hook that scales fee rate with realized volatility — low fees in stable periods to attract volume, high fees in volatile periods to compensate IL. The tranche structure and dynamic fee approach are naturally complementary.

---

## 9. Receipt Token Design

Each LP deposit mints two ERC-1155 tokens:

| Token | Name | Represents | Behavior |
|---|---|---|---|
| SRT | Senior Receipt Token | Fixed-yield claim on $C_s$ | Bond-like; predictable cash flow |
| JRT | Junior Receipt Token | Residual + IL exposure on $C_j$ | Leveraged yield token |

### Composability

| Use Case | Token | Notes |
|---|---|---|
| Lending collateral | SRT | Bond-like cash flow accepted by Aave/Morpho |
| Secondary market trading | JRT | Price reflects market's expectation of fee − IL spread |
| Mid-position risk rebalancing | Both | Sell JRT → transfer IL risk without withdrawing |
| Ratio adjustment post-deposit | Both | Burn + remint via hook to change $\phi$ |

### Token Metadata (ERC-1155)

```solidity
struct SRTMetadata {
    uint256 poolId;
    uint256 depositAmount;    // original capital deposited as senior
    uint256 entryRate;        // r_s at deposit time
    uint256 epochExpiry;      // when current rate epoch ends
    uint256 accrualCheckpoint;
}

struct JRTMetadata {
    uint256 poolId;
    uint256 depositAmount;    // original capital deposited as junior
    uint256 k0;               // pool invariant at deposit — IL reference point
    uint256 entryPrice;       // ETH price at deposit
    uint256 depositTime;
}
```

---

## 10. Implementation Notes

### 10.1 IL Computation — Use TWAP Not Spot

**Critical:** always use the Uniswap v4 built-in TWAP oracle for price feeds when computing IL in hook callbacks. Spot price is manipulable via flash loans:

- A flash loan can briefly move spot price → hook computes large IL → junior claim is drained
- TWAP over 30-minute window smooths this out
- Use `IPoolManager.observe()` or the v4 oracle hook pattern

```solidity
// Prefer this:
uint256 price = getTWAP(poolKey, 1800); // 30 min TWAP

// Not this:
uint256 price = getSqrtPriceX96(); // spot — manipulable
```

### 10.2 Gas Optimization — Lazy Accrual

Do **not** compute full IL and fee accrual on every single swap — gas cost is prohibitive for a busy pool. Instead:

- Store cumulative fee and price checkpoints cheaply on each swap (a single SSTORE)
- Do full accrual computation only on deposit/withdrawal callbacks
- Use a **storage packing** strategy: pack `lastK`, `lastPrice`, `lastTimestamp` into a single slot

```solidity
// Packed checkpoint — 1 SSTORE per swap
struct Checkpoint {
    uint128 kAccumulator;   // cumulative invariant growth
    uint64  lastTimestamp;
    uint64  lastPrice;      // compressed price (sufficient precision)
}
```

### 10.3 Minimum Junior Ratio Enforcement

The hook must enforce $\delta_{\min}$ (e.g. 20%) continuously:

```solidity
modifier checkJuniorHealth() {
    _;
    uint256 juniorBuffer = (juniorCapital + juniorFeeAccrued - ilAccrued)
                           .divWad(totalPoolValue());
    if (juniorBuffer < DELTA_MIN) {
        _pauseSeniorDeposits();
        emit JuniorBufferAlert(juniorBuffer);
    }
}
```

Apply this modifier on `afterAddLiquidity` and `afterSwap`.

### 10.4 Rate Epoch Management

Senior rate $r_s$ should update on a defined epoch schedule (weekly recommended):

```solidity
uint256 constant EPOCH_DURATION = 7 days;

function _updateRate() internal {
    if (block.timestamp < pool.epochStart + EPOCH_DURATION) return;

    uint256 newRate = _computeModelRate(); // Approach C formula
    pool.pendingRate = newRate;
    pool.rateEffectiveAt = block.timestamp + 24 hours; // 24h notice
    pool.epochStart = block.timestamp;

    emit RateUpdated(pool.seniorFixedRate, newRate, pool.rateEffectiveAt);
}
```

The 24-hour notice window allows senior depositors to exit if the new rate is unacceptable.

### 10.5 Realized Volatility Oracle (On-Chain)

Store a rolling window of log price returns in the hook to compute $\hat{\sigma}$:

```solidity
uint256 constant VOL_WINDOW = 30; // 30 observations
int256[30] logReturns;            // circular buffer
uint8 volWriteIndex;

function _updateVol(uint256 newPrice, uint256 lastPrice) internal {
    int256 logReturn = int256(newPrice).divWad(int256(lastPrice)); // ln approx
    logReturns[volWriteIndex % VOL_WINDOW] = logReturn;
    volWriteIndex++;
}

function _computeVol() internal view returns (uint256 sigma) {
    // Compute variance of logReturns circular buffer
    // Annualize: sigma = stddev * sqrt(365 * observations_per_day)
}
```

Use a Babylonian sqrt and fixed-point arithmetic (WAD = 1e18 throughout).

### 10.6 Handling Fee Shortfall (Defer vs Insurance Fund)

Two models for when fees are insufficient to cover senior yield in a period:

**Model A — Defer (Recommended for simplicity):**
```
Unpaid senior yield → accrues as debt against junior capital
Junior cannot withdraw until senior debt is cleared
```

**Model B — Insurance Fund:**
```
0.5% of all fee revenue → protocol insurance reserve
Reserve backstops senior yield shortfalls
Reserve earns lending yield when idle (Aave/Morpho integration)
```

Model A is simpler and more capital-efficient. Model B is safer for seniors and more appealing to institutional depositors. Model B also aligns with the "idle capital → lending" yield enhancement pattern.

### 10.7 Ratio Rebalancing Post-Deposit

Allow LPs to adjust their $\phi$ after deposit without full withdrawal:

```solidity
function rebalanceTranche(
    PoolKey calldata key,
    uint256 positionId,
    uint256 newPhi  // new senior fraction in WAD
) external {
    LPPosition storage pos = positions[positionId];
    require(msg.sender == pos.owner, "not owner");

    uint256 oldSenior = pos.seniorDeposit;
    uint256 newSenior = pos.totalDeposit.mulWad(newPhi);
    int256 delta = int256(newSenior) - int256(oldSenior);

    // Burn old SRT/JRT, mint new ones
    _burnSRT(pos.srtBalance);
    _burnJRT(pos.jrtBalance);

    pos.phi = newPhi;
    pos.seniorDeposit = newSenior;
    pos.juniorDeposit = pos.totalDeposit - newSenior;

    _mintSRT(pos.owner, newSenior, ...);
    _mintJRT(pos.owner, pos.juniorDeposit, ...);

    // Recheck pool aggregate constraint
    _enforceAggregatePhiCap();
}
```

### 10.8 Emergency Circuit Breakers

Implement the following halts:

| Trigger | Action |
|---|---|
| Junior buffer < $\delta_{\min}$ | Pause new senior deposits |
| Junior buffer < $\delta_{\min} / 2$ | Pause ALL new deposits |
| IL > 80% of junior capital | Force rebalancing mode: excess fees routed entirely to senior until buffer restored |
| Oracle price stale > 1 hour | Pause accrual computation; use last known rate |
| Pool invariant decreased (fee bug) | Full pause; emit emergency event |

### 10.9 Reentrancy and Callback Safety

Uniswap v4 hooks execute inside the PoolManager's transient lock. Key safety rules:

- All state updates in hook callbacks must be idempotent under reentrancy
- Use `nonReentrant` modifier on all external LP-facing functions
- Do not make external calls to Aave/Morpho inside `afterSwap` — queue them as pending actions executed by a separate `harvest()` function
- Never store raw token amounts without recording the block number — stale values from a reorg are dangerous

### 10.10 Uniswap v4 Hooks Required

```solidity
function getHookPermissions() public pure override returns (Hooks.Permissions memory) {
    return Hooks.Permissions({
        beforeInitialize:        false,
        afterInitialize:         true,   // set k0, epochStart
        beforeAddLiquidity:      true,   // validate phi, check constraints
        afterAddLiquidity:       true,   // mint SRT/JRT
        beforeRemoveLiquidity:   true,   // compute payouts, enforce ordering
        afterRemoveLiquidity:    true,   // burn SRT/JRT
        beforeSwap:              false,
        afterSwap:               true,   // accrual, vol update, buffer check
        beforeDonate:            false,
        afterDonate:             false,
        beforeSwapReturnDelta:   false,
        afterSwapReturnDelta:    false,
        afterAddLiquidityReturnDelta:    false,
        afterRemoveLiquidityReturnDelta: false
    });
}
```

### 10.11 Testing Checklist

- [ ] IL computation matches analytical formula at known price endpoints
- [ ] Senior payout equals par + accrued yield in all scenarios where junior buffer > 0
- [ ] Junior payout is non-negative (floored at zero)
- [ ] TWAP manipulation test: flash loan price spike should not drain junior claim
- [ ] Epoch rate update with 24h notice window fires correctly
- [ ] Aggregate $\phi$ cap enforcement: senior deposits rejected when pool is senior-heavy
- [ ] Junior wipeout scenario: senior takes correct haircut proportional to remaining pool value
- [ ] Reentrancy: SRT/JRT minting cannot be re-entered via ERC-1155 callback
- [ ] Gas benchmark: `afterSwap` hook overhead < 10,000 gas
- [ ] Rate = 0 condition: hook correctly pauses senior deposits when model rate goes negative

---

## 11. Risk Parameters Summary

| Parameter | Symbol | Recommended Value | Notes |
|---|---|---|---|
| Minimum junior fraction | $\delta_{\min}$ | 20% | Below this, senior deposits paused |
| Senior rate cap | $\alpha$ | 0.5 | Max 50% of trailing fee APY |
| Rate epoch duration | — | 7 days | Weekly re-computation |
| Rate notice window | — | 24 hours | Before new rate takes effect |
| TWAP window | — | 30 minutes | Oracle manipulation protection |
| Vol observation window | — | 30 days | For $\hat{\sigma}$ computation |
| Safety buffer confidence | $z_\alpha$ | 1.645 (95%) | Conservative protection |
| Insurance fund cut (if used) | — | 0.5% of fees | Model B only |
| Circuit breaker threshold 1 | — | $\delta_{\min}$ | Pause new senior deposits |
| Circuit breaker threshold 2 | — | $\delta_{\min}/2$ | Pause all deposits |
| Emergency rebalancing trigger | — | IL > 80% of junior capital | Route all fees to senior |

---

## References

- Voronin et al. (2026). *A Route to Zero Impermanent Loss in CPMMs.* [arXiv:2604.28017](https://arxiv.org/html/2604.28017v1)
- Uniswap v4 Hook Incubator Cohort 8 (UHI8). *Non Toxic Pools submission.* [UHI Hook Directory](https://github.com/AtriumAcademy/UHI-Hook-Data)
- Uniswap v4 Documentation. [docs.uniswap.org](https://docs.uniswap.org)
- Panoptic Finance. *Perpetual Options on Uniswap.* [panoptic.xyz](https://panoptic.xyz)
- Notional Finance. *Fixed Rate Lending via CFMM.* [notional.finance](https://notional.finance)

---

*Document generated: June 2026. For research and educational purposes.*
