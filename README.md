# CULT DAO Governance Dashboard

Community-built dashboard for checking CULT DAO voting readiness and surfacing votes that were submitted on-chain but carried zero voting weight.

The main page is organized into these sections:

- **Delegation Checker**: paste one or more wallet addresses and check whether they are ready to vote. This does not require connecting a wallet.
- **Dashboard**: visual summaries for supply, participation, voting-power sources, missed voting power, delegatee reliability, and proposal risk.
- **Past Proposals**: searchable historical proposal cards with outcome status, summary donuts, details, zero-weight voters, and delegatee-duty rows.
- **Guardians Overview**: compares current contract guardian slots with the reconstructed current top-50 dCULT stakers.

Legacy aggregate tables are kept in the hidden Pre-retirement section for reference while the dashboard and proposal views replace them.

## Running Locally

Serve the folder over localhost:

```bash
python3 -m http.server 8000
```

Then open:

```text
http://localhost:8000/
```

Avoid opening the files through `file://`. Browser storage, static JSON loading, and some cache reads are more reliable through localhost or a published site.

## Publishing

This is a static site. Publish these files together:

- `index.html`
- `styles.css`
- `app.js`
- `delegation-checker.html`
- `delegation-checker-embed.html`
- `delegation-checker.js`
- `favicon.svg`
- `historical-cult-governance-data.json`

If using GitHub Pages from the repository root, place `historical-cult-governance-data.json` in the same folder as `index.html`. If publishing from `/docs`, place all files in `/docs`.

## Updating Historical Data

Historical proposal data is cached in the browser with IndexedDB. The same exported dataset can also include locally refreshed supply samples and the Guardians Overview. The menu has two relevant actions:

- **Rebuild index**: clears local cache and scans again.
- **Export static dataset**: downloads `historical-cult-governance-data.json` from the current local cache.

For publishing, rebuild/index locally, refresh optional cached sections such as Guardians Overview, export the static dataset, then replace the published `historical-cult-governance-data.json`. Visitors can load that static dataset into their own browser cache and only need live RPC for newer or missing data.

## Delegation Checker

The checker answers one question first: is this wallet ready to vote directly?

Common states:

- **Ready to Vote**: wallet has dCULT voting rights and is self-delegated.
- **Delegated Elsewhere**: wallet has dCULT, but voting power is assigned to another delegatee. Direct voting can waste the wallet's own vote attempt unless the delegatee votes.
- **Needs Delegation**: wallet has dCULT but no active delegate.
- **No Voting Rights**: wallet has no dCULT voting power. Empty/inactive wallets do not need delegation action unless they are meant to participate.
- **Guardian**: guardian wallets are protocol-special and not normal voters.

Expandable history shows direct vote-attempt waste and delegated-power missed duty when cached Wasted Votes data is available.

## Wasted Votes

### Potential Wasted dCULT

Sum of dCULT from zero-weight vote attempts on non-canceled proposals. A zero-weight vote attempt is a real on-chain vote transaction or signature vote where the governor counted `0` votes for that voter.

This is cumulative across proposals. The same wallet can appear more than once if it repeated the mistake on different proposals.

Formula:

```text
Potential Wasted dCULT = Wasted For + Wasted Against + Wasted Abstain
```

The cause buckets should also reconcile to the same total:

```text
Potential Wasted dCULT = No Voting Rights Waste + Delegatee Did Not Vote
```

### Wasted For / Wasted Against / Wasted Abstain

Splits the potential wasted dCULT by the voter's intended direction.

**Wasted Abstain** means the wallet submitted an abstain vote on-chain, but it carried `0` voting weight. It is included in Potential Wasted dCULT because the vote attempt happened, but it is not used for yes/no margin-flip checks because abstain is neither For nor Against.

### CULT Supply Distribution

This visual is broader token context, not a governance result. It samples the CULT token roughly once per week from launch using the same addresses as the hub and burntracker:

- **Total supply**: `CULT.totalSupply()` at the sampled block.
- **Treasury balance**: `CULT.balanceOf(TREASURY_ADDRESS)` at the sampled block.
- **Burned supply**: the two configured burn-wallet balances.
- **Staked dCULT**: `dCULT.totalSupply()` at the sampled block.
- **Uniswap pair CULT**: `CULT.balanceOf(UNISWAP_PAIR_ADDRESS)` using the hub CULT/WETH pair.
- **Free-floating CULT**: max supply minus burned, treasury, staked dCULT, and Uniswap pair CULT.

The chart uses max sampled CULT supply as the 100% y-axis ceiling. Burned supply fills the top of the frame; treasury, free-floating CULT, staked dCULT, and Uniswap pair CULT fill the circulating portion below it. Samples are cached locally under the normal IndexedDB cache branch, so only missing or older samples without LP balances are fetched on later loads.

### Wasted Vote Sources

The stacked chart shows cumulative missed voting power over non-canceled proposal history:

- **Direct wasted votes**: all direct zero-weight vote attempts with snapshot dCULT.
- **Repeat wasted**: the part of direct wasted votes from wallets that failed on more than one proposal.
- **One-time no rights**: one-time direct failures where the wallet had no usable voting rights at snapshot.
- **One-time delegate absent**: one-time direct failures where the wallet had delegated elsewhere and the delegatee did not vote.
- **Delegatee absent total**: all direct wasted votes caused by an absent delegatee, including repeat wallets. This is the orange overlay in the chart.
- **Absent delegatee duty**: full delegated representation missed by absent delegatees. This is stacked into this chart as a broader missed-power view.
- **Unclassified**: appears only when cached rows do not reconcile to the current cause buckets. Rebuild the index if this appears.

The chart's combined missed number is broader than the Potential Wasted dCULT summary box. Potential Wasted dCULT remains direct zero-weight vote attempts only.

The legend items can be toggled on/off to inspect smaller layers. Hovering the chart shows the cumulative source values at the proposal under the cursor.

### Proposal Voting Power Composition

The second chart shows each non-canceled proposal normalized to 100% of the currently visible series. Use the **Token weight / Wallet count** toggle to switch between dCULT voting power and wallet counts:

- **Proper votes**: token weight that actually counted on the proposal, or wallets represented by counted votes.
- **Direct wasted votes**: zero-weight vote attempts with snapshot dCULT, or the wallets behind those attempts.
- **Absent delegatee duty**: delegated voting power, or represented wallet count, not represented because the delegatee did not vote.
- **Eligible + ready**: delegated snapshot dCULT, or wallet count, excluding protocol-ineligible guardians. This layer starts hidden and can be toggled on from the legend.
- **Eligible staked**: total staked snapshot dCULT, or holder count, excluding protocol-ineligible guardians. This layer also starts hidden.

Legend values are average percentages across the plotted proposals. Hovering a proposal shows each visible layer in the active unit and as a share of the visible stack, eligible + ready base, and eligible staked base.

### No Voting Rights Waste

Vote attempts that were wasted because the voter did not have usable voting rights at the proposal snapshot. The row detail explains the cause, such as:

- no snapshot dCULT
- not delegated at snapshot
- no voting weight at snapshot

### Delegatee Did Not Vote

Vote attempts from wallets that were delegated to a third-party delegatee at snapshot, where the voter still attempted to vote directly and the delegatee did not vote. This is separated because the failure is not just "no delegation"; there was a responsible delegatee, but that delegatee did not represent the wallet on that proposal.

### Aligned / Misaligned Delegated dCULT

If a wallet voted directly with zero weight but its delegatee did vote:

- **Aligned** means the delegatee voted the same direction as the wallet attempted.
- **Misaligned** means the delegatee voted the opposite direction.

Misaligned votes are not counted as wasted voting power, but they are useful because they show voter intent being overridden by delegation.

### Delegatee Duty Voted

Shows how many delegatees with attached third-party voting power actually voted on proposals, plus how many represented wallets were covered.

### Absent Delegatee Power

Voting power delegated to third-party delegatees who did not vote. This is a separate layer from direct zero-weight vote attempts.

Do not simply add this number to Potential Wasted dCULT as if they were disjoint. They measure different failure surfaces and both are cumulative over proposals.

### Repeat / One-Time Fails

Compares wallets that wasted votes once versus wallets that wasted votes on multiple non-canceled proposals.

The first line is wallet count. The second line is vote-attempt count. The dCULT line compares token weight. This helps distinguish isolated mistakes from repeated behavior with larger impact.

### Net Margin Exceeded

Counts proposals where net wasted votes were greater than the actual margin. This is a warning metric, not a final legal claim that the proposal outcome must have changed.

### Passed/Defeated Margin Exceeded

Same idea as Net Margin Exceeded, but only for decided proposals: passed plus defeated.

### Absent Delegatee Impact

Counts proposals where absent delegated representation had enough voting power to matter:

- **flip**: absent delegated power exceeded the actual margin.
- **dominate**: absent delegated power was larger than the total active vote side it would be compared against.

The proposal cards use icons for this:

- Arrow icon: wasted direct votes would have flipped the proposal outcome.
- Dashed/solid user icon: absent delegated representation could have flipped or dominated.

## Delegatee Voting Power

This section looks at the latest non-canceled proposal snapshot and lists delegatee wallets that held third-party delegated voting power.

Each row shows:

- own voting power
- attached third-party power
- combined power
- represented wallet count
- whether the delegatee voted on the latest snapshot proposal
- historical duty record

The **Duty Record** shows how often the delegatee represented attached wallets across reportable proposals, how much delegated dCULT was missed, and the latest proposal/date where the delegatee fulfilled duty.

## Guardians Overview

This section reconciles two different guardian views:

- **Contract guardian slots**: reads `highestStakerInPool(0, index)` for slot indexes `0..49`.
- **Current top-50 dCULT stakers**: reconstructs current dCULT holder balances from dCULT `Transfer` logs, sorts by current dCULT amount, and takes the top 50.

The overview donut shows how many current top-50 dCULT stakers are inside versus outside the contract guardian slots. The right-side summary compares:

- **Guardian threshold**: `highestStakerInPool(0, 0)[0]`, the same threshold amount shown by the hub.
- **Top-50 floor**: the lowest current dCULT balance in the reconstructed top-50 holder list.
- **Gap**: the difference between the contract threshold and the current top-50 floor.

To update guardian status, stake some CULT or claim rewards and stake them.

Rows list the union of both sets. Detailed mode shows top-50 rank, guardian slot when present, wallet, first pool-0 dCULT staking deposit date (`staker since`), current dCULT, wallet CULT, current dCULT delegation status, and decided submitted proposals. Simple mode shows rank, wallet, current dCULT, and decided submitted proposals, with the extra row context available from the rank badge tooltip. Canceled proposals are excluded from this submitted-proposal count.

Use **Refresh Guardians** to rebuild this section. The result is saved in the normal cache and included in `historical-cult-governance-data.json` when exported.

## Proposal Details

Each proposal has two expandable areas:

- **Show Details**: voting totals, turnout, wasted vote totals, delegatee-duty metrics.
- **Show wasted wallets and third-party delegatee duty**: concrete zero-weight voter rows and delegatee duty rows.

Canceled proposals are hidden by default and excluded from delegatee-duty reports, because including canceled proposals can make missed-duty numbers misleading.

## Data Caveats

- This tool reads public Ethereum data and local/static cache data. It does not prove intent beyond submitted vote direction.
- dCULT balances and delegation are evaluated at proposal snapshot blocks.
- Public RPC providers may rate-limit or fail large scans. Wallet RPC or exported static data is more reliable.
- Cached data can be stale until rebuilt or refreshed.
- Cumulative dCULT totals are proposal-weighted: the same wallet's voting power can count again on every proposal where the issue occurred.

## Contracts

- Governor: `0x0831172b9b136813b0b35e7cc898b1398bb4d7e7`
- dCULT: `0x2d77b594b9bbaed03221f7c63af8c4307432daf1`
- Batch vote helper: `0x4aD54f4bb255529396Bd9506233d9Fb916A38975`
