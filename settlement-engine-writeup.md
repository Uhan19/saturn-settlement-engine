### Assumptions:

- Saturn treasury and the processor wallet are both on-chain therefore funds can be automatically moved between them using an automation engine. (nuance: the engine can stage the treasury transfer, but in v1 a human still approves it in Fireblocks — method 1 is human-approved per the design doc)
- There exists an API layer with the brokerage accounts to help facilitate txns executed by the automation engine.
- Reasons why the queue structure exists: 1. Potentially needs to liquidate in order to provide funds (speed varies depending on whether it is from the treasury or external sell) 2. Instant redemptions might create an unfavorable condition for a "bank-run" 3. Oracle prices could possibly be stale, especially over the weekend. 4. Economies of scale provided by batching, also allows the flexibility to choose between 3 methods of providing USDat liquidity.
- No compliance constraints beyond the existing blacklist checks in the contracts.
- PROCESSOR_ROLE is a Fireblocks-held key and policy-based co-signing is available.

### Questions:

- Who should eat the potential "slippage" for method 3? let's say that the execution off-chain results in the `minUSDatReceived` to not be satisfied, do we mitigate the difference using the vault or treasury USDat reserves or do we pass the slippage to the depositor?
- How is `minUsdatReceived` calculated? Is it calculated by the FE?
- What API is available in the off-chain portion for Method 3?
- What's daily redemption volume look like?
- Is there a target usdatBalance, or is it just deposit float awaiting conversion? How much idle USDat is typically sitting there?
- How do we know that we need to accumulate STRC? what's the actual decision input? Target inventory, price level, cash position? These will affect the configs.
- How much control do we have with the Fireblocks policy engine?

So we have three different methods of processing requests:

### 1. Trade against our own backbook

We want to own STRC (possibly based on current STRC price, below some threshold, e.g. $90), we will send USDat from our own treasury (assuming this is on-chain) to the processor wallet and call `processRequests(tokenIds, totalUsdatReceived, totalStrcSold, executionPrice)` (note: `minUsdatReceived` is NOT a param here — it's set per-request by the user at `requestRedeem` time and the contract checks each request's pro-rata payout against its stored floor internally), and then in the brokerage accounts (assuming we have two different brokerage account for STRC between the vault and Saturn) we transfer the STRC that was "bought" by Saturn to the treasury's brokerage account.

#### Data that the engine needs as signal:

- STRC price oracle from chainlink as a possible evaluation threshold.
- Whether we should buy STRC, e.g. STRC price threshold, or a simple boolean that can be configured in a dashboard. (This should be top level check)
- Treasury USDat balance — do we actually have enough cash to fund the purchase.
- Engine should be able to directly interact with the processor wallet (Fireblocks), possible configuration needed on Fireblocks.
- The engine needs some sort of API from the brokerage side so that we can automatically transfer the funds from one brokerage account to another, otherwise this would need manual intervention.

### 2. Use the vaults idle USDat

Next in line in the redemption decision waterfall is to redeem through the idle stablecoin (USDat) that we have sitting in the vault. (to be clear on the ordering: method 1 only comes first because it's gated by the "we want STRC" appetite flag — when that flag is off, this is the default first rung.) This is a simple process where it looks like there's an existing function that we can call in the contract `convertFromUsdat`. The vault will decrement the USDat balance and increase the STRC balance in the accounting and send the USDat out to the processor wallet. Then the processor will then call `processRequests` the same way as method 1. The automation engine will decide to use this method based on a few different factors - We do not want to purchase STRC, and that the USDat balance in the vault is above a certain level. We have to be a bit more careful in this step with the automation engine, because the funds are just sitting in the processor wallet until `processRequests` is called and the funds are then pulled into the queue. During this transition step, if the engine crashes for whatever reason, we need to be able to recover the correct state when the fallback kicks in or on restart.

#### Data that the engine needs as signal:

- STRC price oracle from chainlink to pass into `convertFromUsdat`.
- USDat balance in the vault and whether it meets the configured threshold.
- All the same on-chain and API layer access as method 1

### 3. Sell STRC in the real market

This is presumably the last resort, unless we want to sell STRC because we believe it is a good opportunity to take in profit and accumulate cash (e.g. STRC trading above $100). (note: opportunistic selling like that is really treasury rebalancing, not redemption settlement — the contract has a separate function for it, `convertFromStrc`, where the processor brings USDat in and the vault's STRC book goes down. keeping that out of the settlement engine's v1 scope.) Otherwise this step will be triggered if the conditions for step 1 and 2 are not satisfied - not enough cash in treasury and vault. I am assuming that this should be an infrequent operation, perhaps a large whale is exiting. This step is going to be a bit more opaque as far as the automation engine is concerned. The broker must sell the necessary STRC position and that potentially has extra considerations (e.g. how good will the execution be?). This step will potentially also take a bit longer, I am assuming that the best settlement will be T+1, longer if it is over the weekend. The automation engine here has to take into account possible slippage between the `minUsdatReceived` and the execution that is performed off-chain.

#### Data that the engine needs as signal:

- STRC price oracle
- Execution price of the actual market sell needs to be checked right before submitting
- Tolerance re-check at processing time: `executionPrice` must be within `toleranceBps` of the oracle AND `totalUsdatReceived` within tolerance of `previewRedeem(totalShares)`, checked when `processRequests` lands — the 1-2 day gap between the sale and processing means a big weekend price move can make a perfectly good sale fail validation on Monday.
- On-ramp operations and completion
- All the same on-chain and API layer access as method 1 & 2

There are some critical things that the automation engine must take into consideration. No matter what method is taken to honor the redemptions, the contract call at the end is `processRequests`. The only difference is where the `USDat` funding comes from. So the automated settlement engine in its simplest form is really a router deciding among the three funding sources (backbook, idle cash in vault, or sell STRC off-chain). Because everything flows to `processRequests`, it must be noted that if any one request fails the `require(usdatAmount >= minUsdatReceived, SlippageExceeded())` check then the entire batch of requests will revert. So it is critical that the settlement engine checks every request's projected payout against its floor to ensure that it will not revert before including it in the batch.

### Settlement engine

#### Indexer

Seems like we are able to get a lot of the information with pending redemption requests through view functions like `getPendingIdsInRange` and use different multicalls through the engine to get an accurate read on all the requests and their statuses. Since right now we are focused on a higher level system design, I am going to assume that we can easily obtain this information without diving into the details of the implementation. We can run an indexer and write the request information to the DB, on-chain data will still act as the source of truth, but having a DB allows us more posterity in the data and less RPC overhead during ops.

For tracking what's pending: the DB keeps a pending set — a request gets added when the indexer first sees it (token ids are dense and sequential so scanning (0, nextTokenId) can't miss any), and it only gets removed when on-chain says that specific id is Processed/Claimed/seized. This means a request that failed during execution or was excluded from a batch just stays pending on-chain, so it stays in the DB and gets re-considered every tick — no separate retry logic needed. As a sanity check we compare the DB's pending count and sum of shares against the on-chain `getPendingCount()` and `getTotalPendingShares()` at the same block — if they don't match, something changed (e.g. a bug) -> alarm, do nothing this tick, and trigger a full rescan to rebuild the pending set.

#### Planner

This is one of the more critical parts of the settlement engine, it will be responsible for constructing the requests (tokenIds) that will be included in the batch. This step will need to balance between request age and size. It can optimize the settlement and batch requests so that the total sum reaches a certain amount before settlement is triggered. We can also build in logic to have a maximum daily settlement amount that can be dynamic and configurable. For example on weekends we can lower the total amount since the oracle price is more stale. Request age is also important in this case, we can make sure that no requests are pending for more than X amount of days and prioritize settlement even if it is uneconomical (could adopt a FIFO rule here). Perhaps a DB can be used to determine the number of days that it has been pending if the data is not available through on-chain sources. This definitely involves some balancing of priorities, because the minValue is presumably calculated with the price at request time, the longer that the request sits, the more potential the price has to drift.

Furthermore, the planner will need to simulate the requests through the `processRequests` call to ensure that the `minValues` are respected before it is included in the batch otherwise we risk the whole batch reverting during execution. It is important to note that simulation doesn't promise the transaction will succeed - it promises the transaction can't succeed incorrectly. For method 3 where the potential for drift is higher, we could build in additional headroom and lower total execution amount when calculating whether the request should be included, especially if it is going to land on a weekend.

#### Executor

After the planner builds the txn, the executor will submit the transaction via Fireblocks. We can also write to the database here with some txn details for posterity. I haven't done a deep dive into Fireblocks, but if it is anything like the MPC wallet providers that I am familiar with then we should be able to build in a lot of safe guards here. We can add txn policies to its policy engine and make sure that only approved transactions can happen, depending on the granularity allowed, we could set different amount caps for different operations. There might even be API co-signers that we can utilize to automatically sign certain txns that we deem to be lower risk to reduce the manual overhead. As far as the idempotency goes, the contract should be the guard. `processRequests` will flip the status to Processed, so no tokenId/requests can be processed more than once. For functions like `convertFromUsdat` where idempotency is not guaranteed, we could use a couple ways to build this into the engine. First we can use the db write that we perform with every txn submitted and check against a txn hash that is saved and its status to ensure that no duplicate txn is submitted. Second, we can build this layer into Fireblocks, there should exist a txn ID and duplicate txns should not be able to be created (need to verify this).

#### Risk module

We could have separation of concerns for the risk and create a risk module that sits between the Planner and the Executor, it could make sure that any caps are enforced (single-batch, daily, weekly, limits). Caps can be dynamic as well as mentioned earlier and the risk module could help enforce that. It could also decide which method requires human intervention or approval. Furthermore, it could act to check oracle health, is price moving at a rate that is unexpected and we want to halt the engine and or send alerts to a human to loop them in. It could also reconcile any DB state, as mentioned before, if there's a mismatch between total pending requests in the DB vs what is on-chain then we make sure that alerts are sent or we re-query the state on-chain. Any kill switches would live here as well. Ideally, some of what is included in the risk module will be defense-in-depth, since as much as possible I would like to include the safe guards in Fireblocks policy engine.

#### Method 3 off-chain tracker

This is perhaps the most opaque part of the engine right now. Since we'd have to check to see what API is available so that as much of this progress is updated automatically as possible. For each state transition during this step we would either use automatic API verification, on-chain state changes, or some sort of manual input as signal. We can have these actions write to state (DB) so that the policy engine can monitor and submit txns when ready. For example:

```
PLANNED → LOCKED → SELL_ORDERED → SOLD → PROCEEDS_SETTLED
        → OFF_RAMPED → USDAT_MINTED → FUNDED → PROCESSED → RECONCILED
```

It can be much simpler than this too, basically just something that mirrors the state off-chain. When API is not available, we can have a human input (e.g. txn id, click a button) and it would move to the state, then we can have the policy engine monitor the state and perform any actions necessary (e.g. send alerts/reminders)

The last step of the method 3 tracker is a three-way tie out before we can close out the settlement: broker records vs actual cash movement vs on-chain amounts. The broker tells us how much STRC was sold and at what price (e.g. 1,800 STRC @ $99.80 = $179,640), the cash side tells us how much actually arrived and got minted (e.g. $179,600 USDat after a $40 wire fee), and the chain tells us what `processRequests` recorded (`totalStrcSold`, `totalUsdatReceived`, and the vault's strcBalance decrement). All three have to agree with every difference explained. If anything doesn't match then the settlement stays open and potentially send a warning. We never auto-close with an unexplained discrepancy, and a settlement that stays open past its expected duration is itself an alarm.

#### Tradeoffs

1. **Router over Optimizer** - The proposed engine is really just a router rather than an optimizer. But this might be a really good place to build an AI agent that helps optimize some of the decisions. But we would need to make sure that we build the right tools with the right parameters to validate the result of the AI agent, but depending on our growth and demand this could be very beneficial. Of course we would have to ensure that the benefits provided outweighs the token-cost.
2. **DB over an off the shelf workflow platforms** - During my research and brainstorming for this assignment I came across Temporal (https://docs.temporal.io), this could genuinely be a better solution for the workflow of distributed systems that we are aiming to create here, but feels a bit like an overkill at the moment, and I would need to do more of deep dive to understand the fit. Right now using a DB and traditional cloud services should suffice.
3. **Settlement Router** - we could build an on-chain component that does some of the things that the settlement engine is currently doing. For example, we could combine `convertFromUsdat` and `processRequests` into one function so that it is atomic. But we would have to get the contract audited which might take a little too long.
4. **Polling vs Webhook/subscriptions** - settlement engine built on events is probably more efficient but we run the risk of missing events and the ramification of that might not be preferable to polling in a set amount of time. Since there is little need for instant execution polling should suffice.

Least confident areas:

1. The method 3 off-chain leg — I don't know what APIs the broker/on-ramp actually expose, so how much of the tracker can be automated vs needing manual input is a guess until we see them.
2. Fireblocks policy engine granularity — I am assuming we can set per-operation caps and co-signer rules, need to verify how much of the risk module can actually be pushed down into it.
3. The actual numbers — caps, thresholds, and request age limits are all placeholders until we see real redemption volume.

#### Risks and controls

| What goes wrong                                                                                                        | What stops it                                                                                                                                                                         | Worst case if it happens anyway                                              |
| ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| Engine computes wrong payout amounts                                                                                   | Simulation before staging; the contract independently re-validates prices, totals, and every minValue at execution time                                                               | One batch, wrong by at most the tolerance band, capped by the batch limit    |
| Engine runs away or is compromised                                                                                     | Spending limits on the Fireblocks key (independent of engine code); three kill switches — engine off-flag, revoke the Fireblocks rules, pause the contract — any one works on its own | One day's automated settlement limit                                                    |
| Same STRC sold twice at the broker                                                                                     | Client-order IDs; ask the broker what it has before ever re-sending                                                                                                                   | One order's size, caught on the next cycle                                   |
| Bad or stale price feed                                                                                                | Cross-check Saturn's feed against Chainlink's; disagreement halts automated settlement; the contract's staleness and bounds checks remain underneath                                         | Settlement pauses; nothing executes at a bad price                           |
| Weekend gap risk: market closed, feed still reports Friday's close, informed users exit before an expected Monday drop | Much smaller automated settlement limit while the market is closed; large batches wait for market hours                                                                                          | Bounded by the weekend limit                                                 |
| A user's minValue makes the whole batch revert                                                                         | Planner simulates first and excludes requests that can't clear; after locking, mins can only move down                                                                                | One reverted transaction; retry next cycle without the offender              |
| Method 3 stalls mid-flow                                                                                               | Every state has a time limit with an alert; a heartbeat alarm fires if the engine itself stops ticking                                                                                | Lost time, not lost funds — at every state the money sits in a known account |
| Postgres is lost                                                                                                       | Managed backups; chain state is re-readable and broker state re-queryable, so the database mostly caches what can be rebuilt                                                          | Settlements pause during restore                                             |
| Two copies of the engine run at once                                                                                   | A lock in Postgres ensures only one may act; the other can only observe                                                                                                               | None, if the lock holds — that is its entire job                             |
| Method 2 drains the vault's idle USDat                                                                                       | A configured floor the planner will not cross; alert on approach                                                                                                                      | Slower settlement (falls through to Methods 1/3), never a broken vault       |

The design philosophy: **every failure should end with the engine doing nothing — never with the engine doing the wrong thing.**

#### 30/90 day plan

The rollout plan is laid out visually in the companion file (`settlement-flows.html`, rollout section). Short version: v1 ships in stages — shadow mode first (engine runs and proposes but executes nothing, we compare its plans against what actually gets done manually), then co-pilot (engine stages the txns, a human approves each one), then automated settlement for method 2 only under strict caps (single-batch, daily, and market-closed limits). Methods 1 and 3 stay human-approved in v1. Post 90 days: raise the caps as trust builds, broker API integration for method 3, and potentially the on-chain Settlement Router from the tradeoffs section (pending audit). Explicitly not in v1: opportunistic STRC selling, multiple in-flight settlements, and the AI optimization idea.
