***************
Latus Consensus
***************

As we have just seen, the Cross-Chain Transfer Protocol does not impose any requirements on the Sidechain architectural design other than the need to support the protocol itself. Having said that, the Horizen Sidechain SDK does offer a ready made implementation of the Latus consensus, which is a Proof of Stake (“PoS”)  consensus based on the Ouroboros Praos protocol.

Consensus Epochs & Forging
===========================

In Latus, the chain is split into “consensus epochs”; each epoch is made of a predefined number of time slots. Each slot is assigned to slot leaders, which are then authorized to generate (“forge”) a block during that slot. So the protocol operates in a synchronous environment where each slot spans over a specific amount of time (e.g. 20 seconds).
Slot leaders of a particular consensus epoch are chosen randomly before the epoch begins from the set of all sidechain forging stakeholders. The forging stake is a subset of all the coins managed by a sidechain. In fact each sidechain participant who wants to be a Forger, must have some forging stake - i.e. a set of “ForgerBoxes” assigned to him. ForgerBox is a particular kind of Box that contains an amount of coins locked for forging, and some specific data used by the forger to prove its block producing eligibility associated with that stake amount. The total amount of coins staked in ForgerBoxes is the total Forging Stake amount.
The possibility of being a slot leader increases with the percentage of forging stake owned. It's possible to have more than one slot leader per slot; if more than one block is propagated, only one will be accepted by each node; the consensus rules will make sure that conflicting chains will eventually converge to a winning chain. Conversely, a consensus epoch could have empty slots, if their slot leader (or leaders) have not created and propagate blocks for them.

A slot leader eligible for a certain slot, that decides to create and propagate a new Sidechain Block for that slot, is called a “forger”. A forger proves its eligibility for a slot by including in the block a cryptographic proof, in such a way that any node can validate, besides the validity of each transaction, also that the "slot leader" selection rule for that specific slot and consensus epoch was respected.

Forgers are also entitled and incentivized to include sidechain transactions and mainchain synchronization data into Sidechain Blocks.
A limited amount of mainchain block data is added to sidechain blocks, in such a way that all the mainchain transactions that refer to a particular sidechain are included in that sidechain, that a reference to each mainchain block is present in all sidechains, and that enough information is published in a sidechain such that any sidechain node is able to validate the mainchain block references without the need for a direct connection to the mainchain itself. Please note, the forger will need its own direct connection to mainchain nodes, to have a source of mainchain blocks data.
The connection between Mainchain and Sidechain nodes is established via a websocket interface provided by the mainchain node. 

The Latus consensus, including MainchainBlock synchronization and all forging logic and functionality, is implemented out-of-the-box by the SDK Core, and developers do not need to make any change to this. The forging process can be fully managed through the API interface provided by the SDK (see the next topic).

Default Latus consensus parameters
==================================

  * Seconds in one slot - 120, i.e. one block could be generated in two minutes
  * Number of slots in one consensus Epoch - 720, i.e. new nonce is generated (and thus forging stake holder could check slot leader possability) every 720 * 120 =  86400 seconds, i.e. 24 hours.
