### Protocol context
Cork protocol aims to create a new defi primitive to allow to trade depeg opportunities. A depeg swap allows market participants to buy a guarantee that a pegged asset its in fact exchangable from its base asset, the design of the protocol allows any pegged asset ( usde, stEth… ) to create a depeg swap it can be used to hedge exposure ( cover fro a depeg event in a delta-n like strategy ) or speculate on a depeg event.

How does cork protocol work

The protocol its composed of 4 types of tokens [Redemption asset, cover token, depeg swap, Pegged asset], and 2 principal modules the PSM and the Liquidity vault.

### ROLES

Config role -> can perform some admin functions and has the ability to Initialize PSM modules and Lvvaults so only pre approved tokens can be used
Users -> can interact with the protocol PSM and vault modules

### MAIN FUNCTIONS

Users can deposit Ra in the psm and then claim it back by pairing cs with ct, also they can clim it back pairing ds with Pa. On expiry users can provide CT and get back Ra and pa left in the psm module. Also users can deposit Ra to the vault in exchange for lv tokens, this lv tokens can then be used to claim accrued ra and pa on expiry in the vault, I need to still understand how does exactly the protocol earn that fees and the incentives of all the parts

A user can deposit redemption assets (aka ETH) to the psm and get back ds + ct that later can be redeemed back for the asset.

When users mint lv tokens a ds fee is sent to the routerState contract, also the same amount of ct goes to the ct/ra pool.
Then the router state allows users to buy and sell depeg swaps with the functions swapDsForRa and swapRaFords, this functions utilizes flash swaps under the hood. Also when users buy ds the router state makes a deposit of RA to the psm and when users sell Ds it claims ra from the psm using the claimerwithctds. All of that behavior creates fees for the lv_vault as this vault has a big part of the lp amm positions.

### TODO

- [x] Tirar del hilo de las 10 principales funciones del protocolo

Invariant flow
1st deploy smart contracts and exclude non callable addresses from the fuzzSelector aka ( config contract )
2nd create actorManager aka contract to create operators of contracts
3rd create actorOperatorContract aka contract that performs the logic with the protocol and add its selectos to the fuzzer
4rd do the same with timeWaper contract
5 create invariant\_ functions to check for state variables in that case lets check the balance for example

### Questions
How does the protocol know how much DS and CT to give a user in a deposit? Using the initial exchangeRate setter up on a new issue
What does exchange rate do in the asset tokens?? Defines de amount of tokens to give given an Ra amount
Where in the protocol does Pa take place what are the profit mechanism in the vault? Pa is obtained trough being deposited in the psm with depeg swap
Pa is deposited to the psi paired with ds to claim back Ra,
What happens at expiry, what occurs with lv holders?
What are all the ways for a user to redeem its initial Ra?
How a psi, vault are created specially por rebasing tokens, admin funds?
Each pair has a different psi and vault contract? -no but foreach “pool” a ct, ds and lv tokens are deployed 🟢
What happens when a CT and a DS token expires? can that create lock or loss of funds?
How does pricing occur in redeem early etc ? What does printing imply to the protocol?
How does the value earn money, how its exactly the process of it?

### Ideas

The math in the protocol I think that can have rounding errors as no libs are used so rounding error can occur, also look the trace of newly open markets when numbers are easily to manipulate. Ej force feeding the contract to manipulate rates etc
Everything related with DOS on assert functions
The protocol has some weird ways of keeping track of accounting, as well as some crucial math funds that can lead to some errors and a minimal error on how that state is managed or calculated can lead to a loss of funds or DDOS attack of the protocol as there are important assertions in some of the main functions of the protocol. Also I’ve seen some repeated blocks of logic in some parts of the contract I would need to understand better uniV2 to exactly get what the flash swap is doing etc
MEV problems when interacting with amm
### INVARIANTS
Play with invariants to see if protocol breaks, the balances tracks etc
Internal balances <= erc20 balance
Ds + ct = ra

### AUDIT REVIEW LEARNINGS
Have always in mind attack vectors where mev its related specially when integrating the protocol with a swap platform etc, here 3 highs were due to that kind of error.
I was able to find the root of the problem but had trouble finding specific situation that would lead to an attack, almost got a high now lest keep trying in future audits with the aquired knoledge.
Im happy with the result as this is the first audit i participate in.

H1 - no slippage protection in vaultLib:redeemEarly where \_liquidatePartial is called with 0 tokensOut param. The impact its that an attacker could frontRun the transaction and manipulate the price such that the procolol transaction bears heavy slippage leading to loss of funds.

H2 - FlashSwapRouter::emptyReserve() return incorrect values

H3 - self sandwich attack as no slippage protection its in the function swapRaForDs an sandwich attack can be executed incurring in protocol loss

Logical and math errors
H4 - incorrect redeemAmount is accounted due to not accounting the exchangeRate
In that case the internal accounting of the protocol fails due to not have in mind the exchangeRate so its an implementation error, in the future to find this errors its crucial to understand well the protcol and maybe focusing on a state variable at a time.
