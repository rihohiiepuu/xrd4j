# XRd4J

XRd4J is a Java library for building X-Road v6 Adapter Servers and clients. The library implements X-Road v6 [SOAP profile](https://github.com/vrk-kpa/X-Road/blob/develop/doc/Protocols/pr-mess_x-road_message_protocol.md) v4.0 and [Service Metadata Protocol](https://github.com/vrk-kpa/X-Road/blob/develop/doc/Protocols/pr-meta_x-road_service_metadata_protocol.md). The library takes care of serialization and deserialization of SOAP messages: built-in support for standard X-Road SOAP headers, only processing of application specific ```request``` and ```response``` elements must be implemented.

By default this library is processing SOAP Body elements in compatibility mode with older versions of X-Road protocol. This means that request messages must contain ```request``` wrapper and response messages must contain ```request``` and ```response``` wrappers. To skip automatic procession of ```request``` and ```response``` wrappers, method ```request.setProcessingWrappers(false)``` must be called before serialization or deserialization of messages is performed. The usage of ```setProcessingWrappers``` method is demonstrated in the examples below.

##### Modules:

* ```client``` : SOAP client that generates X-Road v6 SOAP messages that can be sent to X-Road Security Server. Includes request serializer and response deserializer.
* ```server``` : Provides an abstract servlet that can be used as a base class for Adapter Server implementations. Includes a request deserializer and a response serializer.
* ```common``` : General purpose utilities for processing SOAP messages and X-Road v6 message data models.
* ```rest``` : HTTP clients that can be used for sending requests to web services from Adapter Server.

### Maven Repository

#### Releases

All XRd4J versions are available through [CSC's Maven Repository](https://maven.csc.fi/repository/internal/).

Specify CSC's Maven Repository in a POM:

```XML
<repository>
  <id>csc-repo</id>
  <name>CSC's Maven repository</name>
  <url>https://maven.csc.fi/repository/internal/</url>
</repository>
```

If running ```mvn clean install``` generates the error

```
[ERROR] Failed to execute goal on project example-adapter: Could not resolve dependencies for project fi.vrk.xrd4j.tools:example-adapter:war:0.0.1-SNAPSHOT: Failed to collect dependencies at fi.vrk.xrd4j:common:jar:0.0.1: Failed to read artifact descriptor for fi.vrk.xrd4j:common:jar:0.0.1: Could not transfer artifact fi.vrk.xrd4j:common:pom:0.0.1 from/to csc-repo (https://maven.csc.fi/repository/internal/): sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target -> [Help 1]
```
try one of the two solutions below:


##### Solution 1

Skip certificate validation:

```
mvn install -Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true
```

##### Solution 2

Import CSC's Maven repository's certificate as a trusted certificate into ```cacerts``` keystore. See full [instructions](documentation/Import-a-Certificate-as-a-Trusted-Certificate.md).

#### Dependency Declaration

Declare the following dependencies in your POM-file:

```XML
<!-- Module: common-->
<dependency>
  <groupId>fi.vrk.xrd4j</groupId>
  <artifactId>common</artifactId>
  <version>0.1.0</version>
</dependency>

<!-- Module: client-->
<dependency>
  <groupId>fi.vrk.xrd4j</groupId>
  <artifactId>client</artifactId>
  <version>0.1.0</version>
</dependency>

<!-- Module: server-->
<dependency>
  <groupId>fi.vrk.xrd4j</groupId>
  <artifactId>server</artifactId>
  <version>0.1.0</version>
</dependency>

<!-- Module: rest-->
<dependency>
  <groupId>fi.vrk.xrd4j</groupId>
  <artifactId>rest</artifactId>
  <version>0.1.0</version>
</dependency>
```

### Documentation

Instructions for setting up an environment for XRd4J-related development can be found [here](documentation/Setting-up-Development-Environment.md).

Javadocs can be generated with the included script `generate-javadocs.sh`. The script will create `javadoc` folder with HTML documentation.

The most essential classes of the library are:

* ```fi.vrk.xrd4j.common.member.ConsumerMember``` : represents an X-Road consumer member that acts as a client that initiates a service call by sending a ServiceRequest.
* ```fi.vrk.xrd4j.common.member.ProducerMember``` : represents an X-Road producer member that produces services to X-Road.
* ```fi.vrk.xrd4j.common.message.ServiceRequest<?>``` : represents an X-Road service request that is sent by a ConsumerMember and received by a ProviderMember. Contains the sent SOAP request.
* ```fi.vrk.xrd4j.common.message.ServiceResponse<?, ?>``` : represents an X-Road service response message that is sent by a ProviderMember and received by a ConsumerMember. Contains the SOAP response.
* ```fi.vrk.xrd4j.client.serializer.AbstractServiceRequestSerializer``` : an abstract base class for service request serializers.
* ```fi.vrk.xrd4j.server.deserializer.AbstractCustomRequestDeserializer<?>``` : an abstract base class for service request deserializers.
* ```fi.vrk.xrd4j.server.serializer.AbstractServiceResponseSerializer``` : an abstract base class for service response serializers.
* ```fi.vrk.xrd4j.client.deserializer.AbstractResponseDeserializer<?, ?>``` : an abstract base class for service response deserializers.
* ```fi.vrk.xrd4j.client.SOAPClientImpl``` : a SOAP client that offers two methods for sending SOAPMessage and ServiceRequest objects.
* ```fi.vrk.xrd4j.server.AbstractAdapterServlet``` : an abstract base class for Servlets that implement SOAP message processing. Can be used as a base class for Adapter Server implementations.

##### Client

Client application must implement two classes:

* ```request serializer``` is responsible for converting the object representing the request payload to SOAP
  * extends ```AbstractServiceRequestSerializer```
    * serializes all the other parts of the SOAP message except ```request``` element
  * used through ```ServiceRequestSerializer``` interface
  * must implement ```serializeRequest``` method that serializes the ```request``` element to SOAP
* ```response deserializer``` parses the incoming SOAP response message and constructs the objects representing the response payload
  * extends ```AbstractResponseDeserializer<?, ?>```
    * deserializes all the other parts of the incoming SOAP message except ```request``` and ```response``` elements
	* type of the request and response data must be given as type parameters
  * used through ```ServiceResponseSerializer``` interface
  * must implement ```deserializeRequestData``` and ```deserializeResponseData``` methods

**N.B.** If HTTPS is used between the client and the Security Server, the public key certificate of the Security Server MUST be imported into `cacerts` keystore. [Detailed instructions here](documentation/Import-a-Certificate-as-a-Trusted-Certificate.md).

Main class (generated [request](examples/request1.xml), received [response](examples/response1.xml)):

```Java
  // Security server URL
  // N.B. If you want to use HTTPS, the public key certificate of the Security Server
  // MUST be imported into "cacerts" keystore
  String url = "http://security.server.com/";

  // Consumer that is calling a service
  ConsumerMember consumer = new ConsumerMember("FI_TEST", "GOV", "1234567-8", "TestSystem");

  // Producer providing the service
  ProducerMember producer = new ProducerMember("FI_TEST", "GOV", "9876543-1", "DemoService", "helloService", "v1");
  producer.setNamespacePrefix("ts");
  producer.setNamespaceUrl("http://test.x-road.fi/producer");

  // Create a new ServiceRequest object, unique message id is generated by MessageHelper.
  // Type of the ServiceRequest is the type of the request data (String in this case)
  ServiceRequest<String> request = new ServiceRequest<String>(consumer, producer, MessageHelper.generateId());

  // Please uncomment the following line to disable procession of "request" and "response" wrappers.
  // request.setProcessingWrappers(false);

  // Set username
  request.setUserId("jdoe");

  // Set request data
  request.setRequestData("Test message");

  // Application specific class that serializes request data
  ServiceRequestSerializer serializer = new HelloServiceRequestSerializer();

  // Application specific class that deserializes response data
  ServiceResponseDeserializer deserializer = new HelloServiceResponseDeserializer();

  // Create a new SOAP client
  SOAPClient client = new SOAPClientImpl();

  // Send the ServiceRequest, result is returned as ServiceResponse object
  ServiceResponse<String, String> serviceResponse = client.send(request, url, serializer, deserializer);

  // Print out the SOAP message received as response
  System.out.println(SOAPHelper.toString(serviceResponse.getSoapMessage()));

  // Print out only response data. In this case response data is a String.
  System.out.println(serviceResponse.getResponseData());

  // Check if response contains an error and print it out
  if (serviceResponse.hasError()) {
    System.out.println(serviceResponse.getErrorMessage().getErrorMessageType());
    System.out.println(serviceResponse.getErrorMessage().getFaultCode());
    System.out.println(serviceResponse.getErrorMessage().getFaultString());
    System.out.println(serviceResponse.getErrorMessage().getFaultActor());
  }
```

HelloServiceRequestSerializer (serialized [request](examples/request1.xml)):
```Java
  /**
   * This class is responsible for serialiazing request data to SOAP. Request data is wrapped
   * inside "request" element in SOAP body.
   */
  public class HelloServiceRequestSerializer extends AbstractServiceRequestSerializer {

    @Override
	/**
	 * Serializes the request data.
     * @param request ServiceRequest holding the application specific request object
     * @param soapRequest SOAPMessage's request object where the request element is added
     * @param envelope SOAPMessage's SOAPEnvelope object
	 */
    protected void serializeRequest(ServiceRequest request, SOAPElement soapRequest, SOAPEnvelope envelope) throws SOAPException {
	  // Create element "name" and put request data inside the element
      SOAPElement data = soapRequest.addChildElement(envelope.createName("name"));
      data.addTextNode((String) request.getRequestData());
    }
  }
```
HelloServiceRequestSerializer generates ```name``` element and sets request data ("Test message") as its value.

```XML
  <ts:request>
    <ts:name>Test message</ts:name>
  </ts:request>
```
HelloServiceResponseDeserializer ([response](examples/response1.xml) to be deserialized):
```Java
  /**
   * This class is responsible for deserializing "request" and "response" elements of the SOAP response message
   * returned by the Security Server. The type declaration "<String, String>" defines the type of request and
   * response data, in this case they're both String.
   */
  public class HelloServiceResponseDeserializer extends AbstractResponseDeserializer<String, String> {

    @Override
	/**
	 * Deserializes the "request" element.
	 * @param requestNode request element
	 * @return content of the request element
	 */
    protected String deserializeRequestData(Node requestNode) throws SOAPException {
      // Loop through all the children of the "request" element
      for (int i = 0; i < requestNode.getChildNodes().getLength(); i++) {
	    // We're looking for "name" element
        if (requestNode.getChildNodes().item(i).getLocalName().equals("name")) {
	      // Return the text content of the element
          return requestNode.getChildNodes().item(i).getTextContent();
        }
      }
	  // No "name" element was found, return null
      return null;
    }

    @Override
	/**
	 * Deserializes the "response" element.
	 * @param responseNode response element
	 * @param message SOAP response
	 * @return content of the response element
	 */
    protected String deserializeResponseData(Node responseNode, SOAPMessage message) throws SOAPException {
      // Loop through all the children of the "response" element
      for (int i = 0; i < responseNode.getChildNodes().getLength(); i++) {
	    // We're looking for "message" element
        if (responseNode.getChildNodes().item(i).getLocalName().equals("message")) {
	      // Return the text content of the element
          return responseNode.getChildNodes().item(i).getTextContent();
        }
      }
	  // No "message" element was found, return null
      return null;
    }
}
```

HelloServiceResponseDeserializer's ```deserializeRequestData``` method reads ```name``` elements's value ("Test message") under ```request``` element, and ```deserializeResponseData``` method reads ```message``` element's value ("Hello Test message! Greetings from adapter server!") under ```response``` element.

```XML
  <ts:request>
    <ts:name>Test message</ts:name>
  </ts:request>
  <ts:response>
    <ts:message>Hello Test message! Greetings from adapter server!</ts:message>
  </ts:response>
```

_Receiving an Image From Server_

The server returns images as base64 coded strings that are placed in SOAP attachments. Before being able to use the image the client must convert the base64 coded string to some other format. For example:

```Java
// Get the attached base64 coded image string
AttachmentPart attachment = (AttachmentPart) soapResponse.getAttachments().next();
// Get content-type of the attachment - if jpeg image, the content type is "image/jpeg"
String contentType = attachment.getContentType();
// Get the attachment as a String
String imgStr = SOAPHelper.toString(attachment);
// Convert base64 coded image string to BufferedImage
BufferedImage newImg = MessageHelper.decodeStr2Image(imgStr);
// Write the image to disk or do something else with it
// Content type must be set from "image/jpeg" to "jpg"
ImageIO.write(newImg, contentType, new File("/image/path"));
```

##### Server

Server application must implement three classes:

* ```servlet``` is responsible for processing all the incoming SOAP requests and returning a valid SOAP response
  * extends ```AbstractAdapterServlet```
    * serializes and deserializes SOAP headers, handles error processing
  * incoming requests are passed as ```ServiceRequest``` objects
  * outgoing responses must be returned as ```ServiceResponse``` objects
  * must implement ```handleRequest``` and ```getWSDLPath``` methods
* ```request deserializer``` parses the incoming SOAP request message and constructs the objects representing the request payload
  * extends ```AbstractCustomRequestDeserializer<?>```
    * type of the request data must be given as type parameter
  * used through ```CustomRequestDeserializer``` interface
  * must implement ```deserializeRequest``` method that deserializes the ```request``` element
* ```response serializer```
  * extends ```AbstractServiceResponseSerializer```
    * is responsible for converting the object representing the response payload to SOAP
  * used through ```ServiceResponseSerializer``` interface
  * must implement ```serializeResponse``` method that serializes the ```response``` element to SOAP

A working adapter example can be found under directory `example-adapter` ([documentation](example-adapter/README.md)).

Setting up SSL on Tomcat is explained [here](documentation/Setting-up-SSL-on-Tomcat.md).

Adapter servlet (received [request](examples/request1.xml), generated [response](examples/response1.xml)):

```Java
/**
 * This class implements one simple X-Road v6 compatible service: "helloService".
 * Service description is defined in "example.wsdl" file
 * that's located in WEB-INF/classes folder. The name of the WSDL file and the
 * namespace is configured in WEB-INF/classes/xrd-servlet.properties file.
 *
 * @author Petteri Kivimäki
 */
public class ExampleAdapter extends AbstractAdapterServlet {

    private Properties props;
    private final static Logger logger = LoggerFactory.getLogger(ExampleAdapter.class);
    private String namespaceSerialize;
    private String namespaceDeserialize;
    private String prefix;

    @Override
    public void init() {
        super.init();
        logger.debug("Starting to initialize Enpoint.");
        this.props = PropertiesUtil.getInstance().load("/xrd-servlet.properties");
        this.namespaceSerialize = this.props.getProperty("namespace.serialize");
        this.namespaceDeserialize = this.props.getProperty("namespace.deserialize");
        this.prefix = this.props.getProperty("namespace.prefix.serialize");
        logger.debug("Namespace for incoming ServiceRequests : \"" + this.namespaceDeserialize + "\".");
        logger.debug("Namespace for outgoing ServiceResponses : \"" + this.namespaceSerialize + "\".");
        logger.debug("Namespace prefix for outgoing ServiceResponses : \"" + this.prefix + "\".");
        logger.debug("Endpoint initialized.");
    }

    /**
     * Must return the path of the WSDL file.
     *
     * @return absolute path of the WSDL file
     */
    @Override
    protected String getWSDLPath() {
        String path = this.props.getProperty("wsdl.path");
        logger.debug("WSDL path : \"" + path + "\".");
        return path;
    }

    @Override
    protected ServiceResponse handleRequest(ServiceRequest request) throws SOAPException, XRd4JException {
        // Please uncomment the following line to disable procession of "request" and "response" wrappers.
        // request.setProcessingWrappers(false);

        // Create a new response serializer that serializes the response to SOAP
        ServiceResponseSerializer serializer = serializer = new HelloServiceResponseSerializer();
        ServiceResponse<String, String> response = null;

        // Process services by service code
        if (request.getProducer().getServiceCode().equals("helloService")) {
            // Process "helloService" service
            logger.info("Process \"helloService\" service.");
            // Create a custom request deserializer that parses the request
            // data from the SOAP request
            CustomRequestDeserializer customDeserializer = new CustomRequestDeserializerImpl();
            // Parse the request data from the request
            customDeserializer.deserialize(request, this.namespaceDeserialize);
            // Create a new ServiceResponse object
            response = new ServiceResponse<String, String>(request.getConsumer(), request.getProducer(), request.getId());
            // Set namespace of the SOAP response
            response.getProducer().setNamespaceUrl(this.namespaceSerialize);
            response.getProducer().setNamespacePrefix(this.prefix);
            logger.debug("Do message prosessing...");
            if (request.getRequestData() != null) {
                // If request data is not null, add response data to the
                // response object
                response.setResponseData("Hello " + request.getRequestData() + "! Greetings from adapter server!");
            } else {
                // No request data is found - an error message is returned
                logger.warn("No \"name\" parameter found. Return a non-techinal error message.");
                ErrorMessage error = new ErrorMessage("422", "422 Unprocessable Entity. Missing \"name\" element.");
                response.setErrorMessage(error);
            }
            logger.debug("Message prosessing done!");
            // Serialize the response to SOAP
            serializer.serialize(response, request);
            // Return the response - AbstractAdapterServlet takes care of
            // the rest
            return response;
        }
        // No service matching the service code in the request was found -
        // and error is returned
        response = new ServiceResponse();
        ErrorMessage error = new ErrorMessage("SOAP-ENV:Client", "Unknown service code.", null, null);
        response.setErrorMessage(error);
        serializer.serialize(response, request);
        return response;
    }

    /**
     * This class is responsible for serializing response data of helloService
     * service responses.
     */
    private class HelloServiceResponseSerializer extends AbstractServiceResponseSerializer {

        @Override
        /**
         * Serializes the response data.
         *
         * @param response ServiceResponse holding the application specific
         * response object
         * @param soapResponse SOAPMessage's response object where the response
         * element is added
         * @param envelope SOAPMessage's SOAPEnvelope object
         */
        public void serializeResponse(ServiceResponse response, SOAPElement soapResponse, SOAPEnvelope envelope) throws SOAPException {
            // Add "message" element
            SOAPElement data = soapResponse.addChildElement(envelope.createName("message"));
            // Put response data inside the "message" element
            data.addTextNode((String) response.getResponseData());
        }
    }

    /**
     * This class is responsible for deserializing request data of helloService
     * service requests. The type declaration "<String>" defines the type of the
     * request data, which in this case is String.
     */
    private class CustomRequestDeserializerImpl extends AbstractCustomRequestDeserializer<String> {

        /**
         * Deserializes the "request" element.
         *
         * @param requestNode request element
         * @return content of the request element
         */
        @Override
        protected String deserializeRequest(Node requestNode, SOAPMessage message) throws SOAPException {
            if (requestNode == null) {
                logger.warn("\"requestNode\" is null. Null is returned.");
                return null;
            }
            for (int i = 0; i < requestNode.getChildNodes().getLength(); i++) {
                // Request data is inside of "name" element
                if (requestNode.getChildNodes().item(i).getNodeType() == Node.ELEMENT_NODE
                        && requestNode.getChildNodes().item(i).getLocalName().equals("name")) {
                    logger.debug("Found \"name\" element.");
                    // "name" element was found - return the text content
                    return requestNode.getChildNodes().item(i).getTextContent();
                }
            }
            logger.warn("No \"name\" element found. Null is returned.");
            return null;
        }
    }
}
```

_Returning an Image From Server_

If the server needs to return images, this can be done converting images to base64 coded strings and returning them as SOAP attachments. For example:

```Java
// "img" can be BufferedImage or InputStream
// Image is converted to a base64 coded string, image type must be given
String imgstr = MessageHelper.encodeImg2String(img, "jpg");
// Image string is added as an attachment, content type must be set
AttachmentPart ap = response.getSoapMessage().createAttachmentPart(imgstr, "jpg");
// Set Content-Type header - Security Server does not accept "jpg" and "image/jpeg" can not
// be used with createAttachmentPart because "imgstr" is not an image object. Therefore the
// content type "image/jpeg" must be set afterwards
ap.addMimeHeader("Content-Type", "image/jpeg");
// The attachment is added to the SOAP message
response.getSoapMessage().addAttachmentPart(ap);
```

# Credits

XRd4J library was originally developed by Petteri Kivimäki (https://github.com/petkivim) during 2014-2017. In June 2017 it was agreed that Population Register Centre (VRK) takes maintenance responsibility.
