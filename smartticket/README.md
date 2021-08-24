# Smart Interoperability Layer

The **Smart Interoperability Layer** (InterOp Layer) acts as a flexible, light weight mapping service to allow communication between a range of different REST based web service APIs. Making use of *RML*, provided by an arbitrary remote mapping service, and the corresponding *rml mapping files*, requests are lifted to a *semantic level* from where requests for target services can be queried.

For a quick start, use the provided collection `InterOpLayerRequests.json` in the root directory.

## Installation
Build the project using
```
mvn clean install
````

Then either start the generated jar on its own from the target folder as a standalone springboot service via
```
java -jar target/smartticket.jar
```

However, it is recommended to use the *docker-compose* file in the project parent folder to start the ticket service along with a suitable RML mapping service.

## Connect to a RML mapping service

In order to specify a RML mapping service to be used by the interop layer, send a POST request with the URL to be used as parameter to the endpoint
```
http://<address-and-port-of-interop-layer>/rmlService/
```
## Using the InteropLayer
The InteropLayer handles requests in the following steps:
1. Receive a request at the endpoint
```
http://<address-and-port-of-interop-layer>/ticket
```
With the body containing the request following the API expected by the *client service* and with the following additional parameters:
  1. The *name of the target service* as specified in the registry. The name is necessary to determine the endpoint to forwared the mapped request to and to get the correct *request template* from the template registry.
  2. The *file name of the mapping file* to be used to lift the incoming request on a semantic level
2. The interOp layer will forward the request to the specified RML mapping service to be mapped to turtle RDF using the specified mapping file, receive the response and store it in its internal request triple store.
3. From the internal triple store, the interopService will query the necessary information and put it into the *request template* for the target service to build a proper request for the target API.
4. The response of the target service will be received by the InterOp layer and forwarded to the initial client.

### Registering services to be called
Registering target services in the interop layer is done by sending a *ttl RDF* to the registry endpoint that describes the service to be called. This ttl needs to provide information about the endpoint when the specific service is addressed, as well as the location of the request template to be used to form a valid request for that service.
```
@prefix : <http://www.smart-maas.eu/#> .
@prefix wsdl: <https://www.w3.org/ns/wsdl20-rdf#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .

:SmartMaaSServiceDescription a wsdl:Description .

:[ID of the service to be called] a wsdl:service ;
	wsdl:endpoint [Endpoint to be called for this service] ;
	:hasTemplate [location of the request template to be used] .
```
### Request Templates and Mapping Files
Two types of configuration files are necessary to make sure the communication between two services works as expected: a *request template* that describes the structure and necessary data for the client service, and a *mapping file* that describes how to lift the data send to a semantic level with the help of the previously mentioned *mapping service*

#### Request templates

An example for a simple JSON request template is given below. For each property that sould appear in the acutal request, the *type* can be specified, and at least a *default* value or an *accessor* needs to be provided. In case of an accessor, a SPARQL SELECT query will be performed on the semantic representation of the service call, using the given resource.
```
{
	"$id": "https://example.com/ticket.reguest.schema.json",
	"$schema": "http://json-schema.org/draft-07/schema#",
	"title": "Ticket Booking",
	"type": "object",
	"properties": {
		"name": {
			"type": "string",
			"default": "booking1",
		},
		"begin": {
			"type": "dateTime",
			"accessor": "gr:validFrom"
		},
		"end": {
			"type": "dateTime",
			"accessor": "gr:validThrough"
		}
	}
}
```

#### Mapping Files

For a detailed explanation on how to write RML mapping files, see https://rml.io/specs/rml/
### Endpoints

`POST /rmlService?serviceURL=[RMLMappingService endpoint]` : Set the address of the RML mapping service to be used

`POST /serviceTemplate` : Register a new service, providing a ttl service template as payload

`POST /ticket?targetService=<ServiceName as given in the registry>&mappingFileName=<Filename of the mapping file to be used>` : Lift the request given in the body on a semantic level using the given mapping file, and perform a request to the specified service.