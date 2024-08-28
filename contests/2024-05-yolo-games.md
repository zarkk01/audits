# YOLO Games
## Contest Summary
Code under review: [2024-05-yolo-games](https://cantina.xyz/code/a2c3cc6a-e384-495f-9751-5d7e657bc219/README.md)

Contest Page: [yolo-games-contest](https://cantina.xyz/competitions/a2c3cc6a-e384-495f-9751-5d7e657bc219)

Placement: #16/250

# Findings

# [M-01] Outdated kelly criterion-based max bet due to randomness block delay.

## Relevant Context
The Kelly Criterion is used to calculate the maximum bet in maxPlayAmountPerGame of every Game, but a three-block delay in randomness requests causes potential discrepancies due to changes in the LiquidityPool balance. This can result in suboptimal bets, impacting both the casino's profitability and the rewards for liquidity providers.

## Finding Description
Looking on play function of each Game, we understand that the calculation of the maximum betting amount is performed based on the pool's balance at the moment of the randomness request. However, due to a delay of minimum three blocks in receiving the randomness result, the actual balance of the liquidity pool can change before the bet is executed. This can happen due to other people gambling or deposits/redemptions to the LiquidityPool. This discrepancy can lead to bets being placed that are too large compared to the optimal amount calculated by the Kelly Criterion, potentially affecting the casino's expected value and risk management. Additionally, this results in lower profitability and reduced rewards for liquidity providers. Basically, the usage of kelly criterion is non-sense since the variables that it is depending on, can change before the bet.

## Proof of Concept
To understand better this vulnerability, consider this scenario :

The LiquidityPool initially holds 1000 ETH and max bet calculated using Kelly Criterion is (for example) 100 ETH. Alice calls play function and submits a Chainlink VRF request betting 100 ETH.
In the meantime and while the minimum 3 block confirmation is passing, a significant withdrawal occurs (the redemption request of it may have been submitted before the play call) , reducing the pool balance to 700 ETH.
When the bet is executed in fulfillRandomWords, the calculated max bet of 100 ETH is used, which is now disproportionate to the reduced pool balance.
Impact Explanation
The impact of this issue is high due to the following reasons:

Loss of rewards for LPs : Inaccurate betting limits lead to suboptimal performance of YOLO, which in turn reduces the returns for liquidity providers.
Financial Risk forYOLO: The casino is exposed to significant financial risk if the calculated max bet is based on outdated pool balances. This can result in larger than expected losses or missed profit opportunities.
Likelihood Explanation
The likelihood of this issue occurring is high. Given the dynamic nature of the LiquidityPool, especially in an active casino environment, balance changes over three blocks are very common.

## Recommendation
Implement a mechanism to recalculate the maximum bet based on the updated pool balance just before executing the bet. This can be achieved by checking the current balance in the same transaction where the bet is placed. Then, do NOT revert the bet, if the betting amount is bigger than the max, just limit it.