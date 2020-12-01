# Anon.Credit Audit

This audit was performed against commit hash `e6eef0ef2c343a2763cab2c2000c6d795b1e5f36` and covers the differences between the `Anon.Credit` contract and the `Tornado` contract from which it is based.

## High Severity

None!

## Medium Severity

None!

## Low Severity

### Unnecessary operator functions

The Anon.Credit contract includes permissioned functions which can only be called by the operator. These functions are `updateVerifier`, `setStakingToken`, and `changeOperator`. Since Anon.Credit uses the same trusted setup as Tornado Cash, the verifier should not need to be updated. Since the staking token address can be determined ahead of deployment, the staking token address should be set in the constructor. Implementing these modifications would simplify the state machine and improve the permissionlessness of the contract.

#### Response

`updateVerifier` has been removed. Using create2 to get the Uniswap v2 Exchange address of the staking token (ANON/ETH LP share) before deploying this contract is an additional moving part. It's simpler to deploy the contract, then deploy the Uniswap v2 Exchange, then set the staking token, and then disable the operator.

### Unnecessary state machine complexity

The Anon.Credit contract involves three core components: `Initialization`, `Deposits`, and `Staking`. All three component currently share interdependencies which yields a complex state machine. This complexity leads to strange edge cases like a user being able to call `unstake` without having previously called `stake`. Note, calling unstake in this case will transfer a reward of 0 bonus tokens and therefore is not a critical issue.

Creating a clearly defined state machine for each component of this contract would reduce the risk of unexpected edge cases.

#### Response

That edge case isn't bad. And the core components are largely independent. Initialization is over before any deposits or staking takes place. Deposits in the *first round only* generate ANON, which is used for staking (as ANON/ETH LP shares), but otherwise the only connection between staking and deposits are the deposit fees going to the bonus pool for the staking rewards. The deposit/withdrawl reserve bonding curve and the staking rewards are, in the eyes of the developers, logically isolated.

### Unnecessary duplicate code in staking functions

The staking functions of the Anon.Credit contract have non-trivial logic for the calculation of reward distribution. Some of this logic is duplicated in separate functions. For example, the `restake` convenience function performs the same logic as the `stake`, `addToStake`, and `unstake` functions. Separating this duplicate logic into internal functions would reduce the risk of unintended deviations in the behavior of each function.

#### Response

`restake` has been removed, you can accomplish the same thing with unstake & stake. The overlap between `stake` and `addToStake` has been double checked and organized to be visually identical.

## Informational

### Operator dual purpose

The operator is used for both managing the contract and receiving the developer fees. If the operator is a smart contract, such as the `Splitter` contract, it should implement all the necessary administrative logic as well as logic for handling ERC20 tokens. It is good practice to separate the account with access control permissions from the one receiving tokens in order to accommodate the use of advanced wallet security practices like cold storage.

#### Response

Upon successful deployment the operator is immediately set to the Splitter contract address, which has the same effect as setting it to the 0x00 address. The Splitter contract will only allow token withdrawals, so it will make it impossible for anyone to call `updateVerifier` or `changeOperator` again, and the Anon.Credit system will be immutable.

### Bonus rollover creates deadline for claiming bonus

The Anon.Credit contract implements reward rollover functionality in the `stakerCollectBonus` function. This functionality creates a potentially dangerous time dependency which requires stakers to redeem their credit tokens before the end of the bonus round otherwise their bonus is reallocated to the stakers in the next round. Since credit tokens are ERC20 compliant, it is likely that they will be used on external contracts such as uniswap. Since the credit tokens become valueless after the end of a bonus round, it is possible this will simulate a rug pull due to a race to remove liquidity on these third party platforms. It is important to either 1) make users aware they could get rekt by this unconventional mechanic or 2) remove the reward rollover in favor of no deadline on bonus collection.

#### Response

The rollover period is 1 year which doesn't seem dangerous to us. Having a reward claim deadline is intentional, because we would prefer not to burn rewards on lost private keys or dead stakers.

### Off-chain analysis

The Anon.Credit contract does not expose logs for the staking functions. This makes it difficult to craft a query like 'How much of the bonus was rolled-ever in each bonus round?'. To improve the ability to query this kind of historical information, the contract should implement events for each staking function that exposes all the relevant information. Additionally, the contract could have an additional field in the `BonusPool` struct called `bonusRolledOver` for storing the bonus balance that was rolled over.

#### Response

`bonusRolledOver` has been added to the `BonusPool` struct. Events for staking have been added as well.
