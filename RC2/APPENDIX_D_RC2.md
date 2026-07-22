# RC2 Editorial Revision Notice

**Release:** RC2 (Editorial Review Candidate)

This revision preserves the validated REST API documentation from RC1 while aligning the appendix with the editorial standards adopted throughout the LMWPool documentation suite.

## RC2 objectives

- Harmonize terminology with Volumes 1–4 and Appendices A–F.
- Prepare unified numbering for figures, tables and endpoint listings.
- Prepare explicit cross-references to operational and configuration documentation.
- Improve publication consistency for Release 1.0.

---

# APPENDIX D

# REST API Reference

**Document:** LMWPool Technical Documentation  
**Appendix:** D  
**Title:** REST API Reference  
**Platform:** LMWPool XEC Production Platform  
**Release:** RC1

---

# Document Purpose

This appendix documents the REST interfaces exposed by the LMWPool production platform.

Unlike the previous appendices, which describe infrastructure and configuration, this appendix focuses exclusively on HTTP-based application interfaces.

The primary objectives are:

- API documentation;
- endpoint reference;
- request/response formats;
- authentication model;
- operational considerations;
- integration guidance.

This appendix is intended for developers, system integrators and administrators. Future Release 1.0 will introduce explicit section-level cross references to Volume 3 (Blockchain Notary), Appendix C (Configuration Reference) and Appendix E (Operational Runbooks).

---

# Table of Contents

D.1 API Architecture

D.2 Authentication Model

D.3 Common HTTP Responses

D.4 Blockchain Notary API

D.5 Statistics API

D.6 Status Page Endpoints

D.7 Administrative API

D.8 Error Handling

D.9 API Security

D.10 API Versioning

D.11 Operational Guidelines

D.12 Appendix Summary

---

# D.1 API Architecture

## D.1.1 Overview

The LMWPool platform exposes multiple REST services through the shared Caddy reverse proxy.

Applications never communicate directly with backend services.

All public HTTP requests follow the architecture below.

```text
Client
   │
HTTPS
   │
Caddy Reverse Proxy
   │
REST Services
   │
Application Logic
```

This architecture provides centralized routing, TLS termination and controlled access to backend applications.

---

## D.1.2 API Categories

The platform exposes four logical API groups.

| API | Purpose |
|------|---------|
| Blockchain Notary | Document notarization |
| Statistics API | Mining statistics |
| Status Services | Public information |
| Administrative API | Internal management |

Only the intended public endpoints should be accessible from outside the infrastructure.

---

## D.1.3 Design Principles

All REST services follow common principles.

- HTTPS only.
- Stateless requests.
- JSON payloads.
- Explicit error reporting.
- Independent services.
- Versionable interfaces.

---

# D.2 Authentication Model

## D.2.1 Authentication Philosophy

Authentication requirements depend upon the API being accessed.

Public informational endpoints generally require no authentication.

Operational endpoints require application-level authorization.

---

## D.2.2 Authentication Types

| Method | Typical Usage |
|---------|---------------|
| None | Public statistics |
| User Key | Registered users |
| Internal Authentication | Administrative services |

Authentication requirements should always be evaluated before exposing new endpoints.

---

## D.2.3 HTTPS Requirement

Every public API should be accessed exclusively through HTTPS.

Plain HTTP should only be used internally where explicitly required.

---

# D.3 Common HTTP Responses

## D.3.1 Success Codes

| Code | Meaning |
|------|---------|
| 200 | Request successful |
| 201 | Resource created |
| 202 | Accepted |
| 204 | No content |

---

## D.3.2 Client Errors

| Code | Meaning |
|------|---------|
| 400 | Invalid request |
| 401 | Authentication required |
| 403 | Access denied |
| 404 | Resource not found |
| 409 | Duplicate resource |

---

## D.3.3 Server Errors

| Code | Meaning |
|------|---------|
| 500 | Internal error |
| 502 | Backend unavailable |
| 503 | Service unavailable |

Applications should always return structured JSON error information whenever possible.

---

# D.4 Blockchain Notary API

## D.4.1 Purpose

The Blockchain Notary API provides document notarization services.

Typical operations include:

- document submission;
- blockchain verification;
- certificate generation;
- history retrieval.

---

## D.4.2 Main Endpoints

| Endpoint | Function |
|-----------|----------|
| POST /notarize | Submit document hash |
| GET /verify/{hash} | Verify notarization |
| GET /certificate/{hash}.pdf | Download certificate |
| GET /history | Retrieve user history |

These endpoints constitute the primary public interface of the Blockchain Notary platform.

---

## D.4.3 Typical Request Flow

```text
Client
   │
POST /notarize
   │
Validation
   │
Blockchain
   │
Database
   │
Response
```

The service validates input before interacting with the blockchain.

---

## D.4.4 Request Format

Typical requests contain:

- document hash;
- filename;
- customer identifier;
- email address (where applicable).

Actual payloads depend on the endpoint being invoked.

---

## D.4.5 Responses

Successful responses typically include:

- operation status;
- transaction identifier;
- document hash;
- processing information.

Error responses return descriptive JSON messages.

---

# D.5 Statistics API

## D.5.1 Purpose

The Statistics API provides operational information regarding the mining platform.

Typical information includes:

- miners;
- workers;
- hashrate;
- shares;
- platform statistics.

---

## D.5.2 Typical Endpoints

Examples include:

| Endpoint | Purpose |
|-----------|---------|
| /api/pools | Pool statistics |
| /stats | Operational metrics |
| /miners | Miner information |
| /workers | Worker information |

Endpoint availability depends upon platform configuration.

---

## D.5.3 Response Format

Statistics responses normally use JSON.

Responses should remain lightweight and optimized for dashboard consumption.

---

**End of Appendix D — Part 1**

*(Continues with Status Page Endpoints, Administrative API, Error Handling, API Security, Versioning, Operational Guidelines and Appendix Summary.)*
---

# D.6 Status Page Endpoints

## D.6.1 Purpose

The Status Page represents the public presentation layer of the LMWPool platform.

Unlike the Blockchain Notary API or the Statistics API, the Status Page contains no business logic.

Its responsibilities are limited to:

- presenting mining information;
- displaying operational statistics;
- rendering dashboards;
- consuming backend APIs;
- providing user interaction.

---

## D.6.2 Architectural Role

```text
Browser
    │
HTTPS
    │
Status Page
    │
REST API
    │
Backend Services
```

The browser never communicates directly with the blockchain node or the mining engine.

All backend communication occurs through documented REST interfaces.

---

## D.6.3 Public Resources

Typical public resources include:

| Resource | Purpose |
|----------|---------|
| Dashboard | Pool overview |
| Miner Statistics | Individual miner data |
| Worker Statistics | Worker activity |
| Pool Statistics | Aggregate information |
| Market Information | Cached market data |

The Status Page should remain lightweight and should never perform blockchain operations.

---

## D.6.4 Data Refresh

Operational information is periodically refreshed through API requests.

The refresh interval should balance:

- user responsiveness;
- API load;
- server resources;
- network traffic.

---

# D.7 Administrative API

## D.7.1 Purpose

Administrative APIs provide infrastructure management capabilities that are not intended for public access.

These services support operational maintenance and platform administration.

---

## D.7.2 Administrative Functions

Typical administrative functions include:

- service status;
- maintenance operations;
- configuration verification;
- operational diagnostics;
- infrastructure monitoring.

Administrative interfaces should remain isolated from public endpoints.

---

## D.7.3 Access Policy

Administrative APIs should satisfy the following requirements.

- Authentication required.
- HTTPS only.
- Restricted network exposure.
- Audit logging.
- Principle of least privilege.

---

## D.7.4 Operational Guidelines

Administrative endpoints should:

- remain undocumented for public users;
- expose only necessary functionality;
- return structured error information;
- avoid disclosure of internal implementation details.

---

# D.8 Error Handling

## D.8.1 Purpose

REST services should return predictable and consistent error responses.

Clients must be able to distinguish between:

- invalid requests;
- authentication failures;
- missing resources;
- application failures;
- temporary infrastructure problems.

---

## D.8.2 Error Response Structure

Typical JSON responses should include:

```json
{
    "success": false,
    "error": "...",
    "message": "...",
    "timestamp": "...",
    "request_id": "..."
}
```

Additional diagnostic information should only be returned when appropriate.

---

## D.8.3 Logging

Every unexpected server-side exception should be logged.

Logs should provide sufficient information for troubleshooting while avoiding disclosure of confidential operational details.

---

## D.8.4 Validation

Input validation should occur before any blockchain or database operation.

Typical validation includes:

- required parameters;
- data format;
- string length;
- supported values;
- authentication status.

Invalid requests should terminate immediately with an appropriate HTTP response.

---

# D.9 API Security

## D.9.1 Security Principles

API security follows multiple complementary protection layers.

These include:

- HTTPS encryption;
- authentication;
- reverse proxy filtering;
- application validation;
- backend isolation.

---

## D.9.2 Data Protection

Sensitive information should never be exposed through API responses.

Examples include:

- passwords;
- SMTP credentials;
- private keys;
- internal filesystem paths;
- authentication tokens.

---

## D.9.3 Rate Limiting

Where appropriate, APIs should support request throttling to reduce the impact of:

- accidental overload;
- automated scanning;
- denial-of-service attempts;
- abusive client behavior.

Rate limiting policies should reflect the operational characteristics of each endpoint.

---

# D.10 API Versioning

## D.10.1 Purpose

API versioning allows the platform to evolve while preserving compatibility with existing integrations.

Version identifiers should remain stable throughout the supported lifecycle.

---

## D.10.2 Compatibility

Whenever practical:

- existing endpoints should remain compatible;
- breaking changes should introduce new versions;
- deprecated endpoints should remain available during a defined transition period.

---

## D.10.3 Documentation

Every published endpoint should document:

- purpose;
- request method;
- parameters;
- authentication requirements;
- response structure;
- possible error codes.

---

# D.11 Operational Guidelines

## D.11.1 Monitoring

Operational monitoring should verify:

- API availability;
- response time;
- HTTP error rates;
- authentication failures;
- backend connectivity.

---

## D.11.2 Logging

Application logs should record:

- incoming requests;
- processing errors;
- successful operations;
- unexpected exceptions.

Sensitive request payloads should be excluded from logs whenever possible.

---

## D.11.3 Testing

After deployment, REST services should be verified using representative requests.

Typical verification includes:

- endpoint availability;
- authentication;
- response format;
- integration with backend services;
- expected business functionality.

---

## D.11.4 Operational Maintenance

Routine maintenance activities include:

- log review;
- dependency updates;
- endpoint validation;
- TLS certificate verification;
- documentation synchronization.

The documentation should always reflect the currently deployed production APIs.

---

# D.12 Appendix Summary

This appendix documents the REST interfaces supporting the LMWPool production platform.

The documented APIs provide controlled access to:

- Blockchain Notary services;
- mining statistics;
- public dashboards;
- administrative functionality.

Together with the infrastructure, configuration and operational appendices, this document completes the API reference for the platform.

Future revisions should extend this appendix whenever new production endpoints are introduced while preserving backward compatibility and maintaining consistent documentation standards.

---

# End of Appendix D

**LMWPool Technical Documentation**

**Appendix D — REST API Reference**

**Release:** RC1
