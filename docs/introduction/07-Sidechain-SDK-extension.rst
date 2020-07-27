=======================
Sidechain SDK extension
=======================


Data serialization
##################

Any data like **Box**/**BoxData**/**Secret**/**Proposition**/**Proof**/**Transaction** shall provide a way to  serialize itself to bytes and provide a way to parse it from bytes.
Serialization is performed via special Serializer class. Any custom data, beside defining own Serializer and definition of parsing/serializing,
shall declare those Serializers for the SDK, thus SDK will be able to use proper serializer for custom data. So full actions for describe serialization/parsing for some
CustomData are the following:

``Implement BytesSerializable interface`` for ``CustomData``, i.e. ``functions byte[] bytes()`` and ``Serializer serializer()``, also implement ``public static CustomData parseBytes(byte[] bytes)`` function for parsing from bytes
  
* Create ``CustomDataSerializer``Class and implement ``ScorexSerializer interface``, and implement the following methods:  ``void serialize(CustomData customData, Writer writer)`` and ``CustomData parse(Reader reader)``;

* In your AppModule class (i.e. class which extends  AbstractModule, in SimpleApp it is SimpleAppModule) define Custom Serializer map, for example for boxes it could be ```Map<Byte, BoxSerializer<Box<Proposition>>> customBoxSerializers = new HashMap<>();``` where key is data type id and value is CustomSerializer for those data type id.
  
* Provide a unique id for that data type by implementing a special function. For example for box data type it is the function  ``public byte boxTypeId()``, for other data types the function name could be different and you will be obliged to implement it. 
  
* ``Map<Byte, BoxSerializer<Box<Proposition>>> customBoxSerializers = new HashMap<>();``

* Add your custom serializer into the map, for example it could be something  like ``customBoxSerializers.put((byte)MY_CUSTOM_BOX_ID, (BoxSerializer) CustomBoxSerializer.getSerializer());``
  
* Bind map with custom serializers to your application:
::
 
 TypeLiteral<HashMap<Byte, Common serializer type>() {})
       .annotatedWith(Names.named(Bound property name))
       .toInstance(Created map with custom serializers);
       
Where **Common serializer type** and **Bound property name** can have the following values 


+--------------------------------+----------------------------------------+
| Bound property name            | Common serializer type                 |
+================================+========================================+
| CustomBoxSerializers           | BoxSerializer<Box<Proposition>>>       |  
+--------------------------------+----------------------------------------+
| CustomBoxDataSerializers       | NoncedBoxDataSerializer<NoncedBoxData  |
|                                | <Proposition, NoncedBox<Proposition>>> |           
+--------------------------------+----------------------------------------+
| CustomSecretSerializers        | SecretSerializer<Secret>>              |           
+--------------------------------+----------------------------------------+
| CustomProofSerializers         | ProofSerializer<Proof<Proposition>>    |        
+--------------------------------+----------------------------------------+
| CustomTransactionSerializers   |  TransactionSerializer<BoxTransaction  |                                  
|                                |  <Proposition, Box<Proposition>>>      |
+--------------------------------+----------------------------------------+

Example: 

::

  bind(new TypeLiteral<HashMap<Byte, BoxSerializer<Box<Proposition>>>>() {})
       .annotatedWith(Names.named("CustomBoxSerializers"))
       .toInstance(customBoxSerializers);

where  ``BoxSerializer<Box<Proposition>>>``  - common serializer type ``"CustomBoxSerializers"`` - bound property name 
``customBoxSerializers`` - created map with all defined custom serializers. Overall we have the next expected type and property name.

Custom box creation
###################

  a) SDK Box extension Overview

To build a real application, a developer will need more than just receive, transfer and send coins back. A distributed app, built on a sidechain, will typically have to define some custom data that the sidechain users will be able to exchange according to a defined logic. Creation of new Boxes requires definition of new four classes. We will use name Custom Box as a definition for some abstract custom Box:


+---------------------------------------+------------------------------------------------------------------------------------+
| Class type                            | Class description                                                                  |
+=======================================+====================================================================================+
| Custom Box Data class                 | -- Contains all custom data definitions plus proposition for Box                   |
|                                       | -- Provide required information for serialization of Box Data                      |
|                                       | -- Define the way for creation new Custom Box from current Custom Box Data         |
+---------------------------------------+------------------------------------------------------------------------------------+
| Custom Box Data Serializer Singleton  | -- Define the way how to parse bytes from Reader into Custom Box Data object       |
|                                       | -- Define the way how to put boxData object into Writer                            |
|                                       | Parsing/Serialization itself could be defined in Custom Box Data class             |
+---------------------------------------+------------------------------------------------------------------------------------+
| Custom Box                            | Representation new entity in Sidechain, contains appropriate Custom Box Data class |
+---------------------------------------+------------------------------------------------------------------------------------+
| Custom Box Serializer Singleton       | -- Define the way how to parse bytes from Reader into Box Data object              |
|                                       | -- Define the way how to put boxData object into Writer                            |
|                                       | Parsing/Serialization itself could be defined in Box Data class                    |
+---------------------------------------+------------------------------------------------------------------------------------+

Custom Box Data class creation
##############################

SDK provides base class for any Box Data class: 

::

  AbstractNoncedBoxData<P extends Proposition, B extends AbstractNoncedBox<P, BD, B>, BD extends AbstractNoncedBoxData<P, B, BD>>


where

::
  
  P extends Proposition -- Proposition type for the box, for common purposes PublicKey25519Proposition could be used as it used in regular boxes
  BD extends AbstractNoncedBoxData<P, B, BD>

Definition of type for Box Data which contains all custom data for new custom box

::
  
  B extends AbstractNoncedBox<P, BD, B>
  
Definition of type for Box itself, required for description inside of new Custom Box data 
That base class provide next data by default:

::

  proposition of type P long value

value of that box if required, that value is important in case if Box is coin Box, otherwise it will be used in custom logic only. 
In common case for non-Coin box it could be always equal 1 

So the creation of new Custom Box Data will be created in following way:
``public class CustomBoxData extends AbstractNoncedBoxData<PublicKey25519Proposition, CustomBox, CustomBoxData>``

The new custom box data class  requires the following:

1. Custom data definition
  * Custom data itself
  * Hash of all added custom data shall be returned in ``public byte[] customFieldsHash()`` method, otherwise custom data will not be “protected”, i.e. some malicious actor        could change custom data during transaction creation. 
    
2. Serialization definition
  * Serialization to bytes shall be provided by Custom Box Data by overriding and implementation function public byte[] bytes(). That function shall serialize proposition, value and any added custom data.
  * Additionally definition of Custom Box Data id for serialization by overriding public byte boxDataTypeId() method, please check the serialization chapter for more information about using ids. 
  * Override public NoncedBoxDataSerializer serializer() method with proper Custom Box Data serializer. Parsing Custom Box Data from bytes could be defined in that class as well, please refer to the serialization chapter for more information about it

3. Custom Box creation
  * Any Box Data class shall provide the way how to create a new Box for a given nonce. For that purpose override the function public CustomBox getBox(long nonce). 


Custom Box Data Serializer class creation
#########################################

SDK provide base class for Custom Box Data Serializer
NoncedBoxDataSerializer<D extends NoncedBoxData> where D is type of serialized Custom Box Data
So creation of Custom Box Data Serializer can be done in next way:

:code:`public class CustomBoxDataSerializer implements NoncedBoxDataSerializer<CustomBoxData>`

That new Custom Box Data Serializer require next:

  1. Definition of function for writing Custom Box Data into the Scorex Writer by implementation of public void serialize(CustomBoxData boxData, Writer writer)function.

  2. Definition of function for reading Custom Box Data from Scorex Reader
by implementation of function public CustomBoxData parse(Reader reader)

  3. Class shall be converted to singleton, for example it can be done in following way:

::
  
  private static final CustomBoxDataSerializer serializer = new CustomBoxDataSerializer();

  private CustomBoxDataSerializer() {
   super();
  }

  public static CustomBoxDataSerializer getSerializer() {
   return serializer;
  }
  
Custom Box class creation
#########################

SDK provide base class for creation Custom Box:

:code:`public class CustomBox extends AbstractNoncedBox<PublicKey25519Proposition, CustomBoxData, CustomBoxBox>`

As a parameters for **AbstractNoncedBox** three template parameters shall be provided:
``P extends Proposition``- Proposition type for the box, for common purposes 
PublicKey25519Proposition could be used as it used in regular boxes
``BD extends AbstractNoncedBoxData<P, B, BD>`` -- Definition of type for Box Data which contains all custom data for new custom box
``B extends AbstractNoncedBox<P, BD, B>`` -- Definition of type for Box itself, required for description inside of new Custom Box data.

The Custom Box itself require implementation of next functionality:

  1. Serialization definition

    * Box itself shall provide the way to be serialized into bytes, thus method ``public byte[] bytes()`` shall be implemented 
    * Method ``public static CarBox parseBytes(byte[] bytes)`` for creation of a new Car Box object from bytes, 
    * Providing box type id by implementation of method ``public byte boxTypeId()`` which return custom box type id. And, finally, proper serializer for the Custom Box shall be returned by implementation of method ``public BoxSerializer serializer()``

Custom Box Serializer Class
###########################

SDK provides base class for ``Custom Box Serializer
BoxSerializer<B extends Box>`` where B is type of serialized Custom Box
So creation of **Custom Box Serializer** can be done in next way:
 ``public class CustomBoxSerializer implements NoncedBoxSerializer<CustomBox>``
The new Custom Box Serializer requires the following:

  1. Definition of method for writing Custom Box into the Scorex Writer by implementation of public void serialize(CustomBox box, Writer writer)method.
  2. Definition of method for reading Custom Box from Scorex Reader
by implementation of method public CustomBox parse(Reader reader)
  3. Class shall be converted to singleton, for example it could be done in next way:

    ::
    
      private static final CustomBoxSerializer serializer = new CustomBoxSerializer();

      private CustomBoxSerializer() {
       super();
      }

      public static CustomBoxSerializer getSerializer() {
       return serializer;
      }
      
      
Specific actions for extension of Coin-box
###########################################

Coin box is created and extended as a usual non-coin box, only one additional action is required: Coin box class shall also implements interface CoinsBox<P extends PublicKey25519Proposition> interface without any additional function implementations, i.e. it is a mixin interface.

Transaction extension
#####################

Transaction in SDK is represented by public abstract class BoxTransaction<P extends Proposition, B extends Box<P>> extends Transaction class. That class provides access to data like which boxes will be created, unlockers for input boxes, fee, etc. SDK developer could add custom transaction check by implement custom ApplicationState (see appropriate chapter for it)

ApplicationState and Wallet
###########################

 ApplicationState:
 
  ::
  
    interface ApplicationState {
    boolean validate(SidechainStateReader stateReader, SidechainBlock block);

    boolean validate(SidechainStateReader stateReader, BoxTransaction<Proposition, Box<Proposition>> transaction);

    Try<ApplicationState> onApplyChanges(SidechainStateReader stateReader, byte[] version, List<Box<Proposition>> newBoxes, List<byte[]> boxIdsToRemove);

    Try<ApplicationState> onRollback(byte[] version);
    }

For example, the custom application may have the possibility to tokenize cars by creation of Box entries - let’s call them CarBox. Each CarBox token should represent a unique car by having a unique VIN (Vehicle Identification Number). To do this Sidechain developer may define ApplicationState where to keep the list of actual VINs and reject transactions with CarBox tokens with VIN already existing in the system.

Overall next custom state checks could be done here:

  * public boolean validate(SidechainStateReader stateReader, SidechainBlock block) --  any custom block validation could be done here if function return false then block will note be accepted by Sidechain Node at all
  
  * public boolean validate(SidechainStateReader stateReader, BoxTransaction<Proposition, Box<Proposition>> transaction) -- any custom checks for transaction could be done here, if function return false then transaction is assumed as invalid and for example will not be included in a memory pool. 

  * public Try<ApplicationState> onApplyChanges(SidechainStateReader stateReader, byte[] version, List<Box<Proposition>> newBoxes, List<byte[]> boxIdsToRemove) -- any specific action after block applying in State could be defined here.
  
  * public Try<ApplicationState> onRollback(byte[] version) -- any specific action after rollback of State (for example in case of fork/invalid block) could be defined here
  
Application Wallet 
##################

The Wallet by default keeps user secret info and related balances. The actual data is updated when the new block is applied to the chain or when some blocks are reverted. Developers can specify custom secret types that will be processed by Wallet. But it may be not enough, so he may extend the logic using ApplicationWallet:

::

  interface ApplicationWallet {
    void onAddSecret(Secret secret);
    void onRemoveSecret(Proposition proposition);
    void onChangeBoxes(byte[] version, List<Box<Proposition>> boxesToUpdate, List<byte[]> boxIdsToRemove);
    void onRollback(byte[] version);
  }

For example, some developer needs to have some event-based data, like an auction slot that belongs to him and will start in 10 blocks and will expire in 100 blocks. So in ApplicationWallet he will additionally keep this event-based info and will react when a new block is going to be applied (onChangeBoxes method execution) to activate or deactivate that slot in ApplicationWallet.


Custom API creation 
###################

  Steps to extend the API:
  
    1. Create a class (e.g. MyCustomApi) which extends the ApplicationApiGroup abstract class (you could create multiple classes, for example to group functions by functionality).

    2. In a class where all dependencies are declared (e.g. SimpleAppModule in our Simple App example ) we need to create the following variable: List<ApplicationApiGroup> customApiGroups = new ArrayList<>();

    3. Create a new instance of the class MyCustomApi, and then add it to customApiGroups 

At this point MyCustomApi will be included in the API route, but we still need to declare the HTTP address. To do that, please:

  1. Override the basepath() method -
  
    ::
    
      public String basePath() {
       return "myCustomAPI";
      }

Where "myCustomAPI" is part of the HTTP path for that API group 


  2.  Define HTTP request classes -- i.e. the json body in the HTTP request will be converted to that request class. For example, if as “request” we want to have byte array data with some integer value, we could define the following class:
  
  ::
  
    public static class MyCustomRequest {
     byte[] someBytes;
     int number;

    public byte[] getSomeBytes(){
     return someBytes;
    }

    public void setSomeBytes(String bytesInHex){
     someBytes = BytesUtils.fromHexString(bytesInHex);
    }

    public int getNumber(){
     return number;
    }

    public void setNumber(int number){
    this.number = number;
    }
    }

Setters are defined to expect data from JSON. So, for the given MyCustomRequest we could use next JSON: 

    ::
    
      {
      "number": "342",
      "someBytes": "a5b10622d70f094b7276e04608d97c7c699c8700164f78e16fe5e8082f4bb2ac"
      }

 And it will be converted to an instance of the MyCustomRequest class with vin = 342, and someBytes = bytes which are represented by hex string "a5b10622d70f094b7276e04608d97c7c699c8700164f78e16fe5e8082f4bb2ac"


  3. Define a function to process the HTTP request: Currently we support three types of function’s signature:
  
      * ApiResponse custom_function_name(Custom_HTTP_request_type) -- a function that by default does not have access to SidechainNodeView. To have access to SidechainNodeViewHolder, this special call should be used: getFunctionsApplierOnSidechainNodeView().applyFunctionOnSidechainNodeView(Function<SidechainNodeView, T> function)
      
      * ApiResponse custom_function_name(SidechainNodeView, Custom_HTTP_request_type) -- a function that offers by default access to SidechainNodeView
      
      * ApiResponse custom_function_name(SidechainNodeView) -- a function to process empty HTTP requests, i.e. JSON body shall be empty
      
Inside those functions all required action could be defined, and with them also function response results. Responses could be based on SuccessResponse or ErrorResponse interfaces. The JSON response will be formatted by using the defined getters.  

  4. Add response classes

As a result of an API request some result shall be sent back via HTTP response. In a common case we could have two different types of  responses: operation is successful and some error had appeared during processing of the API request. SDK provides next way to declare those API responses:
For successful response implement SuccessResponse interface with data to be returned. That data shall be accessible via getters. Also that class shall have next annotation which requires for marshaling and correct convertation to JSON: @JsonView(Views.Default.class) . You could define here some other custom class for JSON marshaling. For example if some string shall be returned then next response class could be defined:

  ::
  
    @JsonView(Views.Default.class)
    class CustomSuccessResponce implements SuccessResponse{
    private final String response;

    public CustomSuccessResponce (String response) {
    this.response = response;
    }

    public String getResponse() {
    return response;
    }
    }

In such case API response will be represented in next JSON form:

  ::
  
    {"result": {“response” : “response from CustomSuccessResponse object”}}
    
In case if something going wrong and error shall be returned then response shall implements ErrorResponse interface which by default have next functions to be implemented:

public String code() -- error code

public String description() -- error description 

public Option<Throwable> exception() -- Caught exception during API processing

As a result next JSON will be returned in case of error:

  ::
  
    {
    "error": {
    "code": "Defined error code",
    "description": "Defined error description",
    "Detail": “Exception stack trace”
    }
    }
    
  5. Add defined route processing functions to route

  Override public List<Route> getRoutes() function by returning all defined routes, for example:

    ::
      
      List<Route> routes = new ArrayList<>();
      routes.add(bindPostRequest("getNSecrets", this::getNSecretsFunction, GetSecretRequest.class));
      routes.add(bindPostRequest("getNSecretOtherImplementation", this::getNSecretOtherImplementationFunction, GetSecretRequest.class));
      routes.add(bindPostRequest("getAllSecretByEmptyHttpBody", this::getAllSecretByEmptyHttpBodyFunction));
      return routes;
      
 Where "getNSecrets", "getNSecretOtherImplementation", "getAllSecretByEmptyHttpBody" are defined API end points; this::getNSecretsFunction, this::getNSecretOtherImplementationFunction, getAllSecretByEmptyHttpBodyFunction binded functions;
GetSecretRequest.class -- class for defining type of HTTP request



      
