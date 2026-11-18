# Technical Design: [Feature Name]

## 1. High-Level Architecture & Flow

Provide a narrative overview of how this feature integrates into the AiraMeet architecture.

```mermaid
flowchart TD
    Client["Client Interface (Angular)"] <--> Gateway["API Gateway / Backend (ASP.NET Core)"]
    Gateway <--> Database[("Database (SQL Server)")]
    Gateway <--> AiService["AI Service (FastAPI)"]
```

---

## 2. Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor User as User
    participant Client as Client
    participant Backend as Backend
    participant DB as Database

    User->>Client: Initiate Action
    Client->>Backend: Request with JWT
    Backend->>DB: Query / Mutate State
    Backend-->>Client: Result Payload
    Client-->>User: Render UI Update
```

---

## 3. Data Schema Modifications

Detail any Entity Framework Core migrations or table schema changes required.

```sql
-- Schema DDL additions if applicable
```

---

## 4. API & Communication Contracts

Specify REST routes, request/response JSON schemas, and WebSocket message formats.
