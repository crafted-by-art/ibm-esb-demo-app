# IBM ACE Tea REST API - Migration Discovery Document

**Project Name:** IBM ACE Tea REST API
**Target Platform:** Apache Camel
**Document Version:** 1.0
**Date:** December 2024
**Author:** Migration Analysis Team

**Confluence Documentation:** [https://kb.epam.com/display/EPMAIMMJAV/IBM+ACE+Tea+REST+API+Migration+Discovery](https://kb.epam.com/display/EPMAIMMJAV/IBM+ACE+Tea+REST+API+Migration+Discovery)

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Project Overview](#project-overview)
3. [Architecture and Components](#architecture-and-components)
4. [IBM ACE to Apache Camel Component Mapping](#component-mapping)
5. [Flow Inventory and Catalog](#flow-inventory)
6. [Flow Analysis - GET Tea by ID](#flow-get-tea)
7. [Flow Analysis - POST Create Tea](#flow-post-tea)
8. [Migration Roadmap and Next Steps](#migration-roadmap)

---

<div style="page-break-after: always;"></div>

## 1. Executive Summary

### 🎯 Project Overview

This document provides a comprehensive analysis of the **IBM App Connect Enterprise (ACE) Tea REST API** project and a detailed plan for migrating it to **Apache Camel**. The project is a simple CRUD-based REST API that manages tea records in a database.

### 👌 Key Highlights

- ✅ **Simple Architecture**: 2 REST endpoints (GET and POST)
- ✅ **Low Complexity**: Minimal dependencies (DB2 JDBC, JSON)� ����**No External Services**: No message queues, external APIs, or SOAP web services
- ✅ **Standard Patterns**: REST request/response, database CRUD, JSON transformation
- €️ **Fast Migration**: Estimated 12-20 days (1 developer) or 8-12 days (small team)

### ⚠️ Migration Complexity

| Dimension | Assessment | Notes |
|-----------|--------------|-------|
| **Overall** | 🟢 **LOW to MEDIUM** | Straightforward migration with minimal risks |
| **REST API** | 🟢 **LOW** | Direct mapping to Camel REST DSL |
| **Database** | 🟢 **LOW** | Standard JDBC, easy migration |
| **ESQL Logic** | 🟡 **MEDIUM** | Requires Java/Groovy conversion |
| **Schema Mapping** | 🟡 **MEDIUM** | Needs reimplementation |
| **Dependencies** | 🟒 **LOW** | Minimal external dependencies |

### 🚈 ⊙ ⚠️ Critical Issues Identified

1. ⚠️ **SQL Injection Vulnerability**
   - Uses string concatenation in ESQL
   - **Recommendation**: Use parameterized queries with PreparedStatements

2. ⚠️ **Race Condition in ID Generation**
   - Multiple requests can retrieve the same MAX ID
   - **Recommendation**: Use database auto-increment or unique sequence generator

3. ⚠️ **No Security**
   - No authentication/authorization
   - **Recommendation**: Implement OAuth2 or JWT based authentication

4. ⚠️ **No Transaction Management**
   - No explicit database transactions
   - **Recommendation**: Implement transactional boundaries

### 🎂 Recommended Migration Approach

We recommend a **5-phased migration** approach:

1. 🏯 **Preparation** (2-3 days)
   - Setup development environment
   - Analyze dependencies and test scenarios

2. 📊 Migrate **GET endpoint** (3-4 days)
   - Create Camel REST API
   - Implement JDBC data retrieval
   - Convert ESQ JSON transformation to Java

3. 📍 **Migrate POST endpoint** (5-8 days)
   - Implement schema transformation logic
   - Convert ESQ logic to Java
   - Implement JDBC insert operations

4. 🐒 Sucurity & Refinements (2-3 days)
   - Add authentication/authorization
   - Implement transaction management
   - Fix security vulnerabilities

5. ✅ **Testing & Deployment** (2-3 days)
   - Unit testing
   - Integration testing
   - Performance testing
   - Poo duction deployment

### 🚀 Strategic Benefits

- 🚀 **Modernization**: Move to open-source, cloud-native platform
- ���� **Cost Reduction**: Eliminate IBM ACE licensing costs
- 🚀 **Scalability**: Better horizontal scaling with Kubernetes
- ✅ **Maintainability**: Simpler, more maintainable Java/Groovy code
- 🔥 **Vendor Independence**: No lock-in to IBM ecosystem

---

<div style="page-break-after: always;"></div>

## 2. Project Overview

### 🚀 Business Purpose

The **Tea REST API** is a simple demonstration project designed to showcase IBM ACE's capabilities for building RESTful web services. It provides:

- ☕ Retrieve tea details by ID (GET)/Tea/{id})
- ☕ Create new tea records (POST /Tea)

### 🏗 Technical Stack
)
**Current (IBM ACE)**:

- **Runtime**: IBM App Connect Enterprise (11.0 or later)
- **Database**: DB2 (with JDBC connectivity)
- **Data Format**: JSON
- **Protocol**: HTTP/REST
- **Logic**: ESQL, Java, Message Maps

**Target (Apache Camel)**:

- **Runtime**: Apache Camel 4.x (with Spring Boot 3.x)
- **Database**: DB2 (unchanged)
- **Data Format**: JSON
- **Protocol**: HTTP/REST
- **Logic**: Java beans, Camel processors, Jackson

### 📊  Key Features

1. **RESTful API**: Two endpoints for tea management
2. **Database Integration**: Direct JDBC connection to DB2
3. **JSON Transformation**: Dynamic message transformation for schema compatibility
4. **Modular Design**: Reusable subflows and shared libraries

### 📊 Project Metrics

| Metric | Count |
|-------------------------|------|
| REST Endpoints | 2 |
| Message Flows | 2 |
| Application Subflows | 2 |
| Shared Library Subflows | 3 |
| Java Compute Nodes | 2 |
| ESQL Modules | 3 |
| Message Maps | 1 |
| External Dependencies | 2 |
| Integration Patterns | 8 |

---

<div style="page-break-after: always;"></div>

## 3. Architecture and Components

### 🏗 System Architecture

```plantuml
@startuml

title IBM ACE Tea REST API - System Architecture

package "Client Layer" {
  actor User as user
}

package "IBM ACE Runtime" {
  [Tea REST API] as restapi <<TeaRestApp Application>>
  
  package "Application Flows" {
    [GetTeaById] as getflow
    [PostTea] as postflow
    
    package "Application Subflows" {
      [GetTeaById_Sub] as getsub
      [PostTea_Sub] as postsub
    }
  }
  
  package "Shared Library" {
    [Conmon] as common <<TeaCommonLibrary>>
    
    package "Shared Subflows" {
      [DatabaseConnector] as dbconn
      [ErrorHandler] as err handler
      [Logger] as logger
    }
  }
}

database "DB2 Database" {
  [TEA Table] as teatable
}

cloud "File System" {
  [DB2 JDBC Driver] as jdbcdriver
}

user --> restapi: HTTP REQUEST\nrestapi --> getflow: GET /Tea/{id}
restapi --> postflow: POST /Tea

getflow --> getsub: call subflow
postflow --> postsub: call subflow

getsub --> dbconn: use shared connector
postsub --> dbconn: use shared connector

getsub --> logger: logging
postsub --> logger: logging

dbconn --> jtbstable: JDBC QUERY
dbconn ..> jdbcdriver: uses DB2 driver

getflow .> errhandler: error handling
postflow .> errhandler: error handling

restapi --> user: HTTP RESPONSE

@enduml
```

### 📦 External Dependencies

| Name | Type | Description | Usage |
|---------|--------|-------------|------|
| **DB2 DBMSPA O	DBC** | Database Driver | IBM DB2 JDBC driver for database connectivity | Database connections, SQL query execution, data retrieval/insert |
| **DB2 Database** | Database | Relational database storing tea records | Persistent storage for tea data (tea_id, tea_name, tea_type) |
| **File System** | Storage | Local file system for JDBC drivers and configuration files | Stores JDBC drivers and ACE configuration files |

### 📡 Integration Protocols

| Name | Type | Description | Usage |
|---------|--------|-------------|------|
| **HTTP/REST** | Application Protocol | RESTful HTTP API for tea management | Exposes GET and POST endpoints for client interaction |
| **JDBC** | Database Protocol | Java Database Connectivity for DB2 access | Executes SELECT and INSERT SQL statements against DB2 database |
| **JSON** | Data Format | Lightweight data interchange format | Request/Response payload format for REST API |

### 💐 Security Components

| Name | Type | Description | Usage |
|----------|---------|-------------|------|
| **NONE** | N/A | ⚠️ No security implemented in current project | N/A |

#