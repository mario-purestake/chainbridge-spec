# Trusted Bridge

### Definitions

- `Account`: a user on a blockchain
- `Address`: unique identifier for a user on a given blockchain. Commonly the public key from a keypair.
- `Token`: the representation of an asset, which an account can "own".
- `Home Chain`: every token has a `Home Chain`, which is the chain where the token primarily exists.
- `Native Token`: the underlying token of a blockchain that secures the network.
- `Synthetic Token`: a representation of a Token on any chain that is not the `Home Chain`.
- `Event`: a piece of data stored on-chain, explaining that a certain operation occurred.
- `Relayer`: an account that has the ability to call functions not available to the public, also responsible for running the bridge software.
- `Proposal`: a transfer request from one chain to another.
- `Threshold`: the number of required votes to pass a proposal.
- `Handler`: a method which resolves the transfer of data
- `Hash`: some hashing algorithm, this does not need to be standardized across all blockchains.
- `ChainId`: a unique identifier associated with a single chain (TODO: Specify how they are chosen)
// TODO: Remove or change
- `Safe`: an account that can be in possession of a token or native token. The Safe maintains custody of assets that are transferred out of the chain.
- `Handler`: Responsible for decoding deposits, typically paired with a Safe. Note: Handlers have a relationship between blockchains. The metadata field must be standardized between every blockchain.

### Assumptions
Every blockchain must have:
- the capability of executing application-specific code written for the purpose of bridging (eg: substrate -> pallets, ethereum -> smart contracts)
- some way to represent a token that is interoperable with the ERC standards
- the ability to track a set of bridge relayers (TODO: How configurable does this need to be)
- a mechanism to ensure calls to the on-chain bridge logic can be limited to those signed by relayers
- the following state, events, and methods are provided by the chain

### Types
In the specification there are references to different types, below are the intentions behind them.
|Type |Definition |
|---|---|
|uint |32 bit unsigned integers |
|[]byte | An arbitrary sized array of bytes |
|[x]byte | An fixed sized array of `x` bytes |

## On-Chain State
All chains will need to have a few hardcoded constants, and keep track of some state.

### Constants
- `CHAIN_ID`: All chains are assigned a `chain_id`, and therefore all chains will need to know about their own `chain_id`.

### State
```go
type Counter struct {
    received: uint,
    sent: uint,
}

// For all whitelisted chains the number of transactions between that 
// chain and the current chain must be tracked. These values are used 
// to assign `depositNonce` to a proposal.
var ChainCounters map[ChainId]Counter

// The number of votes required for a proposal to pass (or fail).
var Threshold uint

// TODO Add whitelisted chains
```

## Message Formats
To effectively move data around, there are a few message types, these types have their own respective format for a how a message should be resolved on-chain. The types are:
- Tokens
- Arbitrary Messages


### Token Message Format
- `id: string`: Unique id in the format of `<origin_chain_id><unique_id>`, where the `origin_chain_id` is 4 bytes in length, and the `unique_id` is 32 bytes in length.
- `to: address`: The account representing the recipient
- `amount: uint`: The amount of a token to be transfered
- `destChainId: chainId`: The destination chainId
- `depositNonce: uint`: The nonce for the deposit, found be querying `ChainCounters`
- `metadata: []byte`: Can be any structured data (if needed)
- `handlerAddress: address || null`: The account of the on-chain handler (if needed, based on destination chain)

#### Disecting `token.id`
`id` is a way for chains to understand where a token originated from. It is determined on the home chain of a given token. This is necessary to distinguish proposals, and to ensure chains can't have identifier conflicts.

`token.id` is comprised of two pieces: `chain_id` and `unique_id`. `chain_id` is the home `chain_id`, the `unique_id` is a some identifier that allows the home chain to easily map the token to its native account. For evm based chains, this would most likely be the address, that way the home chain can easily interpret where to release funds from (rather than mint). 

### Events
```go
type DepositEvent event {
    destChain: uint,
    depositNonce: uint,
    to: account,
    amount: uint,
}
```

- `destChain`: the `ChainId` denoting the blockchain where the deposit should be made.
- `depositNonce`: the nonce for the deposit, found be querying `ChainCounters`
- `to`: the account of the recipient of a specific deposit on the destination chain
- `amount`: The amount of a specific token that will be transferred

### Interfaces & Methods
```go
func deposit()
```

```go
func createProposal(hash [32]byte, originChain ChainId, depositNonce uint)
```
// TODO: What about metadata?
- `hash`: a hash of `originChain`, `depositNonce` (from the origin chain), and `metadata`
- `originChain`: The unique `ChainId` denoting which chain the deposit came from.
- `depositNonce`: The deposit Id that was generated on the `originChain`

```go
func executeProposal(originChain ChainId, depositNonce uint, []T params)
```
- `originChain`: The unique `ChainId` denoting which chain the deposit came from.
- `depositNonce`: The deposit Id that was generated on the `originChain`.
- `params`: Any specific parameters pertaining to a given chain.


## Lifecycles
### Proposal Voting
1. A proposal is created with `createProposal`
2. Once a proposal is created, other relayers may vote on that proposal until the threshold is met. Where `yesVotes >= threshold || noVotes >= threshold`
3. If the threshold is met, and the number of `yesVotes` approved a proposal, the deposit can be executed. The deposit function needs at minimum the paramters denoted in `executeProposal()`.
4. To succesfully execute a deposit, the hash provided in `createProposal()` must match the corresponding values submitted to executeDeposit(), if the hashes do not match, the transaction cannot proceed.

### Handler Lifecycle
- A handler is responsible for both the deposit and receiving of a transfer.
- Access to a handler should be restricted by the approval of a deposit, thus a handler should be called upon when the criteria for `executeProposal()` is met.
0. A proposal has been accepted.
1. A handler consumes the any required params from the `executeProposal()`.
2. The respective handler consume the parameters, and performs necessary logic to complete its oppperations.

## Tokens
### Depositing Tokens
0. An address ("AddressA") has ownership over some amount of tokens ("ABCToken")
1. AddressA invokes a function to transfer the tokens. This call should emit an event which can be read by the bridge software.
3. AddressA invokes a function to make a valid transfer of some amount of ABCToken into the Safe.
4. An event of type `DepositEvent` is invoked.

### Receiving Tokens
0. A proposal has been accepted per the terms listed in [Proposal Voting](#Proposal-Voting)
1. The handler then needs to determine if the token's home address is the current chain.
    1. If so, the corresponding Safe should have that token locked. It should then release the `amount` tokens to the `to` address defined in the `tokenMessageFormat`.
    2. If not, the Safe handler should mint `amount` tokens to the `to` address defined in the `tokenMEssageFormat`

# On-chain requirements
## Specification for moving tokens
1. Deposit Tokens
    0. An account ("AccountA") has ownership over some amount of tokens ("ABCToken")
    1. AccountA invokes a function to make a valid transfer of some amount of ABCToken into the Safe.
    2. An event of type `DepositEvent` is invoked.

2. Receiving Tokens
    1. A proposal with event type of `DepositProposalCreated` is invoked.
    2. Relayers then should vote on that proposal.
    3. If the threshold is met, then a handler should be invoked to perform the transfer.

3. Handlers
    1. A handler consumes the `metadata` supplied from the `executeDeposit()`.
    2. The handler decodes the `metadata` as defined in the `tokenMessageFormat`.
    3. The handler then needs to determine if the token's home chain is the current chain.
        1. If so, the corresponding Safe should have that token locked. It should then release the `amount` tokens to the `to` account defined in the `tokenMessageFormat`.
        2. If not, the Safe handler should mint `amount` tokens to the `to` account defined in the `tokenMEssageFormat`

----
# WIP Notes

Main functions
- A chain has relayers
- Only relayers can create proposals
- A chain keeps track of the amount of deposits made
- Relayers can vote on proposals
- Handlers must support tokens that resemble the ERC20 standard, and the ERC721
- The ERC based handlers should allow for whitelisting of tokens
- A token that leaves its home-chain becomes a synthetic, that synthetic cannot go anywhere else but back to its home-chain

## Identying a synthetic from a native token
To properly identify a token, the `tokenMessageFormat` contains a field `id`. The id follows the format of `<origin_chain><unique_id>`, Where, `id` is the full string, and `id[:4]` are reserved for the chain_id, and `id[4:]` are reserved for the `unique_id`. Every chain is the responsible for keeping a reference of the unique_id.

## Ethereum specific functions
```go
func executeProposal(originChain ChainId, depositNonce uint, handlerAddress address, metadata []byte)
```
- `originChain`: The unique `ChainId` denoting which chain the deposit came from.
- `handlerAddress`: The corresponding address of a handler account.
- `depositNonce`: The deposit Id that was generated on the `originChain`
- `metadata`: Byte arrary that is decoded by a handler

2. Receiving Tokens
    1. A proposal is created with `createProposal`
    2. Once a proposal is created, other relayers may vote on that proposal until the threshold is met. Where `yesVotes >= threshold || relayerSet.length - noVotes < threshold`
    3. If the threshold is met, and the number of `yesVotes` approved a proposal, a relayer can execute the deposit. The deposit function needs at minimum the paramters denoted in `executeDeposit()`.
    4. To succesfully execute a deposit, the hash provided in `createProposal()` must match the corresponding values submitted to executeDeposit(), if the hashes do not match, the transaction cannot proceed.

### Handler Lifecycle
- A handler is responsible for both the deposit and receiving of a transfer.
- Access to a handler should be restricted by the approval of a deposit, thus a handler should be called upon when the criteria for `executeProposal()` is met.
0. A proposal has been accepted.
1. A handler consumes the any required params from the `executeProposal()`.
// Eth specific
2. The respective handler decodes the `metadata` as defined in the `Message Fromats` section.
3. Upon decoding of the message, the Handler is free to perform its opperations.
