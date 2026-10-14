# 04 Data model

Each service owns one SQLite database file, kept in its own Docker volume. There are no foreign keys between databases. Where one service refers to another service's data, it stores a plain id, and the meaning of that id is only enforced by the API.

## auth-service

```mermaid
erDiagram
    USERS {
        int id PK
        string username UK
        string password_hash
        datetime created_at
    }
```

Passwords are stored as bcrypt hashes, never in plain text.

## catalog-service

```mermaid
erDiagram
    BOOKS {
        int id PK
        string title
        string author
        int total_copies
        int available_copies
    }
```

`available_copies` is the single source of truth for availability. It is changed only by the reserve and release endpoints.

## loans-service

```mermaid
erDiagram
    LOANS {
        int id PK
        int user_id
        int book_id
        string book_title
        datetime borrowed_at
        datetime due_at
        datetime returned_at
    }
```

`user_id` and `book_id` are references to other services' data. `book_title` is a deliberate copy taken at borrow time so the loan list can be shown without calling catalog-service for every row.

## notification-service

```mermaid
erDiagram
    NOTIFICATIONS {
        int id PK
        int user_id
        string message
        datetime created_at
    }
```

## How the services refer to each other

```mermaid
erDiagram
    USERS ||--o{ LOANS : "referenced by user_id, no foreign key"
    BOOKS ||--o{ LOANS : "referenced by book_id, no foreign key"
    USERS ||--o{ NOTIFICATIONS : "referenced by user_id, no foreign key"
    USERS {
        int id PK
    }
    BOOKS {
        int id PK
    }
    LOANS {
        int id PK
        int user_id
        int book_id
    }
    NOTIFICATIONS {
        int id PK
        int user_id
    }
```

This diagram is logical only. In reality USERS, BOOKS, LOANS and NOTIFICATIONS live in four separate databases, so no join between them is ever possible. Combining their data is the job of the calling service or the browser.

## Seed data

Catalog-service inserts four books the first time it starts with an empty database.

| Title | Author | Copies |
|---|---|---|
| Clean Code | Robert C. Martin | 2 |
| Designing Data-Intensive Applications | Martin Kleppmann | 1 |
| Building Microservices | Sam Newman | 3 |
| The Pragmatic Programmer | David Thomas and Andrew Hunt | 2 |
