# v0.0.1 - Document Introduction and Extra Notes

This is my stab at an architecture for a globally scalable online real-time farming simulator and card battler.
This is not EXACTLY how I would structure a multiplayer FPS or other game types which requires optimizing for latency above _all else_ but it's suitable for games like an MMO such as FFXIV where there is a global cool down and tick can be like 5 or 10hz without interfering with gameplay experience.
If you wanted to build a twitch-shooter or another game with extreme low latency requirements, I'd just get an ENet connection or similar directly to the instanced server, and have the instanced server emit events to kafka async outside of the core gameloop on dedicated threads given less priority than the core loop and player connection threads.
For messages, I would use a binary protocol like protocol buffers, and then compress them with snappy or LZ4 or whatever is hot today. That puts a bit of load on the server, but the reduction in bandwidth vs the cpu cost/time of compression/decompression will result in faster byte transfer and lower network congestion, which would improve the scalability and consistency of the performance characteristics of the server. For a card battler, this will be less important so some of these details are omitted and will be harder for someone less experienced to understand how to implement. It's not hard but it's not what people will be used to doing.

I'll add more details as I go, this is a first look. This omits region segmentation, and cross region replication, but it's easy to see how you would make this work with kafka for example, where you can just replicate events at the expense of a bunch of latency on any events that are cross-regional. Likely you just want to keep users on regional global servers playing on instanced game servers where players are matched based on approximated latency to sub-regions. 

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

```

# Harvest Battles/Gene Weaver: Card Game & Farming Simulator Backend

## Introduction

Harvest Battles is a unique multiplayer game that combines strategic card battles with farming simulation elements, designed to create an engaging player experience that bridges traditional gaming with Web3 technology. The game allows players to cultivate resources, craft cards, battle other players, and eventually mint their unique cards as NFTs.

## System Architecture Overview

The backend architecture is designed to support a scalable, resilient, and secure gaming environment. It employs a microservices architecture deployed on Kubernetes, using event-driven communication patterns to ensure responsiveness and reliability.

### Key Features

- Real-time card battles with secure multiplayer networking
- Persistent farming and resource management systems
- Crafting system for creating unique cards
- Future NFT integration for card ownership and trading
- Cross-platform support (desktop, mobile, web)
- Scalable infrastructure supporting thousands of concurrent players

### Technical Stack

Our backend leverages modern cloud-native technologies:

- **Game Engine**: Godot Engine for both client and server components
- **Networking**: ENet protocol with DTLS encryption for real-time communication
- **Container Orchestration**: Kubernetes for scaling and managing game server instances
- **Event Bus**: Apache Kafka for service communication and event sourcing
- **Databases**: 
  - Cloud SQL/Cloud Spanner for user and game data
  - Redis for caching and real-time state management
- **Security**: 
  - API Gateway for request management and rate limiting
  - Token-based authentication with dedicated service
  - HTTPS/DTLS encryption for all communication
- **Monitoring**: 
  - Prometheus and Grafana for metrics and alerting
  - ELK Stack for centralized logging

## Service Architecture

Our backend is composed of several key service groups, each handling specific aspects of the game:

### Entry Points

The system provides two main entry points for clients:

1. **API Gateway**: Handles HTTP/REST requests for non-real-time operations such as:
   - User authentication
   - Resource management
   - Game state queries
   - Future NFT operations

2. **Global Connection Server**: Manages real-time connections using ENet/WebSockets for:
   - Live game sessions
   - Real-time notifications
   - Chat functionality
   - Player presence management

### Authentication System

The authentication system ensures secure access to game resources through:

- Login Service for credential validation and session management
- Token Service for secure session handling
- User Database for account management
- Integration points for future Web3 wallet authentication

### Game Servers

The game server infrastructure dynamically scales to match player demand:

- Matchmaking Service for pairing players based on skill and preferences
- Instance Servers running actual game sessions
- Kubernetes orchestration for automatic scaling
- Game Data DB for storing game state and player progress

# Harvest Battles: Card Game & Farming Simulator Backend

## Introduction

Harvest Battles is a unique multiplayer game that combines strategic card battles with farming simulation elements, designed to create an engaging player experience that bridges traditional gaming with Web3 technology. The game allows players to cultivate resources, craft cards, battle other players, and eventually mint their unique cards as NFTs.

## System Architecture Overview

The backend architecture is designed to support a scalable, resilient, and secure gaming environment. It employs a microservices architecture deployed on Kubernetes, using event-driven communication patterns to ensure responsiveness and reliability.

### Key Features

- Real-time card battles with secure multiplayer networking
- Persistent farming and resource management systems
- Crafting system for creating unique cards
- Future NFT integration for card ownership and trading
- Cross-platform support (desktop, mobile, web)
- Scalable infrastructure supporting thousands of concurrent players

### Technical Stack

Our backend leverages modern cloud-native technologies:

- **Game Engine**: Godot Engine for both client and server components
- **Networking**: ENet protocol with DTLS encryption for real-time communication
- **Container Orchestration**: Kubernetes for scaling and managing game server instances
- **Event Bus**: Apache Kafka for service communication and event sourcing
- **Databases**: 
  - Cloud SQL/Cloud Spanner for user and game data
  - Redis for caching and real-time state management
- **Security**: 
  - API Gateway for request management and rate limiting
  - Token-based authentication with dedicated service
  - HTTPS/DTLS encryption for all communication
- **Monitoring**: 
  - Prometheus and Grafana for metrics and alerting
  - ELK Stack for centralized logging

## Service Architecture

Our backend is composed of several key service groups, each handling specific aspects of the game:

### Entry Points

The system provides two main entry points for clients:

1. **API Gateway**: Handles HTTP/REST requests for non-real-time operations such as:
   - User authentication
   - Resource management
   - Game state queries
   - Future NFT operations

2. **Global Connection Server**: Manages real-time connections using ENet/WebSockets for:
   - Live game sessions
   - Real-time notifications
   - Chat functionality
   - Player presence management

### Authentication System

The authentication system ensures secure access to game resources through:

- Login Service for credential validation and session management
- Token Service for secure session handling
- User Database for account management
- Integration points for future Web3 wallet authentication

### Game Servers

The game server infrastructure dynamically scales to match player demand:

- Matchmaking Service for pairing players based on skill and preferences
- Instance Servers running actual game sessions
- Kubernetes orchestration for automatic scaling
- Game Data DB for storing game state and player progress

## Event System & Real-time Communication

The event system serves as the backbone of our real-time gaming infrastructure, enabling seamless communication between services and maintaining game state consistency.

### Kafka Event Bus

Our event bus handles several critical streams of information:

- **User Events**: Login/logout, friend requests, achievements
- **Game Events**: Match starts/ends, player actions, resource updates
- **System Events**: Server scaling, performance metrics, error reporting
- **Future Web3 Events**: NFT mints, trades, marketplace activities

Events are structured with clear schemas and versioning, allowing for system evolution while maintaining backward compatibility.

### Real-time Communication Patterns

The system employs different communication patterns based on requirements:

- **Request-Response**: For authenticated API calls through the gateway
- **Pub/Sub**: For game events and notifications via Kafka
- **WebSocket/ENet**: For real-time game state updates and chat
- **Event Sourcing**: For game replay and state reconstruction capabilities

## Monitoring & Operations

Our comprehensive monitoring solution ensures reliable operation and quick problem resolution.

### Metrics & Alerting

The monitoring system tracks key performance indicators:

- Server health (CPU, memory, network usage)
- Game metrics (matches/hour, concurrent players, queue times)
- Business metrics (user engagement, retention rates)
- Network performance (latency, packet loss, connection stability)

Custom alerts are configured for:

- Service degradation or failures
- Unusual player behavior patterns
- Security incidents
- Resource utilization thresholds

### Logging & Diagnostics

Centralized logging provides:

- Structured logs in JSON format for easy querying
- Request tracing across services
- Error aggregation and analysis
- Audit trails for security events

## Data Flows

### Player Authentication Flow

1. Client connects to API Gateway with credentials
2. Gateway routes to Login Service
3. Login Service validates against User Database
4. Token Service generates session token
5. Client receives token and connects to Global Connection Server
6. All subsequent requests include the validated token

### Matchmaking & Game Session Flow

1. Player requests match through Global Connection Server
2. Matchmaking Service processes request and queues player
3. When match is found:
   - New game instance is provisioned in Kubernetes
   - Players receive connection details
   - Instance Server validates players via Token Service
4. Game session begins with encrypted ENet communication
5. Game state updates are persisted to Game Data DB
6. Match results trigger events for rewards and statistics

(if lower latency gameplay is needed, kafka should not be used as an intermediate, but the user should connect directly to the instance server, and events should be generated async out of the instance server and put into kafka for telemetry data.) 

### Resource Management Flow

1. Player farming actions are validated by Instance Servers
2. Resource updates are persisted to Game Data DB
3. Crafting requests are validated against player inventory
4. New cards are created and added to player collection
5. Future: NFT minting process for eligible cards

## Game Loop Integration

The game combines two main loops that interact seamlessly:

### Farming Loop

- Resource cultivation and management
- Time-based growth and harvesting mechanics
- Resource market interactions
- Crafting system for creating new cards

### Battle Loop

- Deck building and management
- Matchmaking and game sessions
- Real-time card battles
- Ranking and reward systems

These loops intersect through:

- Resource requirements for card crafting
- Card usage in battles
- Rewards feeding back into farming economy
- Future NFT integration for both systems


## Web3 Integration

The game's Web3 features are designed to enhance player ownership and economic opportunities while maintaining traditional gaming appeal.

### NFT Implementation

Our NFT system will focus on meaningful digital ownership:

- **Card NFTs**: Unique cards can be minted as NFTs, with attributes including:
  - Rarity and edition numbers
  - Crafting history and originating player
  - Battle statistics and achievements
  - Visual customizations

- **Land NFTs**: Future expansion allowing ownership of premium farming plots:
  - Unique resource generation rates
  - Special crafting capabilities
  - Customizable appearances
  - Rentable to other players

### Blockchain Integration

The system will utilize a hybrid approach:

- **Layer 2 Solution**: Using ImmutableX/Polygon for:
  - Low transaction costs
  - High transaction throughput
  - Environmental sustainability
  - Seamless user experience

- **Smart Contracts**: Managing:
  - NFT minting and trading
  - Marketplace functionality
  - Governance mechanisms
  - Revenue sharing

### Web3 Architecture Components

New services will be added to support Web3 features:

- **Wallet Service**: Managing player wallet connections
- **NFT Metadata Service**: Handling NFT attributes and updates
- **Marketplace Service**: Supporting NFT trading
- **Blockchain Indexer**: Tracking on-chain events

## Deployment & DevOps

### Infrastructure as Code

All infrastructure is defined using:

- Terraform for cloud resources
- Helm charts for Kubernetes deployments
- GitHub Actions for CI/CD pipelines

### Deployment Environments

- **Development**: Local Kubernetes cluster with simulated load
- **Staging**: Cloud-based replica of production
- **Production**: Multi-region deployment with:
  - Geographic load balancing
  - Regional game servers
  - Distributed databases
  - CDN for static assets

### Scaling Strategy

The system scales both vertically and horizontally:

- **Auto-scaling** based on:
  - Current player count
  - Match queue length
  - Server resource utilization
  - Geographic demand

- **Regional Deployment**:
  - Game servers near player populations
  - Database replication across regions
  - Content delivery optimization

## Security & Compliance

### Network Security

- **DDoS Protection**:
  - Cloud provider DDoS mitigation
  - Rate limiting at API Gateway
  - Traffic filtering and analysis

- **Encryption**:
  - TLS 1.3 for HTTPS
  - DTLS for game traffic
  - End-to-end encryption for sensitive data

### Game Security

- **Anti-Cheat Measures**:
  - Server-side validation
  - Client integrity checking
  - Behavior analysis
  - Automated suspension system

- **Smart Contract Security**:
  - External audits
  - Formal verification
  - Bug bounty program
  - Upgradeable contract design

### Data Protection

- **Player Data**:
  - GDPR compliance
  - Data minimization
  - Regular security audits
  - Encrypted backups

- **Wallet Security**:
  - No private key storage
  - Secure wallet linking
  - Transaction signing verification

## Future Roadmap

### Phase 1: Core Gaming (Current)
- Basic card game mechanics
- Farming system implementation
- Matchmaking and rankings
- Social features

### Phase 2: Web3 Foundation
- Wallet integration
- Basic NFT minting
- Marketplace foundation
- Token economics design

### Phase 3: Advanced Features
- Land NFT system
- Guild system
- Tournament infrastructure
- Cross-chain functionality

### Phase 4: Ecosystem Growth
- Developer API
- Custom card creation tools
- Community governance
- E-sports integration
