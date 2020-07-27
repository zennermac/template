====================================
Car Registry Tutorial
====================================

Car Registry App high level overview
************************************

The Car Registry app is then an example of a sidechain that implements specific custom data and logic. The purpose of the application is to manage a simplified service that keeps
records of existing cars and their owners. It’s simplified as sidechain users will be able to register cars by simply paying a transaction fee, while in a real world scenario, 
the ability to create a car will be bound by the presentation of say a certificate signed by the Department of Motor Vehicles or analogous authority, or some other consensus 
mechanism that guarantees that the car really exists in the real world and it’s owned by a user with a given public key.
Accepting that in our example cars will just show up in sidechain, we want to build an application that can store information that identifies a specific car, such as vehicle 
identification number, model, production year, color (etc)... 
We’ll also want that cars’ owners could prove their ownership of the cars without disclosing information about their identity. We also want users to sell and buy cars,
against ZEN coins. 

So, the starting point of the development process is the data representation. A car is an example of a non-coin box because it represents some entity, but not money. 
Another example of a non-coin box is a car which is selling. We need another box for a selling car because a common car box doesn't have additional data like sale price, 
seller proposition address etc. For the money representation standard Regular Box is used (Regular box is coin box), that box is provided by SDK. Besides new entities CarBox
and CarSellOrder we also need to define a way for creating/destroying those new entities. For that purpose new transactions shall be defined: transaction for creating new car, 
transaction which move CarBox to CarSellOrder, transaction which declare car selling, i.e. moving CarSellOrder to the new CarBox. All created transactions are not put into the
memory pool automatically, so a raw transaction in hex representation shall be put by /transaction/sendTransaction API request. In summary we will add next car boxes and 
transactions:

Entities: 

+------------------+-----------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------+
| Entity name      | Entity description                                                                      | Entity fields                                                                               |
+==================+=========================================================================================+=============================================================================================+
| CarBox           | Box which contains car box data, which could be stored and operated in Sidechain        | boxData -- contains  car box data                                                           |
+------------------+-----------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------+
| CarBoxData       | Description of the car by using defined properties                                      | vin -- vehicle identification number which contains unique identification number of the car |
|                  |                                                                                         | year -- vehicle year production                                                             |
|                  |                                                                                         | model -- car model                                                                          |
|                  |                                                                                         | color -- car color                                                                          |
|                  |                                                                                         | description -- car description                                                              |
+------------------+-----------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------+
| CarSellOrder     | Box which contains car sell order data, which could be stored and operated in Sidechain | boxData -- contains  car sell order data                                                    |
+------------------+-----------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------+
| CarSellOrderData | Description of the car which are in sell status                                         | sellerProposition --  seller proposition, i.e. receiver of money for sold car.              |
|                  |                                                                                         | vin -- selling car vin                                                                      |
+------------------+-----------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------+

Transactions which allow to perform next boxes transitions

+------------------------------+--------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------+--------------------+---------------------------------------------------------------------------------------------------------------------+
| Transaction name             | Input parameters                                             | Input parameters purpose                                                                                                                  | Output boxes       | Output boxes purpose                                                                                                |
+==============================+==============================================================+===========================================================================================================================================+====================+=====================================================================================================================+
| Car creation transaction     | Regular Box                                                  | For paying fee                                                                                                                            | Car Box            | Wanted new Car Box                                                                                                  |
|                              +--------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------+--------------------+---------------------------------------------------------------------------------------------------------------------+
|                              | Fee value                                                    | How much fee will be paid for the transaction                                                                                             | Regular Box        | Change for fee                                                                                                      |
|                              +--------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------+--------------------+---------------------------------------------------------------------------------------------------------------------+
|                              | Car proposition                                              | Owner car proposition as PublicKey25519Proposition                                                                                        |                    |                                                                                                                     |
|                              +--------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------+--------------------+---------------------------------------------------------------------------------------------------------------------+
|                              | Vehicle identification number and any other car related data | Identification of the new car                                                                                                             |                    |                                                                                                                     |
+------------------------------+--------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------+--------------------+---------------------------------------------------------------------------------------------------------------------+
| Car sell Order transaction   | Car Box                                                      | Box which identify car for selling, initial car box will be opened and no longer is valid, thus in any case new Car Box shall be created  | Car sell order Box | Representation of car in sell state, also contains additional information like seller coin box proposition address  |
|                              +--------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------+--------------------+---------------------------------------------------------------------------------------------------------------------+
|                              | Seller proposition for coin box                              | Where money will be sent to                                                                                                               |                    |                                                                                                                     |
+------------------------------+--------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------+--------------------+---------------------------------------------------------------------------------------------------------------------+
| Car buying order transaction | Car sell order Box                                           | Identify car for selling, contains seller coin box proposition address                                                                    | Car box            | New owner car box, with buyer proposition address                                                                   |
|                              +--------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------+--------------------+---------------------------------------------------------------------------------------------------------------------+
|                              | Payment regular box id                                       | Id of box with money                                                                                                                      | Regular Box        | New coin box which could be opened by seller which contains coins for selling car                                   |
|                              +--------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------+--------------------+---------------------------------------------------------------------------------------------------------------------+
|                              | buyerProposition                                             | Buyer proposition where money shall be sent                                                                                               |                    |                                                                                                                     |
+------------------------------+--------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------+--------------------+---------------------------------------------------------------------------------------------------------------------+

Car registry implementation
***************************

First of all we need to define new boxes. 
As described before, a Car Box is a non-coin box. As defined before we need Car Box Data class as well for describing custom data. So we need to define CarBox and CarBoxData as separate classes for setting proper way to serialization/deserialization. 

So overall next classes will be created:

  ::
    
    public class CarBox extends AbstractNoncedBox<PublicKey25519Proposition, CarBoxData, CarBox>
 
  ::
    
    public class CarBoxSerializer implements BoxSerializer<CarBox>

  ::
    
    public class CarBoxData extends AbstractNoncedBoxData<PublicKey25519Proposition, CarBox, CarBoxData>

  ::
    
    public class CarSellOrder extends AbstractNoncedBox<PublicKey25519Proposition, CarSellOrderData, CarSellOrder>

  ::
  
    public class CarSellOrderSerializer implements BoxSerializer<CarSellOrder>
   
  ::
  
    public class CarSellOrderData extends AbstractNoncedBoxData<PublicKey25519Proposition, CarSellOrder, CarSellOrderData>
    
  ::
  
    public class CarSellOrderDataSerializer implements NoncedBoxDataSerializer<CarSellOrderData>


Implementation of CarBoxData
****************************
  
  CarBoxData is implemented according description from “Custom Box Data Creation” chapter as public class CarBoxData extends AbstractNoncedBoxData<PublicKey25519Proposition, CarBox, CarBoxData> with custom data as:

    ::
    
        private final BigInteger vin;
        private final int year;
        private final String model;
        private final String color;
        private final String description;
        
        public byte[] bytes() {
         return Bytes.concat(
             proposition().bytes(),
             Longs.toByteArray(value()),
             Ints.toByteArray(year),
             Ints.toByteArray(model.getBytes().length),
             model.getBytes(),
             Ints.toByteArray(color.getBytes().length),
             color.getBytes(),
             Ints.toByteArray(description.getBytes().length),
             description.getBytes(),
             vin.toByteArray()
         );
        }

1. Serialization is implemented as SDK developer, as described before, shall include proposition and value into serialization. Ordering is not important.
2. CarBoxData shall have a value parameter as a Scorex limitation, but in our business logic CarBoxData does not use that data at all because each car is unique and doesn't have any inherent value. Thus value is hidden, i.e. value is not present in the constructor parameter and just set by default to “1” in the class constructor.
3. public byte[] customFieldsHash() shall be implemented because we introduce some new custom data.

Implementation of CarBoxDataSerializer
**************************************

CarBoxDataSerializer is implemented according to the description from “Custom Box Data Serializer Creation” chapter as 
public class CarBoxDataSerializer implements NoncedBoxDataSerializer<CarBoxData>. 
Nothing special to note about.

Implementation of CarBox
************************

CarBox is implemented according to description from “Custom Box Class creation” chapter as
public class CarBox extends AbstractNoncedBox<PublicKey25519Proposition, CarBoxData, CarBox>
Few comments about implementation:

  1. As a serialization part SDK developer shall include long nonce as a part of serialization, thus serialization is implemented in next way:
  
    :: 
        public byte[] bytes()
        {
         return Bytes.concat(
             Longs.toByteArray(nonce),
             CarBoxDataSerializer.getSerializer().toBytes(boxData)
         );
        }
  
  2. CarBox defines his own unique id by implementation of the function public byte boxTypeId(). Similar function is defined in CarBoxData but it is a different ids despite value returned in CarBox and CarBoxData is the same.
  

Implementation of CarBoxSerializer
**********************************

CarBoxSerializer is implemented according to the description from “Custom Box Data Serializer Creation” chapter as 
public class CarBoxSerializer implements BoxSerializer<CarBox>. 
Nothing special to note about.

Implementation of CarSellOrderData
**********************************

CarSellOrderData is implemented according description from “Custom Box Data Creation” chapter as public class CarSellOrderData extends AbstractNoncedBoxData<PublicKey25519Proposition, CarSellOrder, CarSellOrderData> with custom data as:
private final PublicKey25519Proposition sellerProposition;
private final BigInteger vin;

Few comments about implementation:
  1. Proposition and value shall be included in serialization as it done in CarBoxData 
  2. Id of that box data shall different than in CarBoxData   

      
Implementation of CarSellOrderDataSerializer
********************************************

CarSellOrderDataSerializer is implemented according to the description from “Custom Box Data Serializer Creation” chapter as 
public class CarSellOrderDataSerializer implements NoncedBoxDataSerializer<CarSellOrderData>. 
Nothing special to note about.

Implementation of CarSellOrder
******************************

CarSellorder is implemented according to description from “Custom Box Class creation” chapter as
public class CarSellOrder extends AbstractNoncedBox<PublicKey25519Proposition, CarSellOrderData, CarSellOrder>
Nothing special to note about.

Extend API by creating new transactions Car creation transaction and Car sell Order transaction
***********************************************************************************************

For our purpose we need to define two transaction  Car creation transaction and Car sell Order transaction  so according custom API extension manual we shall do next: 

a) Create a new class CarApi which extends ApplicationApiGroup class, add that new class to Route by it in SimpleAppModule, like described in Custom API manual. In our case it is done in CarRegistryAppModule by 

  * Creating customApiGroups as a list of custom API Groups:
  * List<ApplicationApiGroup> customApiGroups = new ArrayList<>();
  * Adding created CarApi into customApiGroups: 
  customApiGroups.add(new CarApi());
  * Binding that custom api group via dependency injection:
    ::
    
      bind(new TypeLiteral<List<ApplicationApiGroup>> () {})
      .annotatedWith(Names.named("CustomApiGroups"))
      .toInstance(customApiGroups);
      
b) Define Car creation transaction.

  1. Defining request class/JSON request body
     As input for the transaction we expected: 
     Regular box id  as input for paying fee; 
     Fee value; 
     Proposition address which will be recognized as a Car Proposition; 
     Vehicle identification number of car. So next request class shall be created:
     
  ::
  
    public static class CreateCarBoxRequest {
    private BigInteger vin;
    private int year;
    private String model;
    private String color;
    private String description;
    private PublicKey25519Proposition carProposition;

    private int fee;
    private String boxId;

    public BigInteger getVin() {
        return vin;
    }

    public void setVin(String vin) {
        this.vin = new BigInteger(vin);
    }


    public int getYear() {
        return year;
    }

    public void setYear(int year) {
        this.year = year;
    }

    public String getModel() {
        return model;
    }

    public void setModel(String model) {
        this.model = model;
    }

    public String getColor() {
        return color;
    }

    public void setColor(String color) {
        this.color = color;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    public PublicKey25519Proposition getCarProposition() {
        return carProposition;
    }

    public void setCarProposition(String propositionHexBytes) {
        byte[] propositionBytes = BytesUtils.fromHexString(propositionHexBytes);
        carProposition = new PublicKey25519Proposition(propositionBytes);
    }


    public int getFee() {
        return fee;
    }

    public void setFee(int fee) {
        this.fee = fee;
    }

    public String getBoxId() {
        return boxId;
    }

    public void setBoxId(String boxId) {
        this.boxId = boxId;
    }
    }

Request class shall have appropriate setters and getters for all class members, also class members' names define structure for related JSON structure according  jackson library so next JSON structure is expected to be set: 

  ::
  
    {
    "vin": "30124",
    “year”: 1984,
    “model”: “Lamborghini”“ color”: ”deep black”“ description”: ”best car in the world” "carProposition": "a5b10622d70f094b7276e04608d97c7c699c8700164f78e16fe5e8082f4bb2ac",
    "fee": 1,
    "boxId": "d59f80b39d24716b4c9a54cfed4bff8e6f76597a7b11761d0d8b7b27ddf8bd3c"
    }

Few interesting moments: setter input parameter could have differ type than set class member, it’s allow us to do all necessary conversation in setters; byte data is represented initially as a hex string, which converted to bytes by BytesUtils.fromHexString() function.

2. Define response for Car creation transaction, result of transaction shall be defined by implementing SuccessResponse interface with class members which shall be returned as API response, all members shall have properly set getters, also response class shall have proper annotation @JsonView(Views.Default.class) thus jackson library is able correctly represent response class in JSON format. In our case we expect to return transaction bytes, so response class is next:

  ::
  
    @JsonView(Views.Default.class)
    class CarResponse implements SuccessResponse {
    private final String createCarTxBytes;

    public CarResponse(String createCarTxBytes) {
        this.createCarTxBytes = createCarTxBytes;
    }

    public String carTxBytes() {
        return createCarTxBytes;
    }

    public String getCreateCarTxBytes() {
        return createCarTxBytes;
    }
    }

3. Define Car creation transaction

  :: 
  
    private ApiResponse createCar(SidechainNodeView view, CreateCarBoxRequest ent)

As a first parameter we pass reference to SidechainNodeView, second reference is previously defined class on step 1 for representation of JSON request. 
During transaction creation we need to do next:

  * check is input box secret is present in our wallet at all
  * is stored coins in that box is enough to pay fee
  * calculate fee for change 
  * create RegularBoxData for change for fee 
  * create new CarBoxData according JSON request data
  * create inputs from input box and outputs RegularBoxData for change and new CarBoxData  
  * calculate additional data like timestamp
  * get list of fake proof which are required to build message to be signed: List fakeProofs = Collections.nCopies(inputIds.size(), null);
  * build transaction bytes to be signed instead of real proof put some fake proof into from the previous step. For transaction creation a special factory shall be used. Access     to that factory could be achieved by call getSidechainCoreTransactionFactory()function:
    SidechainCoreTransaction unsignedTransaction =
    getSidechainCoreTransactionFactory().create(inputIds, outputs, fakeProofs, ent.fee, timestamp);
    byte[] messageToSign = unsignedTransaction.messageToSign();
  * create proof by sign transaction by private key of input box
  * create new transaction 
  * add to the CarResponse created transaction bytes

4. Define request for Car sell order transaction CreateCarSellOrderRequest  similar as it was done for Car creation transaction request

5. Define response for Car sell order transaction CreateCarSellOrderResponce as it was done for Car creation transaction response

6. Define Car Sell order transaction
  private ApiResponse createCarSellOrder(SidechainNodeView view, CreateCarSellOrderRequest ent) 
  Required actions are similar as it was done for Create Car transaction, but we don’t need to worry about fee, i.e. fee is set as 0. Main idea is a moving Car Box into        CarSellOrderBox
  
7. Define Car sell order response 
As a result of Car sell order we want to get hex byte representation of that transaction

8. Define request class for accepting Car Sell Order Transaction, with input:
String carSellOrderId;
String paymentRegularBoxId;
PublicKey25519Proposition buyerProposition;

9. Define response class for CarSellOrder transaction
Response shall contains hex representation of transaction bytes, thus response class are next:

  ::
  
    @JsonView(Views.Default.class)
    class AcceptCarSellOrderResponce implements SuccessResponse {
    private final String acceptedCarSellOrderTxBytes;

    public AcceptCarSellOrderResponce(String acceptedCarSellOrderTxBytes) {
        this.acceptedCarSellOrderTxBytes = acceptedCarSellOrderTxBytes;
    }

    public String acceptedCarSellOrderTxBytes() {
        return acceptedCarSellOrderTxBytes;
    }

    public String getAcceptedCarSellOrderTxBytes() {
        return acceptedCarSellOrderTxBytes;
    }
    }
    
10. Create AcceptCarSellorder transaction










