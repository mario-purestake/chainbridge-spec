# Trusted Bridge

### Definitions

- `Account`: a user on a blockchain
- `Address`: unique identifier for a user on a given blockchain. Commonly the public key from a keypair.
- `Token`: the representation of an asset, which an account can "own".
- `Home Chain`: every token has a `Home Chain`, which is the chain where the token primarily exists.
- `Native Token`: the underlying token of a blockchain that secures the network.
- `Synthetic Token`: a representation of a Token on any chain that is not the `Home Chain`.
- `Event`: a piece of data stored on-chain, explaining that a certain operation occurred.
- `Validator`: an account that has the ability to call functions not available to the public, also responsible for running the bridge software.
- `Proposal`: a transfer request from one chain to another.
- `Threshold`: the number of required votes to pass a proposal.
- `Handler`: a method which resolves the transfer of a token
- `Hash`: some hashing algorithm, this does not need to be standardized across all blockchains.
- `ChainId`: a unique identifier associated with a single chain (TODO: Specify how they are chosen)
// TODO: Remove or change
- `Safe`: an account that can be in possession of a token or native token. The Safe maintains custody of assets that are transferred out of the chain.
- `Handler`: Responsible for decoding deposits, typically paired with a Safe. Note: Handlers have a relationship between blockchains. The metadata field must be standardized between every blockchain.

### Assumptions
Every blockchain must have:
- the capability of executing application-specific code written for the purpose of bridging (eg: substrate -> pallets, ethereum -> smart contracts)
- some way to represent a token that is interoperable with the ERC standards
- the ability to track a set of bridge validators (TODO: How configurable does this need to be)
- a mechanism to ensure calls to the on-chain bridge logic can be limited to those signed by validators
- the following state, events, and methods are provided by the chain

### State

TODO: Whitelist chains

Every chain should track the following state:

```go
type Counter struct {
    received: uint,
    sent: uint,
}

// For all whitelisted chains the number of transactions between that 
// chain and the current chain must be tracked. These values are used 
// to assign `depositId` to a proposal.
var ChainCounters map[ChainId]Counter

// The number of votes required for a proposal to pass (or fail).
var Threshold uint
```

### Events
```go
type DepositEvent event {
    destChain: uint,
    depositId: uint,
    to: account,
    amount: uint,
}
```

- `destChain`: the `ChainId` denoting the blockchain where the deposit should be made.
- `depositId`: the nonce for the deposit, found be querying `ChainCounters`
- `to`: the account of the recipient of a specific deposit on the destination chain
- `amount`: The amount of a specific token that will be transferred

### Interfaces & Methods

```go
func createProposal(hash [32]byte, originChain ChainId, depositId uint)
```
// TODO: What about metadata?
- `hash`: a hash of `originChain`, `depositId` (from the origin chain), and `metadata`
- `originChain`: The unique `ChainId` denoting which chain the deposit came from.
- `depositId`: The deposit Id that was generated on the `originChain`

```go
func executeProposal(originChain ChainId, depositId uint, metadata []byte)
```
//TODO: Do we need hash as a param?
- `originChain`: The unique `ChainId` denoting which chain the deposit came from.
- `depositId`: The deposit Id that was generated on the `originChain`
- `metadata`: Byte arrary that is decoded by a handler

### Messaging Formats
`tokenMessageFormat`
// TODO define how many bytes to parse. eg: to = metadata[:32], amount = metadata[32:40]
- `to: chain-specific`: The account of the recipient
- `amount: uint`: The amount of a token to be transfered
// Im not sold on this. Should the contracts know about their chain Id?
- `id: string`: Unique id in the format of `<origin_chain><unique_id>`
- TBD

# On-chain requirements
## Specification for moving tokens
1. Deposit Tokens
    0. An account ("AccountA") has ownership over some amount of tokens ("ABCToken")
    1. AccountA invokes a function to make a valid transfer of some amount of ABCToken into the Safe.
    2. An event of type `DepositEvent` is invoked.

2. Receiving Tokens
    1. A proposal is created with `createProposal`
    2. Once a proposal is created, other validators may vote on that proposal until the threshold is met. Where `yesVotes >= threshold || validatorSet.length - noVotes < threshold`
    3. If the threshold is met, and the number of `yesVotes` approved a proposal, a validator can execute the deposit. The deposit function needs at minimum the paramters denoted in `executeDeposit()`.
    4. To succesfully execute a deposit, the hash provided in `createProposal()` must match the corresponding values submitted to executeDeposit(), if the hashes do not match, the transaction cannot proceed.

3. Handlers
    1. A handler consumes the `metadata` supplied from the `executeDeposit()`.
    2. The handler decodes the `metadata` as defined in the `tokenMessageFormat`.
    3. The handler then needs to determine if the token's home chain is the current chain.
        1. If so, the corresponding Safe should have that token locked. It should then release the `amount` tokens to the `to` account defined in the `tokenMessageFormat`.
        2. If not, the Safe handler should mint `amount` tokens to the `to` account defined in the `tokenMEssageFormat`

## Identying a synthetic from a native token
To properly identify a token, the `tokenMessageFormat` contains a field `id`. The id follows the format of `<origin_chain><unique_id>`, Where, `id` is the full string, and `id[:4]` are reserved for the chain_id, and `id[4:]` are reserved for the `unique_id`. Every chain is the responsible for keeping a reference of the unique_id.
