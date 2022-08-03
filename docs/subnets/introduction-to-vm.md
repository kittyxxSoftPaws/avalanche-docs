# Introduction to Virtual Machines

## Overview

This is part of a series of tutorials for building a Virtual Machine (VM):

- Introduction to Virtual Machines (this article)
- [How to Build a Simple VM](./create-a-virtual-machine-vm.md)
- [How to Build a Complex VM](./create-a-vm-blobvm.md)

A Virtual Machine is a blueprint for a blockchain and a blockchain instantiates from a VM, very much similar to a class-object relationship. A Virtual Machine can define following: issuance of transactions, transaction types, block structure, block building algorithm, keeping pending transactions in the mempool, gossiping transactions to the connected nodes, etc.

## Blocks and State

Virtual Machines can be broken down into 2 components: blocks and state. The functionality provided by VMs is to:

- Define the representation of a blockchain's state
- Represent the operations in that state
- Apply the operations in that state

Each block in the blockchain contains a set of state transitions. Each block is applied in order from the blockchain's initial genesis block to its last accepted block to reach the latest state of the blockchain.
## Blockchain

A blockchain has 2 components: **Consensus Engine** and **Virtual Machine**. VMs mainly deal with the implementation related to the block's structure: building and parsing. The Consensus Engine helps in reaching consensus on the block issued by VM. Here is a brief overview:

1. A validator node wants to update the blockchain's state
2. It creates a block with state transition details in it
3. It issues the block to the consensus engine
4. Consensus engine gossips the block within the network to reach consensus on it
5. Depending upon the consensus results, the engine can either accept or reject the block
6. Every virtuous node on the network should have the same verdict for a particular block
7. It depends on the VM implementation on what to do with the accepted or rejected block

The consensus engine has been already provided by `AvalancheGo` on the Avalanche network. Developers can implement and customize their own VM.

## How to Load a VM

VMs are created as a module, whose binary is registered by a node running `AvalancheGo`, against the **vmID** (binary file name must be vmID). VMID is a user-defined string that is zero-extended to a 32-byte array and encoded in CB58.

The Avalanche nodes that want to power their chains with a particular VM, must have the built binary put in the proper place. See [here](../nodes/maintain/avalanchego-config-flags.md#--build-dir-string) for more details. There could be multiple VM plugins in this directory.

We can build multiple blockchains with the registered VM. A VM is registered, only when its built binary is present at the required location.

A blockchain can run as a separate process from AvalancheGo and can communicate with `AvalancheGo` over gRPC. This is enabled by `rpcchainvm`, a special VM that uses [`go-plugin`](https://pkg.go.dev/github.com/hashicorp/go-plugin) and wraps another VM implementation. The C-Chain, for example, runs the [Coreth](https://github.com/ava-labs/coreth) VM in this fashion.

### APIs for a VM

We interact with a blockchain and underlying VM through **Static** and **Non-Static Handlers**. Handlers are used for making API calls to VM methods. Specifically, these handlers are implemented as **Services**.

### Handlers

Handlers serve the response for the incoming HTTP requests. Handlers can also be wrapped with **gRPC** for efficiently making calls from other services such as `AvalancheGo`. Handlers help in creating APIs. VM implements 2 kinds of handlers:

- **Non-Static Handlers** - They help in interacting with blockchains instantiated by the VM. The API's endpoint will be different for different chains. `/ext/bc/[chainID]`
- **Static Handlers** - They help in directly accessing VM. These are optional for reasons such as parsing genesis bytes required to instantiate new blockchains. `/ext/vm/[vmID]`

For any developers familiar with object-oriented programming, this is very similar to static and non-static methods on a class.

### Registering a VM

While [registering a VM](https://github.com/ava-labs/avalanchego/blob/master/vms/manager.go#L93), we store its **factory** against the **vmID**. As the name suggests, a VM factory can create new VM instances.

Using the VM's instance, we can initialize it and can create a new blockchain. Note that, instantiating a VM is different than initializing it. We initialize the instance of a VM with parameters like genesis, database manager, etc. to create a functional blockchain. A functional chain doesn't mean the node will start issuing blocks and processing transactions. This will only happen once the node has been bootstrapped.

```go
// Registering a factory inside avalanchego
func (m *manager) RegisterFactory(vmID ids.ID, factory Factory) error {
    ...
    m.factories[vmID] = factory
    ...
}
```

### Instantiating a VM

Each VM has a **factory** that is capable of creating new VM instances. A VM can be initialized as a blockchain along with handlers for accessing it, only when we have the VM's instance. The factory's `New` method shown below, returns the VM's instance to its caller in `AvalancheGo`. It's generally in the [`factory.go`](https://github.com/ava-labs/blobvm/blob/master/factory.go) file of the VM.

```go
// Returning a new VM instance from VM's factory
func (f *Factory) New(*snow.Context) (interface{}, error) { return &vm.VM{}, nil }
```

A VM's functionality is exposed to `AvalancheGo` by registering its factory and [instantiating](https://github.com/ava-labs/avalanchego/blob/master/chains/manager.go#L399) a new VM using this factory whenever a chain is required to be built.

```go
// Instantiating a new VM from its factory in avalanchego
func (m *manager) buildChain(chainParams ChainParameters, sb Subnet) (*chain, error) {
    ...
    vmFactory, err := m.VMManager.GetFactory(vmID)
    vm, err := vmFactory.New(ctx.Context)
    ...
}
```

### Initializing a VM

Blockchains are functional, only when the instantiated VMs are initialized and the node is bootstrapped. Initializing a VM involves setting up the database, block builder, mempool, genesis state, handlers, etc. This will expose the VM's API handlers, start accepting transactions, gossiping them across the network, building blocks, etc. More details on it can be found [later](#Initialize) in the documentation.

```go
if err := vm.Initialize(
    ctx.Context,
    vmDBManager,
    genesisData,
    chainConfig.Upgrade,
    chainConfig.Config,
    msgChan,
    fxs,
    sender,
);
```

You can refer to the [implementation](https://github.com/ava-labs/blobvm/blob/master/vm/vm.go#L92) of `vm.initialize` in the BlobVM repository.

### Interaction

We can use the API handlers to issue transactions, query chain state, and other functionalities provided by the VM.

## Interfaces

The main task of a VM is to issue blocks to the consensus engine whenever requested. It is the VM's responsibility to handle user transactions, and include them into new blocks, mempool handling, and database handling.

The consensus engine will just request the VM for new blocks whenever there is a signal (from VM) and according to other [congestion control mechanisms](https://github.com/ava-labs/avalanchego/blob/master/vms/proposervm/README.md), verify the block, gossip the block within the network for consensus, and request the VM to accept or reject the block.

Every VM should implement the following interfaces:

### `block.ChainVM`

To reach a consensus on linear blockchains (as opposed to DAG blockchains), Avalanche uses the Snowman consensus engine. To be compatible with Snowman, a VM must implement the `block.ChainVM` interface, which can be accessed from [AvalancheGo repository](https://github.com/ava-labs/avalanchego/blob/v1.7.4/snow/engine/snowman/block/vm.go).

```go title="/snow/engine/snowman/block/vm.go"
// ChainVM defines the required functionality of a Snowman VM.
//
// A Snowman VM is responsible for defining the representation of the state,
// the representation of operations in that state, the application of operations
// on that state, and the creation of the operations. Consensus will decide on
// if the operation is executed and the order operations are executed.
//
// For example, suppose we have a VM that tracks an increasing number that
// is agreed upon by the network.
// The state is a single number.
// The operation is setting the number to a new, larger value.
// Applying the operation will save to the database the new value.
// The VM can attempt to issue a new number, of larger value, at any time.
// Consensus will ensure the network agrees on the number at every block height.
type ChainVM interface {
	common.VM
	Getter
	Parser

	// Attempt to create a new block from data contained in the VM.
	//
	// If the VM doesn't want to issue a new block, an error should be
	// returned.
	BuildBlock() (snowman.Block, error)

	// Notify the VM of the currently preferred block.
	//
	// This should always be a block that has no children known to consensus.
	SetPreference(ids.ID) error

	// LastAccepted returns the ID of the last accepted block.
	//
	// If no blocks have been accepted by consensus yet, it is assumed there is
	// a definitionally accepted block, the Genesis block, that will be
	// returned.
	LastAccepted() (ids.ID, error)
}

// Getter defines the functionality for fetching a block by its ID.
type Getter interface {
	// Attempt to load a block.
	//
	// If the block does not exist, an error should be returned.
	//
	GetBlock(ids.ID) (snowman.Block, error)
}

// Parser defines the functionality for fetching a block by its bytes.
type Parser interface {
	// Attempt to create a block from a stream of bytes.
	//
	// The block should be represented by the full byte array, without extra
	// bytes.
	ParseBlock([]byte) (snowman.Block, error)
}
```

### `common.VM`

`common.VM` is a type that every `VM`, whether a DAG or linear chain, must implement.

You can see the full file from [here.](https://github.com/ava-labs/avalanchego/blob/v1.7.4/snow/engine/common/vm.go)

```go title="/snow/engine/common/vm.go"
// VM describes the interface that all consensus VMs must implement
type VM interface {
    // Contains handlers for VM-to-VM specific messages
	AppHandler

	// Returns nil if the VM is healthy.
	// Periodically called and reported via the node's Health API.
	health.Checkable

	// Connector represents a handler that is called on connection connect/disconnect
	validators.Connector

	// Initialize this VM.
	// [ctx]: Metadata about this VM.
	//     [ctx.networkID]: The ID of the network this VM's chain is running on.
	//     [ctx.chainID]: The unique ID of the chain this VM is running on.
	//     [ctx.Log]: Used to log messages
	//     [ctx.NodeID]: The unique staker ID of this node.
	//     [ctx.Lock]: A Read/Write lock shared by this VM and the consensus
	//                 engine that manages this VM. The write lock is held
	//                 whenever code in the consensus engine calls the VM.
	// [dbManager]: The manager of the database this VM will persist data to.
	// [genesisBytes]: The byte-encoding of the genesis information of this
	//                 VM. The VM uses it to initialize its state. For
	//                 example, if this VM were an account-based payments
	//                 system, `genesisBytes` would probably contain a genesis
	//                 transaction that gives coins to some accounts, and this
	//                 transaction would be in the genesis block.
	// [toEngine]: The channel used to send messages to the consensus engine.
	// [fxs]: Feature extensions that attach to this VM.
	Initialize(
		ctx *snow.Context,
		dbManager manager.Manager,
		genesisBytes []byte,
		upgradeBytes []byte,
		configBytes []byte,
		toEngine chan<- Message,
		fxs []*Fx,
		appSender AppSender,
	) error

	// Bootstrapping is called when the node is starting to bootstrap this chain.
	Bootstrapping() error

	// Bootstrapped is called when the node is done bootstrapping this chain.
	Bootstrapped() error

	// Shutdown is called when the node is shutting down.
	Shutdown() error

	// Version returns the version of the VM this node is running.
	Version() (string, error)

	// Creates the HTTP handlers for custom VM network calls.
	//
	// This exposes handlers that the outside world can use to communicate with
	// a static reference to the VM. Each handler has the path:
	// [Address of node]/ext/VM/[VM ID]/[extension]
	//
	// Returns a mapping from [extension]s to HTTP handlers.
	//
	// Each extension can specify how locking is managed for convenience.
	//
	// For example, it might make sense to have an extension for creating
	// genesis bytes this VM can interpret.
	CreateStaticHandlers() (map[string]*HTTPHandler, error)

	// Creates the HTTP handlers for custom chain network calls.
	//
	// This exposes handlers that the outside world can use to communicate with
	// the chain. Each handler has the path:
	// [Address of node]/ext/bc/[chain ID]/[extension]
	//
	// Returns a mapping from [extension]s to HTTP handlers.
	//
	// Each extension can specify how locking is managed for convenience.
	//
	// For example, if this VM implements an account-based payments system,
	// it have an extension called `accounts`, where clients could get
	// information about their accounts.
	CreateHandlers() (map[string]*HTTPHandler, error)
}
```

### `snowman.Block`

You may have noticed the `snowman.Block` type referenced in the `block.ChainVM` interface. It describes the methods that a block must implement to be a block in a linear (Snowman) chain.

Let’s look at this interface and its methods. You can see the full file from [here.](https://github.com/ava-labs/avalanchego/blob/master/snow/consensus/snowman/block.go)

```go title="/snow/consensus/snowman/block.go"
// Block is a possible decision that dictates the next canonical block.
//
// Blocks are guaranteed to be Verified, Accepted, and Rejected in topological
// order. Specifically, if Verify is called, then the parent has already been
// verified. If Accept is called, then the parent has already been accepted. If
// Reject is called, the parent has already been accepted or rejected.
//
// If the status of the block is Unknown, ID is assumed to be able to be called.
// If the status of the block is Accepted or Rejected; Parent, Verify, Accept,
// and Reject will never be called.
type Block interface {
    choices.Decidable

    // Parent returns the ID of this block's parent.
    Parent() ids.ID

    // Verify that the state transition this block would make if accepted is
    // valid. If the state transition is invalid, a non-nil error should be
    // returned.
    //
    // It is guaranteed that the Parent has been successfully verified.
    Verify() error

    // Bytes returns the binary representation of this block.
    //
    // This is used for sending blocks to peers. The bytes should be able to be
    // parsed into the same block on another node.
    Bytes() []byte

    // Height returns the height of this block in the chain.
    Height() uint64
}
```

### `choices.Decidable`

This interface is the superset of every decidable object, such as transactions, blocks, and vertices. You can see the full file from [here.](https://github.com/ava-labs/avalanchego/blob/v1.7.4/snow/choices/decidable.go)

```go title="/snow/choices/decidable.go"
// Decidable represents element that can be decided.
//
// Decidable objects are typically thought of as either transactions, blocks, or
// vertices.
type Decidable interface {
	// ID returns a unique ID for this element.
	//
	// Typically, this is implemented by using a cryptographic hash of a
	// binary representation of this element. An element should return the same
	// IDs upon repeated calls.
	ID() ids.ID

	// Accept this element.
	//
	// This element will be accepted by every correct node in the network.
	Accept() error

	// Reject this element.
	//
	// This element will not be accepted by any correct node in the network.
	Reject() error

	// Status returns this element's current status.
	//
	// If Accept has been called on an element with this ID, Accepted should be
	// returned. Similarly, if Reject has been called on an element with this
	// ID, Rejected should be returned. If the contents of this element are
	// unknown, then Unknown should be returned. Otherwise, Processing should be
	// returned.
	Status() Status
}
```

## rpcchainvm

`rpcchainvm` is a special VM that wraps a `block.ChainVM` and allows the wrapped blockchain to run in its own process separate from AvalancheGo. `rpcchainvm` has two important parts: a server and a client. The [server](https://github.com/ava-labs/avalanchego/blob/master/vms/rpcchainvm/vm_server.go) runs the underlying `block.ChainVM` in its own process and allows the underlying VM's methods to be called via gRPC. The [client](https://github.com/ava-labs/avalanchego/blob/master/vms/rpcchainvm/vm_client.go) runs as part of `AvalancheGo` and makes gRPC calls to the corresponding server in order to update or query the state of the blockchain.

To make things more concrete: suppose that `AvalancheGo` wants to retrieve a block from a chain run in this fashion. `AvalancheGo` calls the client's `GetBlock` method, which makes a gRPC call to the server, which is running in a separate process. The server calls the underlying VM's `GetBlock` method and serves the response to the client, which in turn gives the response to `AvalancheGo`.