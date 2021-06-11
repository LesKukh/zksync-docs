# Basic Concepts

[[toc]]

## Operations

There are two types of operations in zkSync:

1. Priority operations
2. Transactions

### Priority operations

Priority operations are initiated directly on the Ethereum mainnet. For example, the user creates a deposit transaction
to move funds from Ethereum to zkSync. A priority operation can be identified by its numeric ID or by the hash of the ethereum
transaction that created it.

Types of priority operations:

- `Deposit`: Moves funds from the Ethereum network to the designated account in the zkSync network. If the recipient account 
  does not exist yet on the zkSync network, it will be created and a numeric ID will be assigned to the provided address.
  
- `FullExit`:Withdraws funds from the zkSync network to the Ethereum network without interacting with the zkSync server. This 
  operation can be used as an emergency exit when censorship is detected from the zkSync server node, or to withdraw funds when 
  the signing key for an zkSync  account cannot be set (e.g. if the account’s address corresponds to a smart contract).

### Transactions

Transactions are submitted via zkSync operator using the API.

Transactions are identified by the hash of their serialized representation.

Types of transactions:

- `ChangePubKey`: Sets (or changes) the signing key associated with the account. Without a set signing key, the account can authorize 
   only priority operations but not transactions.
- `Transfer`: Transfers funds from one zkSync account to another zkSync account. If the recipient account does not exists yet on the 
   zkSync network, it is created and a numeric ID is assigned to the provided address.
- `Swap`: Atomically swaps funds between two existing zkSync accounts. 
- `Withdraw`: Withdraws funds from the zkSync network to the Ethereum network.
- `ForcedExit`: Withdraws funds from the "target" L2 account that doesn't have a set signing key, to the same "target" address on the 
   Ethereum network. This operation can be used to withdraw funds when the signing key for a zkSync account cannot be set (e.g. if the 
   address corresponds to a smart contract). 

## Blocks

All operations inside zkSync are arranged in blocks. After zkSync operator creates a block, it is pushed with a Commit transaction to the zkSync smart contract on the Ethereum mainnet. 

The fact that the block is committed, does not mean that its state is final. A few minutes are still needed for the Zero Knowledge proof 
of the block’s correctness to be produced afterwards. This proof is published then on Ethereum with the Verify transaction, and it is only after the Verify transaction is mined, that the new state is considered to be final. Multiple blocks can be committed but not verified yet.

The execution model is slightly different though. For the users not to wait till the block is finalized, transactions are grouped into "mini-blocks" with a much smaller timeout. The blocks are then partially applied with a small interval, so that shortly after a transaction is received, it is executed and L2 state is updated correspondingly.

It means that after sending a transaction, the user does not have to wait for block commitment or verification. The 
transferred funds can be used immediately after the transaction is executed.

## Flow

This section describes typical zkSync use-cases.

### Creating an account

Accounts in zkSync can be created by either depositing funds from Ethereum to zkSycn or by transferring funds within zkSync 
to the desired address. Any of these options will create a new account in the zkSync network, if it has not yet existed.

The newly created account can’t authorize any transactions yet. Its owner has to first set a signing key for it.


### Setting the signing key

By default, the signing key for each account is set to the zero value, which marks the account as "unowned". It's required 
because:

- If a transfer to some address is valid in Ethereum, it is also valid in zkSync.
- Not every address can have a private key (e.g. some smart contracts can’t have one).
- Transfers to a user's account may happen before they've been interested in zkSync.

Thus, for an account to initiate L2 transactions, the user must set a signing key for it by `ChangePubKey` transaction.

This transaction needs two signatures:

- zkSync signature of the transaction data, so that it won't be possible to mutate it.
- Ethereum signature to prove the ownership of the account.

Ethereum signature should be a signature of some pre-defined message. It also is possible to authorize a `ChangePubKey` 
transaction on-chain by sending a corresponding transaction to the zkSync smart contract. zkSync server and smart contract 
will both check the ownership of the account for better security.

The zkSync signature on all transaction fields must correspond to the public key provided in the transaction.

### Transferring funds

As mentioned above, any transfer that is valid in Ethereum, is also valid in zkSync.

Users may transfer any amount of funds in either Ether or any supported ERC-20 token. A list of supported tokens can be found on the  
[explorer page](https://zkscan.io/tokens) or via [API](/api).

A transfer to a non-existent account requires slightly more data to be sent on the smart contract (we have to include information 
about the newly created account).  Fees for such transfers are therefore slightly higher than fees for transfers made to existing 
accounts.

### Fees

zkSync requires transaction fees to cover expenses for network maintenance.

Fees for each kind of transaction are calculated based on:

- Amount of data to be sent to the Ethereum network.
- Current gas price.
- Costs of computations for generating a ZK proof for the block containing this transaction.

Since many transactions are included in one block, the cost is spread among all of them, resulting in very small fees.

All input data used for fee calculation is provided via [API method][api_fee].

[api_fee]: /api/v0.1.md#get-tx-fee

### Withdrawing funds

There are three ways to withdraw funds from zkSync to an Ethereum account.

`Withdraw`

This is an L2 transaction to request a withdrawal from an owned account with a set signing key to any Ethereum address. As with transfers, it has to be signed by the correct zkSync private key. 

The transaction initiator covers the fees.

Use withdraw when you own your account and have a private key for it.


`ForcedExit`

This is an L2 transaction to request a withdrawal from an unowned account without a set signing key. Neither Ethereum address nor amount can be chosen in this transaction. Its only option is to request a withdrawal of all available funds of a certain token from the target L2 address to the target L1 address.

The transaction initiator covers  the fees.

Use ForcedExit to withdraw funds from an account for which one can’t set the signing key (i.e. smart contract account), when there exists an L2 account which can send this type of transaction.


`FullExit`
This is a priority operation initiated from L1. The smart contract [guarantees](/faq/security.md#security-overview) that either this request will be processed within a reasonable time, or the network will be considered compromised/dead, and the contract will enter the exodus mode.

Use FullExit if your ever experience censorship from the zkSync server node.
