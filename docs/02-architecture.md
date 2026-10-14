# 02 Architecture

## Component view

The browser only ever talks to the `web` service. The `web` service forwards API calls to the gateway, and the gateway is the only component that knows where the backend services live.

```mermaid
flowchart TB
    subgraph Client["Client tier"]
        BR["Browser single-page app"]
    end
    subgraph Edge["Edge tier"]
        WEB["web :8080"]
        GW["api-gateway :3000"]
    end
    subgraph Services["Service tier"]
        AUTH["auth-service :3001"]
        CAT["catalog-service :3002"]
        LOAN["loans-service :3003"]
        NOTE["notification-service :3004"]
    end
    subgraph Data["Data tier"]
        ADB[("auth.db")]
        CDB[("catalog.db")]
        LDB[("loans.db")]
        NDB[("notifications.db")]
        MQ[("RabbitMQ")]
    end

    BR -- "HTTP" --> WEB
    WEB -- "/api/* forwarded" --> GW
    GW --> AUTH
    GW --> CAT
    GW --> LOAN
    GW --> NOTE
    LOAN -- "reserve / release" --> CAT
    LOAN -- "publish events" --> MQ
    MQ -- "consume events" --> NOTE
    AUTH --- ADB
    CAT --- CDB
    LOAN --- LDB
    NOTE --- NDB
```

## Deployment view

Everything runs as containers on one Docker Compose network. Containers reach each other by service name through Docker's built-in DNS, so `http://catalog:3002` just works.

```mermaid
flowchart LR
    subgraph Host["Your computer"]
        P1["localhost:8080"]
        P2["localhost:3000"]
        P3["localhost:15672"]
        subgraph Net["Docker Compose network"]
            web["web"]
            gateway["gateway"]
            auth["auth"]
            catalog["catalog"]
            loans["loans"]
            notifications["notifications"]
            rabbitmq["rabbitmq"]
        end
        V1[("auth-data volume")]
        V2[("catalog-data volume")]
        V3[("loans-data volume")]
        V4[("notifications-data volume")]
    end
    P1 --> web
    P2 --> gateway
    P3 --> rabbitmq
    auth --- V1
    catalog --- V2
    loans --- V3
    notifications --- V4
```

Only three ports are published to the host. The four backend services are reachable only from inside the network.

## Responsibilities by layer

| Layer | Does | Deliberately does not |
|---|---|---|
| web | Serves static files, forwards `/api` | Contain business logic |
| gateway | Routes by URL prefix | Authenticate or transform requests |
| services | Own business rules and data | Read each other's databases |
| RabbitMQ | Holds events until consumed | Know anything about the business |

## Authentication flow

Auth-service issues a JWT. Loans and notifications verify it themselves with the shared secret, so no service calls auth-service on every request.

```mermaid
flowchart TD
    L["POST /auth/login"] --> V{"Credentials valid?"}
    V -- "no" --> E["401 invalid credentials"]
    V -- "yes" --> T["JWT signed with JWT_SECRET, 12 hour expiry"]
    T --> S["Browser stores token"]
    S --> R["Later request with Authorization: Bearer token"]
    R --> C{"Service verifies signature and expiry"}
    C -- "invalid" --> U["401 invalid or expired token"]
    C -- "valid" --> OK["Request handled for user id in token"]
```

## Data ownership

Each service has a private database. Other services learn about its data only through its API or through events, never through a shared table.

```mermaid
flowchart LR
    subgraph own1["auth-service"]
        D1[("users")]
    end
    subgraph own2["catalog-service"]
        D2[("books")]
    end
    subgraph own3["loans-service"]
        D3[("loans")]
    end
    subgraph own4["notification-service"]
        D4[("notifications")]
    end
    D3 -. "stores user id and book id only, no foreign key" .-> D1
    D3 -. "stores book id and title copy" .-> D2
    D4 -. "stores user id only" .-> D1
```
