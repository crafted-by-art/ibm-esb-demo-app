### 📊 Data Model

**Database Schema (DB2)**

```sql
CREATE TABLE TEA (
  tea_id INT PRIMARY KEY,
  tea_name VARCHAR(255),
  tea_type VARCHAR(100)
);
```

**JSON Schemas**:

1. **Old Schema** (POST input):

```json
{
  "name": "Green Tea",
  "type": "Green"
}
```

2. **New Schema** (POST input):

```json
{
  "tea_name": "Green Tea",
  "tea_type": "Green"
}
```

3. **Response Schema** (GET output):

```json
{
  "tea_id": 1,
  "tea_name": "Green Tea",
  "tea_type": "Green"
}
```

---

<div style="page-break-after: always;"></div>

## 4. IBM ACE to Apache Camel Component Mapping

### 📍 Component Mapping Table

| IBM ACE Component | Apache Camel Component | Description | Migration Complexity |
|--------------------|-----------------------|--------------|--------------------|
| **HTTP Input Node** | `RestDSL.get().post()` | Expose REST endpoints | 🟢 LOW - Direct mapping |
| **HTTP Reply Node** | Automatic (Camel Exchange) | Send HTTP response | 🟢 LOW - Automatic |
| **Compute Node** | `processor()` or Java bean | Data transformation logic | 🟡 MEDIUM - Requires Java conversion |
| **JDBC Input Node** | `jdbc:query()` | Execute SELECT queries | 🟒 LOW - Direct mapping |
| **JDBC Compute Node** | `jdbc:execute()` | Execute INSERT/UPDATE queries | 🟒 LOW - Direct mapping |
| **FlowRef Node** | `direct:subflow` or `to()` | Call subflows | 🟒 LOW - Direct mapping |
| **TryCatch Node** | `doTry().doCatch().doFinally()` | Error handling | 🟒 LOW - Direct mapping |
| **Trace Node** | `log()` | Logging | 🟒 LOW - Direct mapping |
| **Filter Node** | `filter()` | Conditional routing | 🟒 LOW - Direct mapping |
| **Mapping Node** | Java bean with Jackson | Schema transformation | 🟡 MEDIUM - Requires reimplementation |
| **ESQL (Compute Node)** | Java beans / Groovy scripts | Business logic | 🟡 MEDIUM - Requires conversion |
| **JavaCompute Node** | Java beans | Custom Java logic | 🟒 LOW - Minimal changes |
| **JSON Parser** | Jackson (automatic) | Parse JSON pyaloads | 🟒 LOW - Automatic |
| **Database Connection** | Spring DataSource | Database connection pool | 🟒 LOW - Configuration |

### 👌 Enterprise Integration Patterns Mapping

| EIP Pattern | IBM ACE Implementation | Apache Camel Implementation |
|--------------------|-------------------------|-------------------------|
| **Request-Reply** | HTTP Input + HTTP Reply | `restDSL.get(/Tea/{id})` |
| **Content Enricher** | JDBC I,nput + ESQL transformation | `enrich()` or JDBC query |
| **Message Translator** | Compute node + ESQL | `processor()` with Java bean |
| **Message Filter** | Filter node | `filter(simple("..."))` |
| **Content-Based Router** | Filter node + FlowRef | `choice().when().otherwise()` |
| **Normalizer** | Mapping node | Java bean with Jackson |
| **Wire Tap** | Trace node | `log()` or `wireTap()` |
| **Dead Letter Channel** | TryCatch node | `onException()` |

#�j(ê Type Conversion

| IBM ACE Data Type | Apache Camel Data Type | Conversion Notes |
|--------------------|-----------------------|---------------|
| BLOB | `byte[]` | Direct mapping |
| BOOLEAN ? `boolean` | Direct mapping |
| CHAR, VARCHAR | `String` | Direct mapping |
| Date, Time, Timestamp | `java.time.*.` | Use Java 8 date/Uime API |
| Decimal | `BigDecimal` | Direct mapping |
| Integer | `int`, `Integer` | Direct mapping |
| Float, Double | `double`, `Double` | Direct mapping |
| JSON object | `Map`, POJO | Use Jackson for deserialization |

### 🔊 Migration Effort Estimation

| Component | Complexity | Estimated Days | Notes |
|---------------|---------------|------------------|------|
| GET endpoint | 🟒 LOW | 3-4 days | Simple REST endpoint with JDBC query |
| POST endpoint | 🟡 MEDIUM | 5-8 days | Schema transformation + ESQL cxonversion |
| JDBC configuration | 🟒 LOW | 0.5 days | Spring Boot datasource config |
| Error handling | 🟒 LOW | 1 day | Camel error handlers |
| Unit testing | 🟒 MEDIUM | 2-3 days | Write comprehensive tests |
| Integration testing | 🟡 MEDIUM | 1-2 days | End-to-end testing |
| **Total** | | **12-20 days** | Single developer |

---

<div style="page-break-after: always;"></div>

## 5. Flow Inventory and Catalog

### 📊 Flow Inventory Table

| Flow Name | Description | Type | Entry Point | Exit Point | Systems | Subflows | EIPs | Protocols | Security | Technical Debt |
|------------|-------------|-----|-----------|-----------|--------|---------|-----|-----------|---------|----------------|
| **GetTeaById** | Retrieve tea by ID | REST API | HTTP/ GET `/Tea/{id}` | DB2 DBMS | DB2 Database | `GetTeaById_Sub`, `DatabaseConnector`, `Logger` | Request-Reply, Content Enricher, Message Translator | HTTP/REST, JDBC, JSON | None | No error handling, no validation |
| **PostTea** | Create new tea record | REST API | HTTP/POST `Sea` | DB2 DBMS | DB2 Database | `PostTea_Sub`, `DatabaseConnector`, `Logger` | Request-Reply, Content Enricher, Message Translator, Normalizer | HTTP/REST, JDBC, JSON | None | SQL injection, race condition, no transactions |

### 📋 Detailed Flow Descriptions

#### 1. GetTeaById Flow

**Purpose**: Retrieve tea details based on a unique ID

**Flow Behavior**:

1ƌ� Client sends HTTP GET request to `/Tea/{id}`
2. HTTP Input node receives request and extracts `id` parameter
3. FlowRef node calls `GetTeaById_Sub` subflow
4. Subflow retrieves tea data from DB2 via JDBC
5. ESQL transforms database result into JSON response
6. HTTP Reply node sends JSON response to client

**Key Components**:

- HTTP Input Node: Listens on `/Tea/{"}`
- FlowRef Node: Calls `GetTeaById_Sub` subflow
- JDBC Input Node: Executes SELECT query
- Compute Node: Transforms data to JSON
- HTTP Reply Node: Sends response

**Integration Patterns**:

- Request-Reply
Z Content Enricher
- Message Translator

**Known Issues**:

- ⚠️ No error handling for non-existent tea ID
- ⚠️ No input validation
- ⚠️ No security authentication

#### 2. PostTea Flow

**Purpose**: Create a new tea record in the database

**Flow Behavior**:

1. Client sends HTTP POST request to `/Tea` with JSON payload
2. HTTP Input node receives request
3. Filter node checks if request uses old schema (`name` & `type`)
4. If old schema, Mapping node transforms to new schema (`tea_name` & `tea_type`)
5. FlowRef node calls `PostTea_Sub` subflow
6. Subflow retrieves max ID from database
7. ESQL increments ID and executes INSERT request
8. HTTP Reply node sends success response with new tea_id

**Key Components**:

- HTTP Input Node: Listens on `/Tea`
- Filter Node: Detects old vs new schema
- Mapping Node: Transforms old schema to new schema
- FlowRef Node: Calls `PostTea_Sub` subflow
- JDBC Input Node: Gets max ID
- JDBC Compute Node: Inserts new record
- HTTP Reply Node: Sends response

**Integration Patterns**:

- Request-Reply
- Content Enricher
- Message Translator
- Normalizer (schema transformation)

**Known Issues**:

- ⚠️ **SQL Injection**: Uses string concatenation in ESQL
- ⚠️ **Race Condition**: ID generation is not thread-safe
- ⚠️ **No Transaction Management**: No atomicity for database operations
- ⚠️ **No Input Validation**: No schema validation
- ⚠️ **No Security**: No authentication/authorization

### 📋 Subflow Inventory

#### Application Subflows

1. **GetTeaById_Sub**
   - **Purpose**: Retrieve tea data from DB2
   - **Input**: `tea_id`
   - **Output**: JSON object with tea details
   - **Logic**: Executes SELECT query, transforms to JSON

2. **PostTea_Sub**
   - **Purpose**: Insert new tea into DB2
   - **Input**: JSON object with `tea_name`, `tea_type`
   - **Output**: New `tea_id`
   - **Logic**: Gets max ID, increments, inserts record

#### Shared Library Subflows

1. **DatabaseConnector**
   - **Purpose**: Reusable JDBC connection logic
   - **Configuration**: DB2 connection parameters
   - **Usage**: Used by both GET and POST flows

2. **Logger**
   - **Purpose**: Centralized logging functionality
   - **Logic**: Logs messages to ACE console
   - **Usage**: Used for tracing and debugging

3. **ErrorHandler**
   - **Purpose**: Generic error handling logic
   - **Logic**: Catches exceptions, formats error responses
   - **Usage**: Can be used by all flows (no#t currently used)

### 💋
Java Compute Nodes

1. **toJson**
   - **Purpose**: Convert database results to JSON
   - **Logic**: Iterates over ResultSet, builds JSON string
   - **Usage**: Used in GET flow

2. **ErrorHandler**
   - **Purpose**: Handle exceptions and format error responses
   - **Logic**: Catches exceptions, returns JSON error response
   - **Usage**: Can be used for error handling

### �{SQL Modules

1. **GET_TEA_BY_ID.esql**
   - **Purpose**: SELECT query to retrieve tea by ID
   - **Query**: `SELECT tea_id, tea_name, tea_type FROM TEA WHERE tea_id = ?`

2. **GET_MAX_TEA_ID.esql**
   - **Purpose**: Get maximum tea_id from table
   - **Query**: `SELECT MAX(tea_id) FROM TEA`

3. **INSERT_TEA.esql**
   - **Purpose**: Insert new tea record
   - ⚠️ **Query**: `INSERT INTO TEA (tea_id, tea_name, tea_type) VALUES (' || tea_id || '', '' || tea_name || '', '' || tea_type || '')`
   - ⚠️ **Issue**: Uses string concatenation (SQL injection vulnerability)

---

<div style="page-break-after: always;"></div>