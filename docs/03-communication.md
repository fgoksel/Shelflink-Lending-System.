# 03 Communication

ShelfLink uses two styles of communication on purpose.

| Style | Used for | Why |
|---|---|---|
| Synchronous HTTP | loans-service asking catalog-service to reserve a copy | The caller needs the answer before it can continue |
| Asynchronous events | loans-service announcing a loan to notification-service | The caller must not wait for, or depend on, the consumer |

```mermaid
flowchart LR
    subgraph Sync["Synchronous: caller waits"]
        L1["loans-service"] -- "POST /books/id/reserve, 2 s timeout" --> C1["catalog-service"]
        C1 -- "200 or 409 or 404" --> L1
    end
    subgraph Async["Asynchronous: caller moves on"]
        L2["loans-service"] -- "publish loan.created" --> Q[("RabbitMQ")]
        Q -- "deliver whenever ready" --> N2["notification-service"]
    end
```

## Register and log in

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant W as web
    participant G as gateway
    participant A as auth-service
    U->>W: submit credentials
    W->>G: POST /api/auth/login
    G->>A: POST /login
    A->>A: compare bcrypt hash
    A-->>G: 200 token
    G-->>W: 200 token
    W-->>U: show catalog, keep token in browser
```

## Search the catalog

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant W as web
    participant G as gateway
    participant C as catalog-service
    U->>W: type in search box
    W->>G: GET /api/books?search=clean
    G->>C: GET /books?search=clean
    C-->>G: matching books with availability
    G-->>W: JSON
    W-->>U: render book cards
```

## Borrow a book

This is the main flow. It combines a synchronous call, a local write and an asynchronous event.

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant W as web
    participant G as gateway
    participant L as loans-service
    participant C as catalog-service
    participant MQ as RabbitMQ
    participant N as notification-service
    U->>W: click Borrow
    W->>G: POST /api/loans with Bearer token
    G->>L: POST /loans
    L->>L: verify JWT locally
    L->>C: POST /books/id/reserve
    C->>C: decrement only if copies are left
    C-->>L: 200 book
    L->>L: insert loan, due in 14 days
    L-)MQ: publish loan.created
    L-->>G: 201 loan
    G-->>W: 201 loan
    W-->>U: show confirmation
    MQ-)N: deliver loan.created
    N->>N: store notification
    W->>G: GET /api/notifications/mine on next poll
    G->>N: GET /notifications/mine
    N-->>W: new notification
```

Steps 12 onward happen after the user already has their answer.

## Return a book

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant W as web
    participant G as gateway
    participant L as loans-service
    participant C as catalog-service
    participant MQ as RabbitMQ
    U->>W: click Return
    W->>G: POST /api/loans/id/return
    G->>L: POST /loans/id/return
    L->>L: check loan exists, is yours, not returned
    L->>C: POST /books/id/release
    C-->>L: 200
    L->>L: set returned_at
    L-)MQ: publish loan.returned
    L-->>W: 200 loan
```

## Loan lifecycle

```mermaid
stateDiagram-v2
    [*] --> Requested: user clicks Borrow
    Requested --> Rejected: no copies left, 409
    Requested --> Failed: catalog unreachable, 503
    Requested --> Active: copy reserved, loan saved
    Active --> Returned: user clicks Return
    Rejected --> [*]
    Failed --> [*]
    Returned --> [*]
```

## Failure behaviour

### Catalog is down

Loans-service never hangs waiting for catalog. A stopped container refuses the connection, so the answer is an immediate `503`. A catalog that is running but unresponsive makes loans-service give up after 2 seconds and answer `503`. In both cases no loan is saved.

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant L as loans-service
    participant C as catalog-service
    U->>L: borrow request
    L->>C: reserve copy
    Note over C: container stopped or hung
    L--xC: refused, or no response within 2 seconds
    L-->>U: 503 catalog-service unavailable
```

A timeout means the outcome is unknown, not that the request failed. If a slow catalog finishes the reservation after loans-service has given up, one copy stays reserved with no loan attached. This was observed while testing with a deliberately frozen catalog process. Closing the gap properly needs idempotent reservation ids, which is listed under [known limitations](07-design-decisions.md#known-limitations).

### Notifications is down

The borrow succeeds, because loans-service never calls notification-service. The event waits in a durable queue and is delivered when the consumer comes back.

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant L as loans-service
    participant MQ as RabbitMQ
    participant N as notification-service
    Note over N: container stopped
    U->>L: borrow request
    L-->>U: 201 loan created
    L-)MQ: publish loan.created
    MQ->>MQ: hold message in durable queue
    Note over N: container started again
    MQ-)N: deliver held message
    N->>N: store notification
```

### Loan could not be saved after reserving

If the reservation succeeds but saving the loan fails, loans-service releases the copy again, so no copy is leaked.

```mermaid
flowchart TD
    R["Reserve copy in catalog"] --> OK{"Reserved?"}
    OK -- "no" --> F["Return 409 or 404 or 503"]
    OK -- "yes" --> S["Insert loan row"]
    S --> SOK{"Saved?"}
    SOK -- "yes" --> P["Publish loan.created, return 201"]
    SOK -- "no" --> REL["Compensate: release the copy"]
    REL --> E["Return 500"]
```

### Two users take the last copy

Catalog does the check and the decrement in a single SQL statement, so only one request can win.

```mermaid
sequenceDiagram
    autonumber
    participant A as Request A
    participant B as Request B
    participant C as catalog-service
    A->>C: reserve book with 1 copy left
    B->>C: reserve same book
    C->>C: UPDATE ... WHERE available_copies > 0
    C-->>A: 200 reserved
    C->>C: UPDATE ... WHERE available_copies > 0 changes 0 rows
    C-->>B: 409 no copies available
```

## Events

| Event | Published by | Payload | Consumed by |
|---|---|---|---|
| `loan.created` | loans-service | loanId, userId, username, bookId, bookTitle, dueAt | notification-service |
| `loan.returned` | loans-service | loanId, userId, username, bookId, bookTitle | notification-service |

Events travel through a fanout exchange named `library_events`. Notification-service binds a durable queue named `notifications_queue`. A second consumer, for example an email service, could bind its own queue without any change to loans-service.
