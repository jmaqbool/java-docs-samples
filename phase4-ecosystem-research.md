# phax/phase4 Ecosystem Research

## What is phase4?

**phase4** ([github.com/phax/phase4](https://github.com/phax/phase4)) is an embeddable, lightweight Java library for sending and receiving **AS4 messages** across multiple industry profiles. It acts as both an AS4 client and server and is designed to integrate into existing Java systems (servlets, Spring Boot, standalone apps). It is authored by Philip Helger ([@phax](https://github.com/phax)) and published under the Apache 2.0 license.

**AS4** is the messaging protocol used in:
- **Peppol** (pan-European e-procurement / e-invoicing network)
- **CEF eDelivery** (EU Connecting Europe Facility)
- **BDEW** (German energy sector)
- **ENTSOG** (European gas network operators)
- **EESPA** (e-invoicing service providers)
- **BPC** (Business Payments Coalition)
- **DBNAlliance** (North American e-invoicing)
- **EUCTP** (EU Customs)
- **eDelivery2 / eSENS** (EC eDelivery v2 with ECDSA profiles)

---

## Ecosystem Architecture

The phax/phase4 library sits within a broader **helger/phax Java library stack**:

```
ph-commons (core utilities)
  └─ ph-security, ph-xml, ph-jaxb, ph-web, ph-dao, ph-settings ...
       └─ ph-xmldsig, ph-xsds (XML Digital Signature, XML Schema data models)
            └─ peppol-commons (Peppol identifiers, SMP/SML client, SBDH)
                 └─ phase4-lib (core AS4 engine)
                      ├─ phase4-profile-* (per-standard profile modules)
                      ├─ phase4-peppol-client / phase4-peppol-servlet
                      ├─ phase4-server-webapp (reference servlet)
                      └─ phase4-edelivery2-client (eDelivery2 / ECDSA)
```

---

## Core Modules

| Module | Maven Artifact | Purpose |
|--------|---------------|---------|
| `phase4-lib` | `com.helger.phase4:phase4-lib` | Core AS4 engine (message building, signing, encryption, SPI hooks). Zero Peppol dependency. |
| `phase4-test` | `com.helger.phase4:phase4-test` | Test utilities and helpers |
| `phase4-server-webapp` | `com.helger.phase4:phase4-server-webapp` | Reference servlet-based server webapp |

---

## Profile Modules

Each profile module bundles the AS4 PMode configuration and validation rules for a specific standard:

| Module | Standard | Notes |
|--------|----------|-------|
| `phase4-profile-peppol` | OpenPeppol | Peppol AS4 profile v2.x |
| `phase4-profile-cef` | CEF eDelivery | EU Connecting Europe Facility |
| `phase4-profile-bdew` | BDEW | German energy (UTILMD, MSCONS, etc.) |
| `phase4-profile-entsog` | ENTSOG | European gas network |
| `phase4-profile-eespa` | EESPA | E-invoicing service providers |
| `phase4-profile-bpc` | BPC | Business Payments Coalition |
| `phase4-profile-dbnalliance` | DBNAlliance | North American e-invoicing |
| `phase4-profile-euctp` | EUCTP | EU Customs Trading Portal |
| `phase4-edelivery2-client` | eDelivery2 (ECDSA) | v4.5.0+; new ECDSA/ECDH-ES two-corner/four-corner profiles via `Phase4EDelivery2Sender` |

---

## Peppol-Specific Modules

| Module | Purpose |
|--------|---------|
| `phase4-profile-peppol` | PMode definitions, validation, Peppol certificate handling |
| `phase4-peppol-client` | High-level `Phase4PeppolSender` for sending UBL/CII documents |
| `phase4-peppol-servlet` | `Phase4PeppolServletMessageProcessorSPI` for receiving in a servlet container |

**`Phase4PeppolSender`** provides a fluent builder API:
```java
Phase4PeppolSender.builder()
    .documentTypeID(...)
    .processID(...)
    .senderParticipantID(...)
    .receiverParticipantID(...)
    .fromPartyID(...)
    .payload(aUBLDoc)
    .keyStoreProvider(keyStore)
    .sendingDateTimeOrNow()
    .sendMessage();
```

---

## Key Upstream Dependencies (helger stack)

| Library | Group ID | Role |
|---------|----------|------|
| `ph-commons` | `com.helger` | Core utilities (collections, I/O, threading) |
| `ph-security` | `com.helger` | Crypto, keystore, certificate handling |
| `ph-xml` | `com.helger` | XML utilities |
| `ph-jaxb` | `com.helger` | JAXB marshalling |
| `ph-web` | `com.helger` | HTTP client/server, servlet helpers |
| `ph-dao` | `com.helger` | Data access objects, XML persistence |
| `ph-settings` | `com.helger` | Configuration management |
| `ph-xmldsig` | `com.helger` | XML Digital Signature (WS-Security) |
| `peppol-commons` | `com.helger.peppol` | Peppol identifiers, SMP/SML client, SBDH, MLR |

**Version alignment as of phase4 v4.5.0:** requires `ph-commons >= 12.2.5` and `peppol-commons >= 12.4.0`.

---

## Standalone Example

[phax/phase4-peppol-standalone](https://github.com/phax/phase4-peppol-standalone) is the reference Spring Boot 3.x application demonstrating a fully functional Peppol AS4 access point:

- Java 17+, Spring Boot 3.x, embedded Tomcat on port 8080
- Sends and receives Peppol AS4 messages
- Performs SMP lookups to resolve endpoints dynamically
- Run: `java -jar target/phase4-peppol-standalone-x.y.z.jar`

---

## 2026 Breaking Changes: Peppol SML Insourcing

OpenPeppol is insourcing its SML (Service Metadata Locator) DNS infrastructure away from the EU DIGIT service. Key migration requirements:

- **From:** `DIGIT` ESML enum values (`edelivery.tech.ec.europa.eu`)
- **To:** OpenPeppol ESML enum values (new `peppol.org` domains)
- **Deadline:** August 31, 2026
- **Required versions:** `peppol-commons >= 12.3.9`, `phase4 >= 4.4.1`
- The `ESML` enumeration in peppol-commons was extended with new `PEPPOL_*` predefined values for test and production environments.

---

## Maven Coordinates (current)

```xml
<!-- Core AS4 library -->
<dependency>
    <groupId>com.helger.phase4</groupId>
    <artifactId>phase4-lib</artifactId>
    <version>4.5.x</version>
</dependency>

<!-- Peppol sending client -->
<dependency>
    <groupId>com.helger.phase4</groupId>
    <artifactId>phase4-peppol-client</artifactId>
    <version>4.5.x</version>
</dependency>

<!-- Peppol receiving servlet -->
<dependency>
    <groupId>com.helger.phase4</groupId>
    <artifactId>phase4-peppol-servlet</artifactId>
    <version>4.5.x</version>
</dependency>
```

Maven Central group: [`com.helger.phase4`](https://mvnrepository.com/artifact/com.helger.phase4)

---

## Known Users

The phase4 Wiki lists known commercial and open-source users including ERP vendors, access point providers, and national e-invoicing platforms across Europe and North America.

---

## Key Links

- Main repo: [github.com/phax/phase4](https://github.com/phax/phase4)
- Wiki: [github.com/phax/phase4/wiki](https://github.com/phax/phase4/wiki)
- Peppol overview: [github.com/phax/peppol](https://github.com/phax/peppol)
- peppol-commons: [github.com/phax/peppol-commons](https://github.com/phax/peppol-commons)
- Standalone example: [github.com/phax/phase4-peppol-standalone](https://github.com/phax/phase4-peppol-standalone)
- Peppol news/status: [peppol.helger.com](https://peppol.helger.com/public)
- Maven Central: [mvnrepository.com/artifact/com.helger.phase4](https://mvnrepository.com/artifact/com.helger.phase4)
- SML insourcing 2026 discussion: [github.com/phax/phase4/discussions/357](https://github.com/phax/phase4/discussions/357)
