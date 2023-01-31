# OpenQ contest details

- Join [Sherlock Discord](https://discord.gg/MABEWyASkp)
- Submit findings using the issue page in your private contest repo (label issues as med or high)
- [Read for more details](https://docs.sherlock.xyz/audits/watsons)

# Resources

- See "About OpenQ" below
- To run tests, run:

```bash
mv .env.sample .env
yarn
yarn test
```

- To deploy contracts, run:

```bash
yarn
yarn ethnode (in one terminal)
yarn deploy-contracts:localhost (in another terminal)
```

- Optionally, you can run this code block to both deploy all contracts, and fund bounties by running:

```bash
yarn deploy-contracts:localhost
yarn configure-whitelist:localhost
yarn deploy-bounties:localhost
yarn fund-bounties:localhost
```

# On-chain context

```
DEPLOYMENT: polygon mainnet
LANGUAGE: solidity 0.8.17
ERC20: any
ERC721: any
ERC777: any
FEE-ON-TRANSFER: any
REBASING TOKENS: any
ADMIN: restricted for upgrades, trusted for other admin functionality described below
EXTERNAL-ADMINS: oracle, claimManager, depositManager, openQ, issuer/minter of a bounty
```

In case of restricted, by default Sherlock does not consider direct protocol rug pulls as a valid issue unless the protocol clearly describes in detail the conditions for these restrictions.

For contracts, owners, admins clearly distinguish the ones controlled by protocol vs user controlled. This helps watsons distinguish the risk factor.

## Roles

There are several roles that interact with all OpenQ contracts:

- `owner`
- `oracle` (an off-chain signer hosted on [Open Zeppelin Defender Relay](https://docs.openzeppelin.com/defender/relay))
- `claimManager` (a proxy address)
- `depositManager` (a proxy address)
- `openQ` (a proxy address)
- `issuer/minter of a bounty`
- `any external user`

Additionally, time is an actor:

- `timelock`

Finally, there is one third party contract, [kycDAO](https://kycdao.xyz/) with some bearing on our security

- `kyc dao`

kycDAO however is not part of this audit, and only their interfact `IKycValidity.sol` is included in this repository.

## OpenQV1.sol

* `OpenQV1.sol` is owned by the protocol.

* `OpenQV1.sol` is UUPS Upgradeable and therefore should only be initialized ONCE and thereafter always called via `OpenQProxy.sol`.

#### `owner` in `OpenQV1.sol` should be the SOLE CALLER of these methods:

- `setBountyFactory`
- `setClaimManager`
- `setDepositManager`
- `_authorizeUpgrade`
- `transferOracle`

#### `oracle` in `OpenQV1` should be the SOLE CALLER of this 1 method:

- `associateExternalIdToAddress`

#### Any `external user` in `OpenQV1` can call:

- `mintBounty`

## DepositManagerV1.sol

* `DepositManagerV1.sol` is owned by the protocol.

* `DepositManagerV1.sol` is UUPS Upgradeable and therefore should only be initialized ONCE and thereafter always called via a `OpenQProxy.sol`.

#### `owner` in `DepositManagerV1.sol` should be the SOLE CALLER of these methods:

- `setTokenWhitelist`
- `_authorizeUpgrade`

#### Any `external user` in `DepositManagerV1.sol` can call:

- `fundBountyToken`
- `fundBountyNFT`

#### A `deposit` should remain `timelocked` in the target bounty for the entire expiration period. Withdrawls should not be possible before that block timestamp.

## ClaimManagerV1.sol

* `ClaimManagerV1.sol` is owned by the protocol. 

* `ClaimManagerV1.sol` is UUPS Upgradeable and therefore should only be initialized ONCE and thereafter always called via `OpenQProxy.sol`.

#### `owner` in `ClaimManagerV1.sol` should be the SOLE CALLER of these methods:

- `_authorizeUpgrade`
- `transferOracle`
- `setOpenQ`

#### `oracle` in `ClaimManagerV1.sol` should be the SOLE CALLER of this method:

- `claimBounty`

#### The `external user` is able to call the following method on `ClaimManagerV1.sol` in order to attempt a claim if they've been designated as a winner by the `bounty issuer` by calling `OpenQV1.setTierWinner`:

- `permissionedClaimTieredBounty`

#### The `external user` calling `permissionedClaimTieredBounty` should only be able to make their claim after completing the following criteria:

- if the `external user` has associated their wallet with their external user id by going through our oracle process defined off-chain and only callable by our oracle using `OpenQV1.associateExternalIdToAddress`
- if `bounty.tierWinner(closer)` returns true. This should only occur if the `bounty issuer` set their `external user id` as the winner of that tier's index by calling `OpenQV1.setTierWinner`
- if `bounty.invoiceRequired` is true, `invoiceComplete` must be set to `true` for that user's external id for that tier. if `bounty.invoiceRequired` is false, no requirement
- if `bounty.supportingDocumentsRequired` is true, `supportingDocumentsComplete` must be set to `true` for that user's external id for that tier. if `bounty.supportingDocumentsRequired` is false, no requirement
- if `bounty.kycRequired` is true, then `ClaimManagerV1.hasKyc`, which calls KYC DAO contracts OUTSIDE THE SCOPE OF THIS AUDIT, should return true. if `bounty.kycRequired` is false, no requirement

* NOTE: The reason for having `claimBounty` callable by off-chain oracle and `permissionedClaimTieredBounty` is because certain bounty types other than contests allow for configurable off-chain oracles for customized claim logic.

## BountyFactory.sol

* `BountyFactory.sol` is owned by the protocol.

* The `OpenQ proxy address` should be the SOLE CALLER of the `BountyFactory.mintBounty` method.

* `BountyFactory.sol` is not upgradeable. If an update is needed, we would simply re-deploy a new BountyFactory and set it on OpenQ with the only-owner `setBountyFactory` method. We could then direct it to the same `BountyBeacon` for minting `BeaconProxy`s.

## BountyV1 of all types (AtomicBountyV1, OngoingBountyV1, TieredFixedBountyV1, TieredPercentageBountyV1)

#### only the `OpenQ proxy contract` should be able to call the following methods on all relevant (i.e. implementing these methods) bounty types:

- `setKycRequired`
- `setFundingGoal`
- `setSupportingDocumentsRequired`
- `setInvoiceComplete`
- `setSupportingDocumentsComplete`
- `setTierWinner` (only on Tiered contracts)
- `setPayoutScheduleFixed` (only on TieredFixed contracts)
- `setPayoutSchedule` (only on TieredPercentage contracts)

#### only the `ClaimManager proxy contract` should be able to call the following methods on all relevant (i.e. implementing these methods) bounty types:

- `claimTiered`
- `claimTieredFixed`
- `closeCompetition`

#### only the `DepositManager proxy contract` should be able to call the following methods on all relevant (i.e. implementing these methods) bounty types:

- `receiveFunds`
- `refundDeposit`
- `extendDeposit`

#### The `minter/issuer` should be the initial address that called OpenQV1.mintBounty.
* This is stored as `bounty.issuer`, and grants that `external user` certain privileges, seen below.

#### The `issuer/minter` of the bounty should be the sole user able to call the follwoing methods on `OpenQV1`, which checks bounty state in its `require` statements guarding each of these:

- `setTierWinner`
- `setFundingGoal`
- `setKycRequired`
- `setInvoiceRequired`
- `setSupportingDocumentsRequired`
- `setInvoiceComplete`
- `setSupportingDocumentsComplete`
- `setPayout`
- `setPayoutSchedule`
- `setPayoutScheduleFixed`
- `closeOngoing`

#### private storage variables `_claimManager` and `_depositManager` should NOT be changeable after `initialize` is called on any BountyV1 type. This is defined in `ClaimManagerOwnable` and `DepositManagerOwnable`.

* `BountyV1.sol` is, however, lying behind a Beacon Proxy, and each Beacon Proxy held on BountyFactory is owned by the protocol.

## BeaconProxy.sol

#### `owner` of each bounty type's BeaconProxy should be the SOLE ACTOR able to call this 1 method defined in the Open Zeppelin `UpgradeableBeacon` :

- `upgradeTo`

* Note: This `onlyOwner` requirement is built into Open Zeppelin's `UpgradeableBeacon`, which is `Ownable`

## OpenQTokenWhitelist.sol

#### `owner` of `OpenQTokenWhitelist.sol` should be the SOLE CALLER of the following methods:

- addToken
- removeToken
- setTokenAddressLimit

* Because certain bounty types accept any ERC20 in order to be crowdfundable, and payouts occur in for-loops of `safeTransferFrom`'s in `OpenQV1.sol`, OpenQTokenWhitelist's purpose is to limit the amount of token addresses able to deposit on a bounty in order to avoid OUT_OF_GAS attacks.

# Audit scope

Includes:

- All contracts in `/contracts`, **EXCLUDING** the `Mocks` directory

Excludes:

- Any off-chain services, like our oracles which are all running in Node.js on [Open Zeppelin Defender Autotask](https://docs.openzeppelin.com/defender/autotasks).

# About OpenQ

# OpenQ Audit 👨‍💻🥷👩‍💻

Hello! Thank you for hacking OpenQ.

Give it your all, pull no punches, and try your best to mess us up.

Here's everything you need to get started. Godspeed!

## About Us

OpenQ is a Github-integrated, crypto-native and all-around-automated marketplace for software engineers.

We specialize in providing tax-compliant, on-chain hackathon prize distributions.

## How People Use Us

You can read all about how OpenQ works from the user's perspective by reading our docs.

https://docs.openq.dev

## Trusted Services

- Assume OpenQ Oracles are trusted
- Assume KYC DAO is secure, and only has NFTs for addresses which have undergone their KYC process

## Architecture Overview

### Upgradeability

#### UUPSUPgradeable

`OpenQV1.sol`, `ClaimManagerV1.sol` and `DepositManagerV1.sol` are all [UUPSUpgradeable](https://docs.openzeppelin.com/contracts/4.x/api/proxy). 

The implementation lies behind a proxy.

#### Beacon Proxy

All bounty types, `AtomicBountyV1.sol`, `OngoingBountyV1.sol`, `TieredPercentageBountyV1.sol`, `TieredFixedBountyV1.sol` are also upgradeable.

Because we have MANY deployed at any one time and want to be able to update them without calling `upgradeTo()` on each contract, we use the [Beacon Proxy pattern](https://docs.openzeppelin.com/contracts/3.x/api/proxy#beacon).

Each bounty contract lies behind a proxy. That proxy gets it's target for `delegatecall`'s from the appropriate beacon set during minting on `BountyFactory.mintBounty`.

#### Bounty Factory

Bounty Factory holds Beacons for all 4 bounty types as storage variables on it.

When it mints a bounty, it passes to it the appropriate beacon.

Since each bounty is a [BeaconProxy](https://docs.openzeppelin.com/contracts/3.x/api/proxy#BeaconProxy).

## Developer Perspective on All OpenQ Flows: Minting, Funding and Claiming

## Contract Types

OpenQ supports FOUR types of contracts.

Each one differs in terms of:

- Number of claimants
- Fixed payout (e.g. 100 USDC), percentage payout (e.g. 1st place gets 50% of all escrowed funds), or whatever the full balances are on the bounty (funded with anything, full balance paid to claimant)

The names for those four types are:

- `ATOMIC`: These are fixed-price, single contributor contracts
- `ONGOING`: These are fixed-price, multiple contributors can claim, all receiving the same amount
- `TIERED_PERCENTAGE`: A crowdfundable, percentage based payout for each tier (1st, 2nd, 3rd)
- `TIERED_FIXED`: Competitions with fixed price payouts for each tier (1st, 2nd, 3rd)

### Minting a Bounty

Minting a bounty begins at `OpenQ.mintBounty(bountyId, organizationId, initializationData)`.

Anyone can call this method to mint a bounty.

`OpenQV1.sol` then calls `bountyFactory.mintBounty(...)`.

The BountyFactory deploys a new `BeaconProxy`, pointing to the `beacon` address which will point each bounty to the proper implementation.

All the fun happens in the `InitOperation`. This is an ABI encoding of everything needed to initialize any of the four types of contracts.

The BountyV1.sol `initialization` method passes this `InitOperation` to `_initByType`, which then reads the type of bounty being minted, initializing the storage variables as needed.

### Funding a Bounty

All funding of bounties MUST go through the `DepositManagerV1.fundBountyToken()` or `DepositManagerV1.fundNft()` methods.

This is because the DepositManager address is where events are emitted, which we use in the [OpenQ subgraph](https://thegraph.com/hosted-service/subgraph/openqdev/openq).

All bounties use the core function `receiveFunds` inherited from `BountyCore.sol` to actually transfer the approved funds and NFTs.

All deposits are **TIMELOCKED** and become refundable after the deposit's `expiration` perioid by calling `DepositManager.refundDeposit`.

### Claiming a Bounty

All bounty claims are managed by `ClaimManagerV1.sol`.

There are two methods that allow for claims: `claimBounty`, and `permissionedClaimTieredBounty`.

#### `claimBounty`

`claimBounty` is only callable by the OpenQ oracle per the `Oraclize.onlyOracle` modifier.

The OpenQ oracle is an [Open Zeppelin Defender Autotask](https://docs.openzeppelin.com/defender/autotasks) attached to a signer using [Open Zeppelin Defender Relay](https://docs.openzeppelin.com/defender/relay).

All bounty types can be claimed this way. Depending on the use case, users like hackathon organizers might prefer a "pull" rather than "push" method of payment.

That is where `permissionedClaimTieredBounty` comes in for tiered bounties.

#### `permissionedClaimTieredBounty`

`permissionedClaimTieredBounty` allows a user who has associated their external user id (usually an OpenQ user id) to an on-chain address to claim a tier.

This on-chain/off-chain association is set in `OpenQV1.associateExternalIdToAddress` and protected by `onlyOracle`. We have another [Open Zeppelin Defender Autotask](https://docs.openzeppelin.com/defender/autotasks) that authenticates the user and calls this method with the desired associated address if authentication is successful.

With this two-way association between external user id and address, we can fetch the external user id from `msg.sender` and determine if `TieredBounty.tierWinner(userId)` is indeed the address who has been designate by the bounty issuer as the winner.

### Potential Exploits (free alpha!)

- Messing with `OpenQV1.associateExternalIdToAddress` to claim another user's tier.

#### Directly sending tokens to the bounty address

While not explicitly refused (which we can't, since ERC20 doesn't have any kind of callback mechanism for an address to know when it has received ERC20), bounty's should only be funded via DepositManager.

No guarantees can be made for payouts of funds directly sent to a bounty contract address.