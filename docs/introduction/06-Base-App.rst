========
Base App
========

SidechainsSDK provides to the developers an out of the box implementation of the Latus Consensus Protocol and the Crosschain Transfer Protocol.
Additionally to this, the SDK provides basic transactions, network layer, data storage and node configuration, as well as entry points for any custom extension.


Secret / Proof / Proposition
****************************

* **Secret / Proof / Proposition** - SDK uses its own terms for secret key / public key / signed message and provides various types of them.
* **Secret** -  Private key 
* **Proof** -  Signed message
* SDK provides implementations for Secret / Proof / Proposition:
  * Curve 25519 (PrivateKey25519/PublicKey25519Proposition/Signature25519)
  * VRF based on  ginger-lib (VrfSecretKey/VrfPublicKey/VrfProof)
  * Schnorr based on ginger-lib (SchnorrSecret/SchnorrPropostion/SchnorrProof)


Boxes
*****

Data in a sidechain is meant to be represented as a Box, that we can see as data kept “closed” by a Proposition, that can be open only with the Proposition’s Secret(s).
The Sidechain SDK offers two different Box types: Coin Box and non-Coin Box. A Non-Coin box represents a unique entity which can be transferred between different owners,
for example it can be sold. A Coin box is a box which contains ZEN, examples of a Coin box are RegularBox and ForgingBox. A Coin Box can be used to add custom data to an object
that represents some coins, i.e. that holds an intrinsic defined value. As an example, a developer would extend a Coin Box to manage a time lock on a UTXO, e.g. to implement 
some kind of smart contract.
In particular, any box could be logically split in two parts: Box and BoxData (box data is included in the box). The Box itself represents the entity in the blockchain, 
i.e. all operations like create/open etc. are performed on boxes. Box data contains information about the entity like value, proposition address and any custom data.
Every box has its own unique boxId (do not be confused with box type id which is used for serialization). That box id is calculated for each box by next function:

::

	public final byte[] id() {
	   if(id == null) {
	       id = Blake2b256.hash(Bytes.concat(
		       this instanceof CoinsBox ? coinsBoxFlag : nonCoinsBoxFlag,
		       Longs.toByteArray(value()),
		       proposition().bytes(),
		       Longs.toByteArray(nonce()),
		       boxData.customFieldsHash()));
	   }
	   return id;
	}

.. note::
	The id is used during transaction verification, so it is important to add custom data  into customFieldsHash()  function.

The following Coin-Box types are provided by SDK:
  * **RegularBox** -- contains ZEN coins
  * **ForgerBox** -- contains ZEN coins are used for forging 
  * **WithdrawalRequestBox** -- contain ZEN coins are used to backward transfer, i.e. move coins back to the mainchain  
An SDK developer can declare his own Boxes, please refer to SDK extension section.

Transactions
************

There are two basic transactions: `MC2SCAggregatedTransaction
<https://github.com/HorizenOfficial/Sidechains-SDK/blob/master/sdk/src/main/java/com/horizen/transaction/MC2SCAggregatedTransaction.java>`_ and `SidechainCoreTransaction
<https://github.com/HorizenOfficial/Sidechains-SDK/blob/master/sdk/src/main/java/com/horizen/transaction/SidechainCoreTransaction.java>`_.
An MC2SCAggregatedTransaction is the implementation of Forward Transfer and can be only added as a part of the MainchainBlock reference data during synchronization with Mainchain.
When a Forger is going to produce a sidechain block and a new mainchain block appears, the forger will recreate that mainchain block as a reference that will contain sidechain 
related data. So, if some Forward Transfer exists in the mainchain block, it will be included into the MC2SCAggregatedTransaction and added as a part of the reference.
The SidechainCoreTransaction is the transaction, which can be created by anyone to send coins inside a sidechain, create forging stakes or perform withdrawal requests
(send coins back to the MC). 
The SidechainCoreTransaction can be extended to support custom logic operations. For example, if we think about real-estate sidechain, we can tokenize some private
property as a specific Box using SidechainCoreTransaction. Please refer to SDK extensions for more details.

Serialization
*************

Because the SDK is based on Scorex we implement the Scorex way of data serialization. 
  * Any serialized data like Box/BoxData/Secret/Proof/Transaction implements Scorex BytesSerializable interface/trait.
  * BytesSerializable declare functions byte[] bytes() and Serializer serializer(). 
  * Serializer  itself works with Reader/Writer which are wrappers on byte stream. 
  * Scorex Reader and Writer also implements some functionality like reading/parsing data of integer/long/string etc. 
  * Serialization and parsing itself implemented in data class by implementation byte[] bytes() (required by BytesSerializable interface) and implementation static function for parsing bytes public static Data parseBytes(byte[] bytes)
  * Also, for correct parse purposes, special bytes such as a unique id of data type is put at the beginning of the byte stream (it is done automatically), thus any serialized data shall provide a unique id. Specific serializers shall be set for those unique ids during dependency injection setting as well as custom Serializer shall be put into Custom Serializers Map which are defined at AppModule. Please refer to SDK extension section for more information

SidechainNodeView
*****************

SidechainNodeView is a provider to current Node state including NodeWallet, NodeHistory, NodeState, NodememoryPool and application data as well. SidechainNodeView is accessible during custom API implementation.  

Memory Pool
***********

A mempool is a node's mechanism for storing information on unconfirmed transactions. It acts as a sort of waiting room for transactions that have not yet been included in a block

Node wallet
***********

Contains available private keys, required for generating correct proofs

State
*****

Contains information about current node state

History
*******

Provide access to history, i.e. blocks not only from active chain but from forks as well.
 
Network layer
*************

The network layer can be divided into communication between Nodes and communication between the node and user.
Node interconnection is organized as a peer-to-peer network. Over the network, the SDK handles the handshake, blockchain synchronization, and transaction transmission.

Physical storage
****************

Physical storage. The SDK introduces the unified physical storage interface, this default implementation is based on the LevelDB library. Sidechain developers can decide to use the default solution or to provide the custom one. For example, the developer could decide to use encrypted storage, a Key Value store, a relational database or even a cloud solution.

User specific settings
**********************

The user has the ability to define custom configuration options for example, a specific path to the node data storage, wallet seed, node name and api server 
address/port, etc. To do this he should fill the configuration file in a `HOCON notation
<https://github.com/lightbend/config/blob/master/HOCON.md/>`_. The configuration file consists of the SDK required fields and application custom fields 
if needed. Sidechain developers can use com.horizen.settings.SettingsReader utility class to extract Sidechain specific data and Config object itself to get custom parts.

::

	class SettingsReader {
	    public SettingsReader (String userConfigPath, Optional<String> applicationConfigPath)

	    public SidechainSettings getSidechainSettings()

	    public Config getConfig()
	}

Moreover, if a specific sidechain contains general application settings that should be controlled only by the developer, it is possible to define basic application 
config that can be passed as an argument to SettingsReader.


SidechainApp class
******************

The starting point of the SDK for each sidechain is the SidechainApp class. Every sidechain application should create an instance of SidechainApp with passing all required parameters to it and then execute the sidechain node flow:

::

	class SidechainApp {
		public SidechainApp(
			// Settings:
			SidechainSettings sidechainSettings,

			// Custom objects serializers:
			HashMap<> customBoxSerializers,
			HashMap<> customBoxDataSerializers,
			HashMap<> customSecretSerializers,
			HashMap<> customTransactionSerializers,

			// Application Node logic extensions:
			ApplicationWallet applicationWallet,
			ApplicationState applicationState,

			// Physical storages:
			Storage secretStorage,
			Storage walletBoxStorage,
			Storage walletTransactionStorage,
			Storage stateStorage,
			Storage historyStorage,
			Storage walletForgingBoxesInfoStorage,
			Storage consensusStorage,

			// Custom API calls and Core API endpoints to disable:
			List<ApplicationApiGroup> customApiGroups,
			List<Pair<String, String>> rejectedApiPaths
		)

		public void run()
	}


The SidechainApp instance can be instantiated directly or through Guice DI library.

We can split SidechainApp arguments into 4 groups:
	1. Settings
		* The instance of SidechainSettings is retrieved by custom application via SettingsReader as was described above.
	2. Custom objects serializers
		* Developers will want to add their custom business logic. For example, tokenization of real-estate properties will 
		be required to create custom Box and BoxData types. These custom objects must be somehow managed by SDK to be sent through 
		the network or stored to the disk. In both cases SDK should know how to serialize a custom object to bytes and how to restore 
		it back. To maintain this, sidechain developers should specify custom objects serializers and add them to custom...Serializers map
		following the specific rules (see chapter 9 Code your own dApp...)
	3. Application node extension of State and Wallet logic
		* As was said above, State is a snapshot of all closed boxes of the blockchain at some moment of time. So when the next block arrives it should be validated by the State to prevent spending of non existing boxes or transaction inputs and outputs coin balances inconsistency. State can be extended by developers by introducing some logic in ApplicationState and ApplicationWallet. Seep appropriate chapters.
	4. API extension was described in chapter 
	5. Node communication
	
	
Inside the SDK we implemented a SimpleApp example, that was designed to demonstrate the basic SDK functionalities. It's the fastest way to play with our SDK.
SimpleApp has no custom logic at all: no custom boxes and transactions, no custom API and with empty ApplicationState and ApplicationWallet.

SimpleApp requires a single argument to start: the path to the user configuration file.
Under the hood it has to parse its config file using SettingsReader, and then initialize and run SidechainApp

	



















