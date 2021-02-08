# Smart Yield

# General Architecture

We propose a system where there are 2 types of participants:

- Junior token holders
- Senior BOND holders

Junior token holders provide liquidity and buy risk from Senior bond investors. The risk in the case of Smart Yield product is the risk of variable rate annuities going down.

Investors that buy the Senior bonds (sBONDs) will have a guaranteed yield for the life of the sBOND, which will be one of the 2 main properties of the sBOND (along with TVL).

The liquidity provided by the "juniors" and the value locked in the sBONDs are invested in various DeFi lending platforms and the resulting rewards cover the guaranteed reward for senior sBONDs. Some of these rewards come in the form of governance tokens which our system will sell on Uniswap for the native token of our pool. In this document, we will consider Compound as our lending platform and DAI as our pool token.

Juniors will benefit from the extra rewards generated by liquidity (DAI) locked in sBONDs by seniors in situations where the variable APY of Compound (including the COMP rewards) are higher than the weighted average guaranteed yields of current sBONDs. On the other hand, in the event of falling rewards from Compound, the returns of Juniors are diminished and if need be, their locked funds will be used to pay for the guaranteed returns of sBONDs.

In order to provide the best UX for Juniors and encourage them to participate in our Smart Yield pools, we want the system to allow them to join the pool at any time. Moreover we want the possibility of instant withdrawal of at least part of their funds, without affecting the integrity of the system and keeping the guarantees.

To do that, we have to be able to calculate the profits and losses of the pool very efficiently. We do that by averaging all existing sBONDs into one weighted average sBOND with the following properties:
- Principal (DAI) locked (sum) -> ABOND.principal
- Guaranteed rewards (DAI, sum) -> ABOND.gain
- Start timestamp -> ABOND.issuedAt
- Weighted average end timestamp -> ABOND.maturesAt

### The Weighted average maturesAt is calculated in the following way:

#### new sBOND added
`newMaturesAt = (abond.maturesAt * current_debt + new_sBOND.maturesAt * new_sBOND.gain) / (current_debt + new_sBOND.gain)`


newIssuedAt is calculated in a way that the paid portion of the new ABOND (ABOND.paid) is equal to the paid portion of the old ABOND.

`newIssuedAt = newMaturesAt - (1 + ((abond.gain + new_sBOND.gain) * (newMaturesAt - now) / (current_debt + new_sBOND.gain)`

`abond.gain += new_sBOND.gain`
`abond.principal += new_sBOND.principal`

#### old sBOND redeemed after maturity date

Because the entirety of the sBOND is in the paid portion of the ABOND, the maturesAt property of the ABOND will not be changed (i.e. debt remains the same).

`newIssuedAt = abond.maturesAt - (1 + (abond.gain - old_sBOND.gain) * (abond.maturesAt - now) / abond.debt`

`abond.gain -= old_sBOND.gain`
`abond.principal -= old_sBOND.principal`

#### Calculating the elapsed portion of an ABOND's gain

`abond_duration = abond.maturesAt - abond.issuedAt`

`abond.paid = (ABOND.gain * min(now - abond.issuedAt, abond_duration)) / abond_duration`

`abond.debt = abond.gain - abond.paid`

total_duration = wa_end - wa_start
elapsed_time = block.timestamp - wa_start

The aggregate sBOND (ABOND) represents the current Senior pool and it will help us calculate the health of the Senior pool at an instant. If the rewards generated by the Senior pool so far exceed the guaranteed rewards at this time (abond.paid), the extra reward can be considered profit for the junior pool (and loss if it's negative).

To keep track of profits and losses of Junior Token holders (jTokenDAI, if the provider is compound), we update the exchange rate of jTokenDAI to DAI before certain events (juniorDeposit, juniorWithdraw).

jTokenDAI_to_DAI_ratio (jToken price) start at 1 and it's updated in the following way:
`jTokenDAI_to_DAI_ratio = total_junior_pool_dai / total_jTokenDAI_supply`

Where
```
total_junior_pool_dai = total_pool_dai - abond.principal - abond.paid
```

`total_pool_dai = total_dai_holdings - owed_dai`

`owed_dai` is anything owed to juniors after liquidation events plus fees accrued owed to the DAO.

total_pool_dai includes the unrealized DAI profits, which includes the COMP rewards given to the pool priced in DAI and periodically sold for DAI on Uniswap and added to the pool.

To prevent an arbitrage attack, the pool calculates the moving average price of COMP.

![](https://i.imgur.com/umIL8gy.png)

The shaded areas are the elapsed guaranteed part of the aggregate sBOND (abond.paid). As discussed above, we calculate the current profit and loss of the Junior pool based on this.

In order to keep the jToken price unchanged after a new sBOND is created or when a sBOND is redeemed (the 2 events that creates a new ABOND based on the old ABOND and the respective change), the blue shaded area should be equal to the black shaded area.


## Junior Deposit
Steps for deposit (in order):

- recalculate jToken price
- withhold the pool fee
- mint jTokenDAI proportional to the new price and transfer to Junior
- exchange deposited DAI to cDAI

## Senior deposit (buy BOND)
When an sBOND is created, a fixed guaranteed yield is calculated for it from on-chain data. The fixed rate is based on the moving average APY of the pool's provider, in this example Compound. We track the NET APY of Compound in a Smart Contract and calculate its moving average. This moving average includes the sold COMP rewards given by compund through our pool's public `harvest()` function. Calling it is incentivized by sending a portion of the proceeds to msg.sender.


### Calculating the sBOND yield

The pool offers a reward rate of `(junior_loanable_liquidity / total_pool_liquidity) * moving_averate_yield_rate`. The more junior liquidity in the pool (vs abond.principal), the higher the offered rate for new senior BONDs is.

In order to take into account the slippage caused by creating a new sBOND, we first calculate the approximate gain offered at current pool composition:

`approxGain = compound(
    principal,
    (dailyRate * junior_loanable_liquidity) / (total_pool_liquidity + principal),
    duration_in_days) - principal`

`dailyRate` is reported by our oracle and is the 3-day moving average value.

After we have the approxGain, we get the final rate for the new sBOND:

`rate = dailyRate * (loanable - approxGain) / (total + principal)`

`sBOND.gain = compound2(principal, rate, duration_in_days) - principal`

- Calculate the guaranteed rate based on above
- Add principal to aggregate sBOND
- Add guaranteed reward to aggregate sBOND
- Recalculate aggregate sBOND (debt) weighted maturesAt
- Recalculate ABOND issuedAt
- Allocate the sBOND the the Senior
- Exchange Senior deposit DAI to cDAI

## Senior withdraw (redeem sBOND)

Steps:
- Require current_timestamp > sBOND.end_timestamp
- Require sBOND.owner == Senior
- Substract sBOND.principal from aggregate BOND
- Substract sBOND.reward from aggregate BOND
- Recalculate aggregate sBOND issuedAt
- Exchange enough cDAI to pay the BOND's initial DAI + guaranteed reward (gain)
- Transfer the DAI to senior
- Delete sBOND


## Junior Initiate Withdraw
A junior is subject to Senior tranche risk. In order for a jToken holder to exit the pool, the current sBONDs need to mature. Because it is computationally intensive and gas inefficient to track each and every sBOND with each tx, we opt to use the Aggregated Bond (ABOND) as an approximation for the Senior tranche.

Steps for exit:
1st step:
- recalculate price
- transfer Junior jTokenDAI to pool (lock)
- mint a junior BOND token (NFT) and transfer to Junior
- queue liquidation after ABOND.maturesAt. First tx after that timestamp will trigger the liquidation.

2nd step:
After the lock period has ended, the Junior can redeem the jBOND NFT  and finish the Withdraw process:

Steps:
- send junior's liquidated DAI
- burn the jBOND NFT


The 2-step process allows a user to first signal the intention of exiting the pool. In return they receive a non-transferrable NFT that can be redeemed after the current ABOND ends. The first tx after ABOND.maturesAt triggers the liquidation for all jTokens scheduled for liquidation. The user can come anytime after ABOND.maturesAt to withdraw their part of the liquidation.

All jTokens locked after the 1st step are excluded from the junior pool when calculating new sBOND rates (`junior_loanable_liquidity`), but they will be subject to profit and losses of the pool until the maturity date.

Each 2 step withdraw initiation will create a jBOND NFT with a maturity date for the second step. The second and final step of withdraw, can be called publicly by anyone and it will burn the NFT and send the owed DAI to NFT's owner.

`sellTokens()` allows juniors to instantly withdraw their unlocked DAI. The locked part of each jToken is burnt in favor of the existing jToken holders, increasing the price. Effectively using this function, a junior forfeits their locked DAI. This allows instant arbitrage opportunities between our pool and secondary markets where jTokens are traded, like Uniswap, creating a floor for the jToken price.

```solidity
// share of these tokens in the debt
uint256 debtShare = jTokensToSell * 1e18 / totalJTokenSupply;
// debt share is forfeit, and only diff is returned to user
uint256 sellProceeds = jTokensToSell * price - abond.debt * debtShare;
```


## Pool overview
![](https://i.imgur.com/lNdoOL8.jpg)


# Fee Structure
On the Senior side, fees are collected after maturity date, when sBonds are redeemed.

On the Junior side, fees are collected on jToken purchases (jTokenDAI in the example above).

The percentage of fee that goes to BBDAO treasury is set by the BBDAO.


# Secondary Markets
The Junior Tokens are ERC20 compatible since they are fungible. Secondary markets can easily be created for these (e.g. Uniswap). This will allow jToken holders that need liquidity to exit quickly, probably with a discount.

The Senior BOND tokens (sBOND) are non-fungible (NFTs), but transferrable. One could sell these tokens before their maturity date at a discount to someone who is willing to wait until the end to claim the principle plus the guaranteed reward.

BarnBridge will build a marketplace for selling and buying of senior tokens. The pool is permissionless so others can build marketplaces as well.

# Junior-Side Incentivization
To further incentivize participation by Juniors and to bootstrap the Smart Yield product, c_bbDAI tokens (and other Junior pool tokens) can be locked in our YieldFarming pools for weekly BOND rewards.

The Smart Yield YF pool will accept all Junior Tokens created by BarnBridge protocol. To keep the scale equal, the default `jTokenDAI_to_DAI_ratio` value of each pool will be equal to the current price of the underlying asset (1 in case of stablecoins).
