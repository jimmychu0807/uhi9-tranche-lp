I want to create a test script that can read the following input:

## 1. LP Deposit

```
lpDeposit: lpUser1, 10, 20000, 80, 20
lpDeposit: lpUser2, 100, 200000, 100, 0
```
- The first line represents a liquidity provider deposits (10 ETH, 20000 USDC), and get 80% senior tranche back. This indicates there are 20% retrieved as junior tranche.
- The 2nd line represents a LP deposits (100 ETH, 200000 USDC), and getting back 100% senior tranche and 0% junior tranche.
- The last two digits should add up to 100.

## 2. Swap

```
swap: swapUser1, 2,
swap: swapUser2,  , 100
```
- The first line means swapUser1 make a swap of 2 ETHs to USDC.
- The second line means swapUser2 make a swap of 100 USDC to ETHs.

## 3. Time Pass

```
pass: 86400
```
This means time has passed for 86400 seconds.

## 4. LP Withdrawal

```
lpWithdraw: lpUser1, 60
```
- The first line means **lpUser1** withdraw from LP, withdrawing 60% from its original state, returning the srt & jrt tokens and get back ETH & USDC.
- Show how much ETH & USDC are retrieved due to the srt, and how much ETH & USDC are retrieved due to the jrt.

## 5. User Assertion

```
userAssert: lpUser1, >= 10.01, > 20000
userAssert: lpUser2, , ~20000
```

- The first line indicates to assert that **lpUser1** has greater than or equal amount of 10.01 ETH and has USDC strictly greater than 20000.
- The second line indicates to assert that **lpuser2** has an amount of about 20000 USDC. (You can define a small delta value. If the abs difference of lpUser2 and the specified amount is less than delta, they are regarded as the same)

## 6. Pool Assertion

```
hookAssertion: lpUser1, ~20%, ~3%
poolAssert: ~1, ~2000
```
- The first line indicates the hook should have about 20% srt tokens and about 3% of jrt tokens for **lpUser1**.
- The second line indicates the pool should have a total of about 1 ETH and about 2000 USDC.
