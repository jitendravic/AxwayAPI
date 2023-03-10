# OpenAPI Validator for Axway API-Management

This project provides a library that you can use to check the request or response against the API specification imported into the API Manager or an external API specification. Both the body (Request, Response) and the request parameters (Query, Path) are checked.  
Based on the API path, HTTP verb, and headers, the correct model is loaded from the API specification and used for verification.  
The following illustrates how it works and behaves in the API-Gateway at runtime:  

![OpenAPI Validation](images/openapi-validation-overview.png)

## Installation

To install, download the [release package](https://github.com/Axway-API-Management-Plus/openapi-validator/releases) and install it under `ext/lib`. After that, restart the API Gateway. It is recommended to make the jars files known in the Policy Studio as well as it is describe here: https://docs.axway.com/bundle/axway-open-docs/page/docs/apim_policydev/apigw_poldev/general_ps_settings/index.html#runtime-dependencies

## Setup

To call the OpenAPI validator, use a scripting filter. You can use Groovy or any other supported language here. You can use it to validate requests and responses.  

### Validate request

Use a request policy and, for example, a custom property to enable or disable the check. The validator can be included using a scripting filter as in the following example.  

![Sample request policy](images/sample-policy.png)  

Here is an example for Groovy:  

```groovy
import com.axway.apim.openapi.validator.OpenAPIValidator;
import com.vordel.trace.Trace

def invoke(msg)
{
    // Get the ID of the API currently processed
    def apiId = msg.get("api.id");
    // Get/Create an OpenAPIValidator instance based on the API-ID
    // For additional options please see: 
    // https://github.com/Axway-API-Management-Plus/openapi-validator#get-an-openapi-validator-instance
    def validator = OpenAPIValidator.getInstance(apiId, "apiadmin", "changeme");
    
    // Optionally you can configure an internal cache. For more details please see:
    // https://github.com/Axway-API-Management-Plus/openapi-validator/issues/2
    // validator.getExposurePath2SpecifiedPathMap().setMaxSize(5000);
    
    // By default query parameters are decoded by default. You may turn this off for each 
    // individual validator instance
    // validator.setDecodeQueryParams(false);
    
    // Get required parameters for the validation
    def payload = bodyAsString(msg.get('content.body'));
    def path = msg.get("http.request.path");
    def verb = msg.get("http.request.verb");
    def queryParams = msg.get("params.query");
    def headers = msg.get("http.headers");
    def contentHeaders = msg.get("http.content.headers");
    try {
        // Content-Headers contains the required Content-Type header - Merge them into the headers attribute
        headers.addHeaders(contentHeaders);
        // Call the validator itself
        def rc = validator.isValidRequest(payload, verb, path, queryParams, headers);
        Trace.info('rc: ' + rc);
        return rc;
        // If you would like to return the validation messages, use the following method, that returns you a 
        // ValidationReport object.
        // def validationReport = validator.validateRequest(payload, verb, path, queryParams, headers);
        // if (validationReport.hasErrors() ) {
        //    msg.put("circuit.failure.reason", validationReport.getMessages() );
        //    msg.put("http.response.status", 400 );
        //    return false;
        // } else {
        //   return true;
        // }
    } catch (Exception e) {
        Trace.error('Error validating request', e);
        return false;
    }
}

def bodyAsString(body) {
    if (body == null) {
        return null
     }
     try {
         return body.getInputStream(0).text
     } catch (IOException e) {
         Trace.error('Error while converting ' + body.getClass().getCanonicalName() + ' to java.lang.String.', e)
         return null
     }
}
```

### Validate response

Analogously, you can also check the response. Since the response scheme depends on the status code, this must also be passed.

```groovy
import com.axway.apim.openapi.validator.OpenAPIValidator;
import com.vordel.trace.Trace

def invoke(msg)
{
    def apiId = msg.get("api.id");
    def validator = OpenAPIValidator.getInstance(apiId, "apiadmin", "changeme");
    def payload = bodyAsString(msg.get('content.body'));
    def path = msg.get("http.request.path");
    def verb = msg.get("http.request.verb");
    def status = msg.get("http.response.status");
    def headers = msg.get("http.headers");
    def contentHeaders = msg.get("http.content.headers");
    Trace.debug('Calling OpenAPIValidator: [path: ' + path + ', verb: ' + verb + ', status: ' + status + ']');
    try {
        // Content-Headers contains the required Content-Type header - Merge them into the headers attribute
        headers.addHeaders(contentHeaders);
        def rc = validator.isValidResponse(payload, verb, path, status, headers);
        return rc;
        // If you would like to return the validation messages, use the following method, that returns you a 
        // ValidationReport object.
        // def validationReport = validator.validateResponse(payload, verb, path, status, headers);
        // if (validationReport.hasErrors() ) {
        //    msg.put("circuit.failure.reason", validationReport.getMessages() );
        //    msg.put("http.response.status", 400 );
        //    return false;
        // } else {
        //   return true;
        // }
    } catch (Exception e) {
        Trace.error('Error validating response', e);
        return false;
    }
}

def bodyAsString(body) {
    if (body == null) {
        return null
     }
     try {
         return body.getInputStream(0).text
     } catch (IOException e) {
         Trace.error('Error while converting ' + body.getClass().getCanonicalName() + ' to java.lang.String.', e)
         return null
     }
}
```

## Get an OpenAPI-Validator instance

The OpenAPI validator can be created in several ways. It is implemented as a singleton. This means that only one instance exists per API specification/API ID. Basically, the validator is instantiated based on a Swagger 2.0 or OpenAPI 3.0.x specification. You can provide it in JSON or YAML format.

### Based on the API-Manager API-ID

With this option, you pass the ID of the API in the API Manager. The OpenAPI validator uses the credentials to load the associated API specification (Swagger 2.0, OpenAPI 3.0) and initialize itself with it. This process is done only on the first call for each API-ID. 
The user must be a User, which has access to this API. The user must be a member of an organization that can see this API.
You can also specify the URL of the API manager, otherwise `https://localhost:8075` is used as default.

```
OpenAPIValidator.getInstance(apiId, "apiadmin", "changeme");
// or
OpenAPIValidator.getInstance(apiId, "user", "password", "https://manager.customer.com");
```

By default, the OpenAPI validator uses the API specification generated by the API-Manager during import, which is also associated with the Frontend-API. So this is also the API-Specification that users can download from the API Catalog.

Alternatively, you can instantiate an OpenAPI validator that should use the original imported backend API specification. You have to specify an additional boolean parameter like in the following example.

```
OpenAPIValidator.getInstance(apiId, "apiadmin", "changeme", true);
// or
OpenAPIValidator.getInstance(apiId, "user", "password", "https://manager.customer.com", true);
```

### Using an Inline Swagger

With this procedure, you pass the API specification as a string to the OpenAPI validator. You can read this from the KPS via policies, for example. In this case, a hash value is determined for the API specification and an OpenAPI Validator instance is created for each hash value.

```
def swagger = msg.get("swaggerAsString");
def validator = OpenAPIValidator.getInstance(swagger);
```

### Using a URL

With this option you specify the URL that the API specification returns. For example: https://petstore.swagger.io/v2/swagger.json. Authentication is not currently supported. Please create an issue if this is necessary.

```
def validator = OpenAPIValidator.getInstance("https://petstore.swagger.io/v2/swagger.json");
```


## API Management Version Compatibilty

This artefact has been tested with API-Management Versions


Please let us know, if you encounter any [issues](https://github.com/jitendravic/AxwayAPI/issues) with your API-Manager version.  


