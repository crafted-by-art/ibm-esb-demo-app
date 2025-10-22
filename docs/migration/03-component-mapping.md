# 3. Component Mapping: IBM ACE to Apache Camel

This document provides detailed mappings from IBM ACE components to their Apache Camel equivalents.

## Core Component Mapping

| IBM ACE Component | Apache Camel Component | Description |
|-------------------|------------------------|-------------|
| **HTTPInput** | `rest().get/post()` or `from("jetty:http://...")` | HTTP endpoint listener |
| **HTTPReply** | Automatic in REST DSL | HTTP response sender |
| **Compute Node (ESQL)** | `process()` or `bean()` | Custom logic processor |
| **Filter Node** | `filter(predicate)` | Conditional routing |
| **RouteToLabel** | `to("direct:...")` | Internal routing |
| **Label** | `from("direct:...")` | Named route endpoint |
| **JavaCompute** | `process()` or `bean()` | Java processor |
| **Mapping Node** | `marshal/unmarshal()` | Data transformation |
| **TryCatch** | `doTry().doCatch()` | Exception handling |
| **Throw** | `throwException()` | Explicit exception |
| **Database (JDBC)** | `to("sql:...")` or JPA | Database operations |
| **FileRead** | `from("file:...")` | File reading |
| **FileWrite** | `to("file:...")` | File writing |
| **MQInput** | `from("jms:...")` | JMS message consumer |
| **MQOutput** | `to("jms:...")` | JMS message producer |
| **FlowOrder** | `process()` with sequencing | Sequential processing |

## REST API Layer Mapping

### IBM ACE REST API
```
// swagger.json definition
{
  "paths": {
    "/index": {
      "get": {...},
      "post": {...}
    }
  }
}

// Flows
- TeaRESTAPI.msgflow (HTTPInput)
- getIndex.subflow
- postSetTea.subflow
```

### Apache Camel REST DSL
```java
restConfiguration()
    .component("servlet")
    .bindingMode(RestBindingMode.json)
    .dataFormatProperty("prettyPrint", "true");

rest("/index")
    .description("Tea Index REST API")
    
    .get()
        .description("Get all teas or filter by strength")
        .param()
            .name("strength")
            .type(RestParamType.query)
            .required(false)
        .endParam()
        .outType(Tea[].class)
        .to("direct:getIndex")
    
    .post()
        .description("Add new tea to index")
        .type(Tea.class)
        .outType(Response.class)
        .to("direct:postTea");
```

## Data Transformation Mapping

### ESQL to Java/Camel

**IBM ACE ESQL:**
```sql
-- getTeaByStrength.esql
CREATE COMPUTE MODULE getTeaByStrength
    CREATE FUNCTION Main() RETURNS BOOLEAN
    BEGIN
        DECLARE strength CHARACTER;
        SET strength = InputLocalEnvironment.REST.Input.Parameters.strength;
        
        SET OutputRoot.JSON.Data.teas[] = 
            SELECT NAME, STRENGTH, CAFFEINATED 
            FROM Database.TEA_INDEX 
            WHERE STRENGTH = strength;
        
        RETURN TRUE;
    END;
END MODULE;
```

**Apache Camel with JPA:**
```java
// Route
from("direct:getTeaByStrength")
    .to("bean:teaService?method=findByStrength")
    .marshal().json();

// Service
@Service
public class TeaService {
    @Autowired
    private TeaRepository repository;
    
    public List<Tea> findByStrength(String strength) {
        return repository.findByStrength(strength);
    }
}

// Repository (JPA)
@Repository
public interface TeaRepository extends JpaRepository<Tea, String> {
    List<Tea> findByStrength(String strength);
}
```

## Error Handling Mapping

### IBM ACE
```
[HTTPInput] -> [TryCatch]
                 |- Try: [Compute] -> [HTTPReply]
                 |- Catch: [Error Handler] -> [HTTPReply with error]
```

### Apache Camel
```java
onException(Exception.class)
    .handled(true)
    .setHeader(Exchange.HTTP_RESPONSE_CODE, constant(500))
    .setBody(simple("{\"error\": \"${exception.message}\"}"))
    .marshal().json();

from("direct:processRequest")
    .doTry()
        .to("bean:teaService")
        .marshal().json()
    .doCatch(ValidationException.class)
        .setHeader(Exchange.HTTP_RESPONSE_CODE, constant(400))
        .setBody(simple("{\"error\": \"${exception.message}\"}"))
    .end();
```

## Database Operations Mapping

### IBM ACE Compute Node with ESQL
```sql
-- INSERT operation
INSERT INTO Database.TEA_INDEX (NAME, STRENGTH, CAFFEINATED)
VALUES (InputRoot.JSON.Data.name, 
        InputRoot.JSON.Data.strength, 
        InputRoot.JSON.Data.caffeinated);
```

### Apache Camel with JPA
```java
// Option 1: Using JPA Repository
@Service
public class TeaService {
    @Autowired
    private TeaRepository repository;
    
    @Transactional
    public Tea createTea(Tea tea) {
        if (repository.existsById(tea.getName())) {
            throw new DuplicateTeaException("Tea already exists");
        }
        return repository.save(tea);
    }
}

// Option 2: Using SQL Component
from("direct:insertTea")
    .to("sql:INSERT INTO TEA_INDEX (NAME, STRENGTH, CAFFEINATED) "
        + "VALUES (:#${body.name}, :#${body.strength}, :#${body.caffeinated})");
```

## Enterprise Integration Patterns

| Pattern | IBM ACE | Apache Camel |
|---------|---------|-------------|
| **Content-Based Router** | Filter/Switch nodes | `choice().when().otherwise()` |
| **Message Filter** | Filter node with condition | `filter(predicate)` |
| **Message Translator** | Compute/Mapping nodes | `process()`, `transform()` |
| **Content Enricher** | Compute with DB lookup | `enrich()`, `pollEnrich()` |
| **Splitter** | FOR loop in ESQL | `split()` |
| **Aggregator** | Aggregate node | `aggregate()` |
| **Request-Reply** | HTTPInput + HTTPReply | REST DSL (automatic) |
| **Wire Tap** | Additional output node | `wireTap()` |
| **Dead Letter Channel** | Error queue | `errorHandler(deadLetterChannel())` |
| **Idempotent Consumer** | Custom logic | `idempotentConsumer()` |

## Technology Stack Comparison

### IBM ACE Stack
```
┌─────────────────────────────┐
│   Integration Server        │
├─────────────────────────────┤
│   Message Flows (ESQL)      │
│   Java Compute Nodes        │
├─────────────────────────────┤
│   ACE Runtime               │
├─────────────────────────────┤
│   Derby Database            │
└─────────────────────────────┘
```

### Apache Camel Stack
```
┌─────────────────────────────┐
│   Spring Boot Application   │
├─────────────────────────────┤
│   Camel Routes (Java DSL)   │
│   Spring Beans/Services     │
├─────────────────────────────┤
│   Apache Camel Runtime      │
│   Spring Framework          │
├─────────────────────────────┤
│   JPA / Hibernate           │
├─────────────────────────────┤
│   Database (Derby/PostgreSQL)│
└─────────────────────────────┘
```

## Migration Code Examples

### Example 1: GET Operation

**Before (IBM ACE):**
```sql
-- getIndex.esql
CREATE FUNCTION Main() RETURNS BOOLEAN
BEGIN
    SET OutputRoot.JSON.Data.teas[] = 
        SELECT * FROM Database.TEA_INDEX;
    RETURN TRUE;
END;
```

**After (Apache Camel):**
```java
// Route
from("direct:getIndex")
    .routeId("getTeaIndex")
    .log("Fetching all teas from database")
    .to("jpa:com.example.model.Tea?query=SELECT t FROM Tea t")
    .marshal().json(JsonLibrary.Jackson)
    .log("Returned ${body.size} teas");

// Or with Repository
from("direct:getIndex")
    .bean(teaRepository, "findAll")
    .marshal().json();
```

### Example 2: POST with Validation

**Before (IBM ACE):**
```sql
CREATE FUNCTION Main() RETURNS BOOLEAN
BEGIN
    DECLARE count INTEGER;
    DECLARE name CHARACTER InputRoot.JSON.Data.name;
    
    SET count = SELECT COUNT(*) FROM Database.TEA_INDEX 
                WHERE NAME = name;
    
    IF count = 0 THEN
        INSERT INTO Database.TEA_INDEX VALUES (...);
        SET OutputRoot.JSON.Data.status = 'success';
    ELSE
        SET OutputRoot.JSON.Data.status = 'error';
    END IF;
    
    RETURN TRUE;
END;
```

**After (Apache Camel):**
```java
from("direct:postTea")
    .routeId("createTea")
    .log("Creating new tea: ${body.name}")
    
    // Validation
    .process(exchange -> {
        Tea tea = exchange.getIn().getBody(Tea.class);
        if (tea.getName() == null || tea.getName().isEmpty()) {
            throw new ValidationException("Name is required");
        }
    })
    
    // Service call
    .bean(teaService, "createTea")
    
    // Success response
    .setBody(simple("{\"status\": \"success\", \"name\": \"${body.name}\"}"))
    .setHeader(Exchange.HTTP_RESPONSE_CODE, constant(201));

// Exception handling
onException(DuplicateTeaException.class)
    .handled(true)
    .setHeader(Exchange.HTTP_RESPONSE_CODE, constant(400))
    .setBody(simple("{\"error\": \"Tea already exists\"}"));
```

### Example 3: Legacy Format Filter

**Before (IBM ACE):**
```sql
-- LegacyFormatFilter subflow
IF InputRoot.JSON.Data.format = 'legacy' THEN
    CALL ConvertLegacyTea();
END IF;
```

**After (Apache Camel):**
```java
from("direct:postTea")
    // Content-based routing
    .choice()
        .when(jsonpath("$.format", "legacy"))
            .log("Legacy format detected, converting...")
            .bean(legacyConverter, "convertToModern")
        .otherwise()
            .log("Modern format detected")
    .end()
    
    .to("direct:saveTea");

// Converter bean
@Component
public class LegacyConverter {
    public Tea convertToModern(Map<String, Object> legacy) {
        Tea tea = new Tea();
        tea.setName((String) legacy.get("tea_name"));
        tea.setStrength((String) legacy.get("str"));
        tea.setCaffeinated("Y".equals(legacy.get("caff")));
        return tea;
    }
}
```

## Testing Strategy

### IBM ACE Testing
- Flow testing in Toolkit
- Manual integration testing
- Limited unit testing

### Apache Camel Testing

```java
@SpringBootTest
@CamelSpringBootTest
public class TeaRouteTest {
    
    @Autowired
    private CamelContext camelContext;
    
    @Autowired
    private ProducerTemplate template;
    
    @Test
    public void testGetAllTeas() throws Exception {
        // Given
        List<Tea> result = template.requestBody(
            "direct:getIndex", null, List.class);
        
        // Then
        assertNotNull(result);
        assertTrue(result.size() > 0);
    }
    
    @Test
    public void testCreateTea() throws Exception {
        // Given
        Tea tea = new Tea("Green", "medium", true);
        
        // When
        String result = template.requestBody(
            "direct:postTea", tea, String.class);
        
        // Then
        assertTrue(result.contains("success"));
    }
}

// REST endpoint testing
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
public class TeaApiIntegrationTest {
    
    @LocalServerPort
    private int port;
    
    @Test
    public void testGetTeaEndpoint() {
        given()
            .port(port)
        .when()
            .get("/index")
        .then()
            .statusCode(200)
            .contentType(ContentType.JSON)
            .body("size()", greaterThan(0));
    }
}
```

## Migration Effort by Phase

### Phase 1: Foundation (5-7 days)
- Spring Boot project setup: 1 day
- JPA/Database configuration: 1 day
- Base infrastructure (error handling, logging): 1-2 days
- Security implementation: 2-3 days

### Phase 2: Flow Migration (8-12 days)
- GET operations (5 flows): 3-5 days
- POST operations (2 flows): 3-5 days
- Legacy format handling: 2 days

### Phase 3: Testing (3-5 days)
- Unit tests: 1-2 days
- Integration tests: 1-2 days
- Performance testing: 1 day

### Phase 4: Deployment (2-3 days)
- CI/CD pipeline: 1 day
- Documentation: 1 day
- Production deployment: 1 day

**Total: 18-27 days**

---

**Document Version**: 1.0  
**Last Updated**: 2025  
**Status**: Final