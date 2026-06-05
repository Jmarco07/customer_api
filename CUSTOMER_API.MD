# Architecture Diagrams

## 1. Current Architecture — Transaction History API

```mermaid
flowchart TB
    subgraph Clients["Client Applications"]
        APP1["Transaction History\nConsumer App"]
    end

    subgraph AWS["AWS Cloud (ap-southeast-1)"]
        subgraph APIGW["Amazon API Gateway v2 (HTTP API)"]
            ROUTE1["POST /v1/api/generate_token"]
            ROUTE2["POST /v1/api/transactions"]
            ROUTE3["GET /v1/api/transactions/{retRefNo}"]
            ROUTE4["POST /v1/api/wallet-transaction-history"]
            ROUTE5["POST /v1/api/remittance-transaction-history"]
            ROUTE6["POST /v1/api/qr-emvco-transaction-history"]
            ROUTE7["POST /v1/api/ibank-fund-transfer-history"]
            ROUTE8["POST /v1/api/corporate-transaction-history"]
            ROUTE9["POST /v1/api/pesonet-transaction-history"]
            ROUTE10["POST /v1/api/bulk-pepp-disbursement-history"]
        end

        subgraph AUTH["Authentication Layer"]
            AUTHORIZER["Lambda Authorizer\n(cauth_token)"]
            DDB["DynamoDB\n(cauth_tokens)"]
        end

        subgraph VPC["VPC (Private Subnets)"]
            subgraph LAMBDA["Lambda Functions (Python 3.11)"]
                GEN_TOKEN["generate_cauth_token"]
                TXN["transaction_history_logs"]
                TXN_VIEW["transaction_view"]
                WALLET["wallet_transaction"]
                REMIT["remittance_transaction"]
                QR["qr_emvco_transaction"]
                IBANK["ibank_ft_transaction"]
                CORP["corporate_transaction"]
                PESO["pesonet_transaction"]
                BULK["bulk_pepp_disbursement"]
            end

            subgraph DATA["Data Layer"]
                SM["Secrets Manager\n(Redshift Credentials)"]
                RS["Amazon Redshift\nServerless"]
            end

            VPCE_SM["VPC Endpoint\n(Secrets Manager)"]
            VPCE_RS["VPC Endpoint\n(Redshift Data)"]
        end

        CW["CloudWatch Logs"]
        LAYER["Lambda Layer\n(psycopg2, pydantic, PyJWT)"]
    end

    APP1 -->|"HTTPS Request"| APIGW
    APIGW -->|"Authorization\nCheck"| AUTHORIZER
    AUTHORIZER -->|"Lookup Token"| DDB
    AUTHORIZER -->|"Allow/Deny"| APIGW

    ROUTE1 --> GEN_TOKEN
    GEN_TOKEN -->|"Store Token"| DDB

    ROUTE2 --> TXN
    ROUTE3 --> TXN_VIEW
    ROUTE4 --> WALLET
    ROUTE5 --> REMIT
    ROUTE6 --> QR
    ROUTE7 --> IBANK
    ROUTE8 --> CORP
    ROUTE9 --> PESO
    ROUTE10 --> BULK

    TXN --> VPCE_SM --> SM
    TXN --> VPCE_RS --> RS
    WALLET --> VPCE_SM
    WALLET --> VPCE_RS
    REMIT --> VPCE_SM
    REMIT --> VPCE_RS

    LAMBDA --> CW
    LAMBDA -.->|"Shared Dependencies"| LAYER
```

---

## 2. Current Architecture — Simplified View (for Presentation)

```mermaid
flowchart LR
    CLIENT["Client App"] -->|"HTTPS + Bearer Token"| APIGW["API Gateway\n(HTTP API)"]
    APIGW -->|"Validate JWT"| AUTH["Lambda Authorizer"]
    AUTH -->|"Lookup"| DDB["DynamoDB\n(Tokens)"]
    AUTH -->|"Allow/Deny"| APIGW
    APIGW -->|"Route"| LAMBDA["Lambda Handler\n(Python 3.11)"]
    LAMBDA -->|"Get Credentials"| SM["Secrets Manager"]
    LAMBDA -->|"Query"| RS["Redshift\nServerless"]
    LAMBDA -->|"Response\n(GMT+8 ISO 8601)"| CLIENT

    style APIGW fill:#FF9900,color:#fff
    style AUTH fill:#FF9900,color:#fff
    style LAMBDA fill:#FF9900,color:#fff
    style DDB fill:#3B48CC,color:#fff
    style SM fill:#DD344C,color:#fff
    style RS fill:#3B48CC,color:#fff
```

---

## 3. Problem Statement — Current State (Multiple Apps Direct Access)

```mermaid
flowchart TB
    subgraph Apps["Applications"]
        A1["Transaction History App"]
        A2["Customer Info App"]
        A3["Reports App"]
        A4["Mobile App"]
        A5["Partner Portal"]
    end

    subgraph Problems["⚠️ Problems"]
        P1["❌ No centralized auth"]
        P2["❌ Direct DB access"]
        P3["❌ No standardized responses"]
        P4["❌ Hard to audit/monitor"]
        P5["❌ DB schema changes break apps"]
    end

    RS["Amazon Redshift\nServerless"]

    A1 -->|"✅ Via API Gateway\n(already secured)"| RS
    A2 -->|"❌ Direct Connection"| RS
    A3 -->|"❌ Direct Connection"| RS
    A4 -->|"❌ Direct Connection"| RS
    A5 -->|"❌ Direct Connection"| RS

    Problems -.-> RS

    style A1 fill:#2E8B57,color:#fff
    style A2 fill:#DC143C,color:#fff
    style A3 fill:#DC143C,color:#fff
    style A4 fill:#DC143C,color:#fff
    style A5 fill:#DC143C,color:#fff
    style RS fill:#3B48CC,color:#fff
```

---

## 4. Proposed Architecture — Centralized Customer API

```mermaid
flowchart TB
    subgraph Clients["Client Applications"]
        A1["Transaction History App"]
        A2["Customer Info App"]
        A3["Reports App"]
        A4["Mobile App"]
        A5["Partner Portal"]
    end

    subgraph AWS["AWS Cloud (ap-southeast-1)"]
        subgraph GATEWAY["Centralized Customer API Gateway (HTTP API)"]
            direction TB
            STAGE["Stage: /v1"]
        end

        subgraph AUTH["Shared Authentication Layer"]
            AUTHORIZER["Lambda Authorizer\n(JWT Validation)"]
            DDB["DynamoDB\n(Tokens + Roles/Scopes)"]
        end

        subgraph DOMAINS["Domain-Based Lambda Handlers"]
            subgraph TXN_DOMAIN["Transaction History Domain"]
                TXN1["GET /transactions"]
                TXN2["GET /transactions/{id}"]
                TXN3["POST /wallet-transactions"]
                TXN4["POST /remittance-transactions"]
                TXN5["POST /corporate-transactions"]
                TXN6["POST /pesonet-transactions"]
                TXN7["POST /qr-emvco-transactions"]
                TXN8["POST /ibank-fund-transfers"]
                TXN9["POST /bulk-pepp-disbursements"]
            end

            subgraph CUST_DOMAIN["Customer Domain"]
                CUST1["GET /customers/{id}"]
                CUST2["GET /customers/{id}/accounts"]
                CUST3["POST /customers/search"]
            end

            subgraph REPORT_DOMAIN["Reports Domain"]
                RPT1["POST /reports/daily-summary"]
                RPT2["POST /reports/settlement"]
            end
        end

        subgraph VPC["VPC (Private Subnets)"]
            SM["Secrets Manager"]
            RS["Amazon Redshift\nServerless"]
        end

        CW["CloudWatch\n(Logging & Monitoring)"]
    end

    A1 & A2 & A3 & A4 & A5 -->|"HTTPS + Bearer Token"| GATEWAY
    GATEWAY -->|"Validate"| AUTHORIZER
    AUTHORIZER <-->|"Token + Scope\nLookup"| DDB
    AUTHORIZER -->|"Allow/Deny"| GATEWAY

    GATEWAY --> TXN_DOMAIN
    GATEWAY --> CUST_DOMAIN
    GATEWAY --> REPORT_DOMAIN

    TXN_DOMAIN & CUST_DOMAIN & REPORT_DOMAIN --> SM --> RS
    TXN_DOMAIN & CUST_DOMAIN & REPORT_DOMAIN --> CW

    style GATEWAY fill:#FF9900,color:#fff
    style AUTHORIZER fill:#FF9900,color:#fff
    style DDB fill:#3B48CC,color:#fff
    style SM fill:#DD344C,color:#fff
    style RS fill:#3B48CC,color:#fff
    style CW fill:#FF9900,color:#fff
```

---

## 5. Proposed — Detailed Request Flow

```mermaid
sequenceDiagram
    participant App as Client App
    participant APIGW as API Gateway
    participant Auth as Lambda Authorizer
    participant DDB as DynamoDB
    participant Lambda as Lambda Handler
    participant SM as Secrets Manager
    participant RS as Redshift

    Note over App,RS: Token Generation (One-time)
    App->>APIGW: POST /v1/api/generate_token<br>{sub, aud, description}
    APIGW->>Lambda: Invoke (no auth required)
    Lambda->>DDB: Check existing token
    alt Token exists and valid
        DDB-->>Lambda: Return existing token
        Lambda-->>App: 200 {token, expires_at}
    else Token expired or not found
        Lambda->>DDB: Store new token + secret
        Lambda-->>App: 200 {token, expires_at}
    end

    Note over App,RS: API Request Flow
    App->>APIGW: POST /v1/api/transactions<br>Authorization: Bearer <token>
    APIGW->>Auth: Validate token
    Auth->>Auth: Decode JWT (get sub + aud)
    Auth->>DDB: Get secret by sub + aud
    DDB-->>Auth: {secret, scope}
    Auth->>Auth: Verify signature + expiry

    alt Valid Token
        Auth-->>APIGW: Allow (IAM Policy)
        APIGW->>Lambda: Forward request
        Lambda->>SM: Get Redshift credentials
        SM-->>Lambda: {username, password}
        Lambda->>RS: SQL Query
        RS-->>Lambda: Result rows
        Lambda-->>App: 200 {result: {data: [...]}}
    else Invalid/Expired Token
        Auth-->>APIGW: Deny (IAM Policy)
        APIGW-->>App: 403 Forbidden
    end
```

---

## 6. Proposed — Security & Access Control

```mermaid
flowchart TB
    subgraph ACCESS["Access Control Matrix"]
        direction TB
        subgraph TOKENS["Token Scopes"]
            AGENT["sub: agent\n(Branch/Agent Apps)"]
            CORP["sub: corporate\n(Corporate Apps)"]
            INTERNAL["sub: internal\n(Internal Tools)"]
        end

        subgraph PERMISSIONS["Endpoint Permissions"]
            TXN_EP["Transaction Endpoints"]
            CUST_EP["Customer Endpoints"]
            RPT_EP["Report Endpoints"]
        end
    end

    AGENT -->|"✅ Allowed"| TXN_EP
    AGENT -->|"❌ Denied"| CUST_EP
    AGENT -->|"❌ Denied"| RPT_EP

    CORP -->|"✅ Allowed"| TXN_EP
    CORP -->|"✅ Allowed"| CUST_EP
    CORP -->|"❌ Denied"| RPT_EP

    INTERNAL -->|"✅ Allowed"| TXN_EP
    INTERNAL -->|"✅ Allowed"| CUST_EP
    INTERNAL -->|"✅ Allowed"| RPT_EP

    style AGENT fill:#2E8B57,color:#fff
    style CORP fill:#FF9900,color:#fff
    style INTERNAL fill:#3B48CC,color:#fff
```

---

## 7. Migration Path

```mermaid
flowchart LR
    subgraph PHASE1["Phase 1 ✅ (Done)"]
        direction TB
        P1A["Transaction History API"]
        P1B["JWT Auth (cauth)"]
        P1C["API Gateway + Lambda"]
    end

    subgraph PHASE2["Phase 2 (Current)"]
        direction TB
        P2A["Centralize API Gateway"]
        P2B["Add Role-Based Access"]
        P2C["Onboard Other Apps"]
    end

    subgraph PHASE3["Phase 3 (Future)"]
        direction TB
        P3A["Rate Limiting"]
        P3B["API Versioning"]
        P3C["Remove Direct DB Access"]
    end

    PHASE1 -->|"Blueprint"| PHASE2 -->|"Scale"| PHASE3

    style PHASE1 fill:#2E8B57,color:#fff
    style PHASE2 fill:#FF9900,color:#fff
    style PHASE3 fill:#3B48CC,color:#fff
```

---

## How to View These Diagrams

1. **VS Code** — Install "Markdown Preview Mermaid Support" extension, then preview this file
2. **GitHub** — Push this file; GitHub natively renders Mermaid diagrams
3. **Mermaid Live Editor** — Copy any diagram block to https://mermaid.live
4. **Export to PNG/SVG** — Use Mermaid Live Editor to export for slides

