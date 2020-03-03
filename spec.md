# Trusted Bridge

### Definitions
- `Address`: A unique way to identify a user on a given blockchain. Commonly the public key from a keypair.
- `Token`: A representation of a currency, an address can hold a balance. The most popular blockchain version being the ERC20 standard made popular on Ethereum.
- `Handler`: A function that accepts a byte array, and decodes it to a specific format.
- `Home Chain`: The blockchain where a token originally was created.
- `Safe`: A set of functions that live on a blockchain. The Safe can take ownership of tokens, and is able to also transfer them.
- `Event`: An identifier that comes from an on-chain function, explaining that a certain operation occured.
- `Handler`: Responsible for decoding deposits, typically paired with a Safe. Note: Handlers have a relationship between blockchains. The metadata field must be standardized between every blockchain. This means proposing a new handler, means a data format must be specified through the ChainBridge Improvement Process (CHIP)
- `Validator`: An address that has the ability to call functions not available to the public.
- `Proposal`: A transfer request from one chain to another.
- `Threshold`: A number that represents the reuqired number of votes to pass a proposal.
- `Hash`: The use of a hash function that is most applicable to a blockchain, this does not need to be standardized across all blockchains.

### Assumptions
- A blockchain has a method that allows a user to call a function (eg: substrate -> pallets, ethereum -> smart contracts)
- A blockchain has some way to represent a token

### Interfaces & Methods
`DepositEvent`
- `destChain: uint`: The ChainBridge unique chain Id, denoting the blockchain where the deposit should be made.
- `depositId: uint`: A counter that represents the number of deposits made on a specific blockchain
- `to: chain-specific`: The address of the recipient of a specific deposit
- `amount: uint`: The amount of a specific token that will be transferred
- TBD

`createProposal()`
- `hash: bytes[32]`: a hash of `originChain`, `depositId` (from the origin chain), and `metadada`
- `originChain: uint`: The chainBridge unique chain Id, denoting which chain the deposit came from.
- `depositId: uint`: The deposit Id that was generated on the originChain

`executeProposal()`
- `originChain: uint`: The chainBridge unique chain Id, denoting which chain the deposit came from.
- `depositId: uint`: A counter that represents the number of deposits made on a specific blockchain
- `metadata: bytes[]`: Byte arrary that is decoded by a handler

### Messaging Formats
`tokenMessageFormat`
// TODO define how many bytes to parse. eg: to = metadata[:32], amount = metadata[32:40]
- `to: chain-specific`: The address of the recipietn
- `amount: uint`: The amount of a token to be transfered
// Im not sold on this. Should the contracts know about their chain Id?
- `id: string`: Unique id in the format of `<origin_chain><id_unique_to_chain>`
- TBD

# On-chain requirements
## Specification for moving tokens
1. Deposit Tokens
    1. An address ("AddressA") has ownership over some amount of tokens ("ABCToken")
    2. AddressA invokes a function a function for some `depositAmount` of ABCToken into the Token Handler, such that `depositAmount <= ABCToken.balanceOf(AddressA)`.
    3. The handler takes the deposit, and invokes a function on the Token Safe, which takes custody of `depositAmount` worth of ABCToken.
    4. An event representing the pseudocoded `DepositEvent` is invoked.

2. Receiving Tokens
    1. Assumptions:
        1. There must be a set of validators (`validatorSet`).
        2. A function must exist that allows a validator to create a validator proposal.
        3. There must be a threshold (`threshold`).
    2. Note: Chains that wish to optimize, may combine any of the following steps into one or more functions. 
    3. Steps:
        1. A proposal is created with the params from `createProposal`
        2. Once a proposal is created, other validators can vote on that proposal until the threshold is met. Where `yesVotes >= threshold || validatorSet.length - noVotes < threshold`
        3. If the threshold is met, and the number of `yesVotes` approved a proposal, a validator can execute the deposit. The deposit function needs at minimum the paramters denoted in `executeDeposit()`.
        4. To succesfully execute a deposit, the hash provided in `createProposal()` must match the corresponding values submitted to executeDeposit(), if the hashes do not match, the transaction cannot proceed.

3. Handlers
    1. A handler consumes the `metadata` supplied from the `executeDeposit()`.
    2. The handler decodes the `metadata` as defined in the `tokenMessageFormat`.
    3. The handler then needs to determine if the token's home address is the current chain.
        1. If so, the corresponding Safe should have that token locked. It should then release the `amount` tokens to the `to` address defined in the `tokenMessageFormat`.
        2. If not, the Safe handler should mint `amount` tokens to the `to` address defined in the `tokenMEssageFormat`
