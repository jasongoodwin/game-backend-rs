# Game Backend Architecture Document

``` mermaid
graph TB
    subgraph Clients
        C1[Client 1]
        C2[Client 2]
        C3[Client N]
    end

    subgraph Backend
        subgraph EntryPoint[Entry Points]
            AG[API Gateway]
            GCS[Global Connection Server]
        end

        subgraph Auth[Authentication]
            LS[Login Service]
            DB[User Database]
            TS[Token Service]
        end

        subgraph GameServers[Game Servers]
            MS[Matchmaking Service]
            subgraph K8S[Kubernetes]
                IS[Instance Servers]
            end
            GD[Game Data DB]
        end

        subgraph EventSystem[Event & Monitoring]
            K[Kafka Event Bus]
            CS[Chat Service]
            M[Metrics]
            L[Logging]
        end
    end

    %% Main flows
    C1 & C2 & C3 --> AG
    C1 & C2 & C3 --> GCS
    
    %% Auth flow
    AG --> LS
    LS --> DB
    LS <--> TS
    GCS <--> TS
    
    %% Game flow
    GCS --> MS
    MS --> IS
    IS --> GD

    %% Event system
    GCS & LS & MS & IS -.-> K
    K -.-> CS
    CS --> GCS
    K --> M

    %% Logging
    AG & LS & GCS & MS & IS & TS --> L

    %% Styling
    style TS fill:#ccf
    style K fill:#fcf
    style K8S fill:#cfc
    style Auth fill:#fee
    style EntryPoint fill:#dfd
    style EventSystem fill:#eff
```

# 1. Overview

This document describes the architecture for the backend of a multiplayer card game with crafting/farming elements. The architecture is designed for scalability, resilience, and security, leveraging microservices, containerization, and event-driven communication.

# 2. Components

Clients: Godot game clients (desktop, mobile, or web).

API Gateway (AG):

Entry point for all client requests.

Handles authentication/authorization (delegating to the Login Service and Token Service).

Performs rate limiting to prevent DoS attacks.

Routes requests to the appropriate backend services.

Can transform requests (e.g., adding headers).

Technology: Kong, Tyk, AWS API Gateway, Azure API Management, or similar.

Authentication (Auth) Subsystem:

Login Service (LS):

REST API for user authentication.

Validates credentials against the User Database.

Interacts with the Token Service to generate session tokens.

Publishes user_login events to Kafka.

Technology: Python (Flask/FastAPI), Node.js (Express), Go, or similar.

User Database (DB):

Stores user account information (usernames, hashed passwords, etc.).

Technology: Cloud SQL, Cloud Spanner (GCP), or similar.

Token Service (TS):

Generates, validates, and revokes security tokens (session tokens, game server access tokens).

Uses strong cryptography (HMAC-SHA256 or RSA signatures).

Stores tokens securely.

Technology: Go, Python, Node.js, with libraries like PyJWT or golang-jwt/jwt.

Game Servers Subsystem:

Global Connection Server (GCS):

Godot server (headless).

Maintains persistent connections (ENet with DTLS or WebSockets) with clients.

Validates session tokens (using the Token Service).

Manages client state.

Pushes global notifications to clients (received from Kafka).

Forwards requests to the Matchmaking Service.

Handles lightweight chat.

Matchmaking Service (MS):

Queues players for matches.

Implements matchmaking logic.

Requests new game server instances from Kubernetes (via the Kubernetes API).

Publishes match_found events to Kafka.

Provides game server connection information to clients (via the GCS).

Kubernetes (K8S):

Container orchestration platform.

Manages the Instanced Game Servers.

Instanced Game Servers (IS1, IS2, ISN):

Godot server builds (headless).

Run the actual game logic for a single match/farming instance.

Communicate with clients via ENet (with DTLS).

Validate session tokens (using the Token Service).

Enforce game rules.

Publish game-related events to Kafka.

Kafka Event Bus (K):

Asynchronous communication between services.

Decouples services.

Used for event sourcing (for monitoring).

Technology: Apache Kafka, Confluent Cloud, or similar.

Chat Service (CS):

Handles real-time chat functionality (optional, could be integrated into the GCS for smaller scale).

Can use WebSockets or ENet.

Game Data DB (GD):

Stores persistent game data (card definitions, crafting recipes, etc.).

Technology: Cloud SQL, Cloud Spanner, or similar.

Monitoring Subsystem:

Metrics & Alerting (M):

Collects and stores metrics from all services (CPU, memory, request latency, error rates, ENet connections, Kafka lag, etc.).

Provides dashboards and alerting for critical conditions.

Technology: Prometheus, Grafana, Datadog, GCP Monitoring, or similar.

Logging Service (L):

Centralized logging for all services.

Uses structured logging (JSON format).

Allows for searching, filtering, and analysis of logs.

Technology: Elasticsearch, Logstash, Kibana (ELK stack), Fluentd, GCP Logging, or similar.

# 3. Data Flow (Example: Login and Matchmaking)

Client sends credentials to the API Gateway.

API Gateway forwards the request to the Login Service.

Login Service validates credentials against the User Database.

Login Service requests a session token from the Token Service.

Token Service generates the token and returns it to the Login Service.

Login Service returns the session token to the client (via the API Gateway).

Login Service publishes a user_login event to Kafka.

Client connects to the Global Connection Server (GCS) using the session token.

GCS validates the token with the Token Service.

Client sends a "find_match" request to the GCS.

GCS forwards the request to the Matchmaking Service.

Matchmaking Service queues the player and finds a match.

Matchmaking Service requests a new game server instance from Kubernetes (if needed).

Matchmaking Service publishes a match_found event to Kafka.

Matchmaking Service sends game server connection info to the GCS.

GCS relays the connection info to the client.

Client connects to the Instanced Game Server via ENet (with DTLS), including the session token.

Instanced Game Server validates the session token with the Token Service.

The game begins.

# 4. Technology Stack

Godot Engine: Game client and server development.

ENet: Real-time networking (with DTLS).

Kubernetes: Container orchestration.

Kafka: Event bus.

Python/Flask/FastAPI, Node.js/Express, Go: For REST APIs (Login Service, Matchmaking Service).

Cloud SQL/Cloud Spanner (GCP): Databases.

Redis/Memcached: Caching.

Prometheus/Grafana/Datadog: Metrics and alerting.

ELK Stack/Fluentd/GCP Logging: Logging.

Kong/Tyk/AWS API Gateway/Azure API Management: API Gateway (optional, but recommended).

Hashicorp Vault / Cloud Provider Secret Manager Secret storage.

# 5. Security Considerations

DTLS for ENet: Encrypt all ENet communication.

HTTPS for REST APIs: Use HTTPS for all API communication.

Session Tokens: Use cryptographically secure session tokens, managed by a dedicated Token Service.

API Gateway: Centralized authentication, authorization, and rate limiting.

Input Validation: Validate all input on the server-side.

Database Security: Use SSL/TLS, restricted access, and least privilege for database connections.

Secret Management: Store secrets securely using a dedicated secret management service.

Monitoring and Alerting

# 6. Day 1 Features

Core Game Loop: Basic card game mechanics, farming/crafting (simplified), and a single common area.

Login/Authentication: User registration (handled externally), login with session token generation.

Global Connection Server: Persistent connections, basic user presence (online/offline), friend login notifications.

Matchmaking (Basic): A simple queue-based matchmaking system.

Instanced Game Servers: Ability to start and play a basic card game match.

ENet Communication (with DTLS): Secure communication between clients and game servers.

Kafka Eventing (Basic): user_login, match_found, game_started, game_ended events.

Metrics and Logging: Basic metrics (CPU, memory, request counts) and logging for all services.

Database Persistence: User accounts, basic player inventory, card definitions.

Kubernetes Deployment: Deployment of game servers to Kubernetes.

Token service

# 7. Future Enhancements

Sharding

More advanced matchmaking

More complex farming.

NFT integration.
