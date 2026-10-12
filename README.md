# Psychology Support Microservices

Backend for **EmoEase**, a mental-health support platform that combines an AI companion chat, an anonymous peer-support community and self-assessment tools, built as a set of .NET microservices.

![.NET 9](https://img.shields.io/badge/.NET-9.0-512BD4?logo=dotnet&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-pgvector-4169E1?logo=postgresql&logoColor=white)
![RabbitMQ](https://img.shields.io/badge/RabbitMQ-MassTransit-FF6600?logo=rabbitmq&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-cache%20%26%20locks-DC382D?logo=redis&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)

📖 In-depth architecture documentation: **[DeepWiki](https://deepwiki.com/anht04/psychology-support-microservices)**

## Overview

People looking for emotional support often want two things that pull in opposite directions: a personalised experience (an assistant that remembers them) and strong anonymity (a community where nobody can link a post to their real identity or health data). EmoEase addresses both:

- **AI companion** that classifies each message, retrieves relevant long-term memories and can trigger in-app tools such as a DASS-21 assessment.
- **Anonymous community** with posts, nested comments, reactions and a personalised feed, where users act through public *aliases* instead of their account.
- **Clinical and wellness features**: patient profiles, DASS-21 self-assessments, wellness modules, challenges and mood journaling.
- **Commerce**: subscription packages, promo and gift codes, digital goods and PayOS / VNPay payments.

The backend is split into domain-aligned services behind a YARP API gateway. Services own their data, communicate synchronously over gRPC where an immediate answer is needed, and asynchronously over RabbitMQ (MassTransit) for everything else. The CI pipeline builds and deploys only the services whose files changed.

## Engineering Highlights

**Identity isolation at the gateway.** Client tokens carry only an opaque subject reference. For each request, the gateway adds the one identifier the destination needs (an alias ID for social services, a patient ID for clinical ones), so services such as Post and Feed work with aliases and never receive a patient ID.

**Reliable event publishing.** Several services (Post, Alias, UserMemory, Wellness, Media and others) write integration events to an **outbox table in the same transaction** as their state change; a background processor then publishes them to RabbitMQ. An event is delayed, not lost, if the broker is unavailable at commit time.

**Idempotency and concurrency control.** A shared MediatR pipeline behaviour (`IdempotentCommandPipelineBehaviour`) returns the stored response when a command is retried with the same idempotency key, and Redis locks serialise reaction changes on the same target.

**Multi-stage LLM pipeline.** ChatBox routes each message with a lightweight Gemini call, runs a tool-selector call only for tool intents, retrieves memories from UserMemory over gRPC only when needed, and composes an intent-specific instruction for the final response model.

**AI moderation in the event flow.** New posts are moderated asynchronously: Post publishes `PostCreated`, AIModeration evaluates it with Gemini and publishes the verdict, and both Post (approve or reject) and Feed (index approved posts) react to it.

**Selective CI/CD.** GitHub Actions maps changed paths to services, builds Docker images only for those services, pushes them to GHCR and redeploys only the affected containers on the VPS.

## Architecture

The diagrams follow the [C4 model](https://c4model.com/): a System Context view for people and external systems, then a Container view for the services inside the system. They are drawn as Mermaid flowcharts using C4 notation (element type in brackets, technology on each relationship) because Mermaid's native C4 syntax is experimental and lays out labels poorly on GitHub.

**Legend (all diagrams):** solid arrow = synchronous call or dependency (HTTP, gRPC, SignalR, database access); dashed arrow = asynchronous message or trigger (RabbitMQ event, workflow trigger). In sequence diagrams, `-)` open arrows are asynchronous messages and dashed arrows are replies.

### System context

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 400}}}%%
flowchart TB
    user["User<br/>[Person]<br/>Chats with the AI companion, posts<br/>anonymously, takes assessments"]
    doctor["Doctor<br/>[Person]<br/>Maintains a professional profile"]
    staff["Admin / Manager<br/>[Person]<br/>Manages tests, media and payments"]

    emo["EmoEase backend<br/>[Software system]<br/>Psychology Support Microservices"]

    gemini["Google Gemini API<br/>[External system]"]
    firebase["Firebase<br/>[External system]"]
    pay["PayOS and VNPay<br/>[External system]"]
    blob["Azure Blob Storage<br/>[External system]"]
    sight["Sightengine<br/>[External system]"]
    smtp["SMTP server<br/>[External system]"]

    user -->|"Uses<br/>[HTTPS, WebSocket]"| emo
    doctor -->|"Uses<br/>[HTTPS]"| emo
    staff -->|"Uses<br/>[HTTPS]"| emo

    emo -->|"Generates replies, embeddings,<br/>moderation verdicts, images<br/>[HTTPS]"| gemini
    emo -->|"Verifies ID tokens,<br/>sends push messages<br/>[Firebase Admin SDK]"| firebase
    emo -->|"Creates payment links<br/>[HTTPS]"| pay
    pay -->|"Payment callbacks<br/>[HTTPS webhook]"| emo
    emo -->|"Stores media<br/>[HTTPS]"| blob
    emo -->|"Moderates images<br/>[HTTPS]"| sight
    emo -->|"Sends email<br/>[SMTP]"| smtp

    classDef person fill:#08427b,stroke:#052e56,color:#ffffff
    classDef system fill:#1168bd,stroke:#0b4884,color:#ffffff
    classDef external fill:#6b6b6b,stroke:#4d4d4d,color:#ffffff
    class user,doctor,staff person
    class emo system
    class gemini,firebase,pay,blob,sight,smtp external
```

Users reach the backend through Android, iOS and web clients (the `DeviceType` values in Auth); those clients live outside this repository.

### Containers

```mermaid
flowchart TB
    apps["Mobile and web apps<br/>[Container: outside this repo]"]

    subgraph emo["EmoEase backend"]
        gw["API Gateway<br/>[Container: ASP.NET Core, YARP]<br/>Routing, auth, token exchange, rate limiting"]

        subgraph domain["Domain services [Containers: ASP.NET Core]"]
            auth["Auth"]
            alias["Alias"]
            post["Post"]
            feed["Feed"]
            media["Media"]
            mod["AIModeration"]
            chat["ChatBox"]
            mem["UserMemory"]
            rt["RealtimeHub"]
            care["Care and commerce:<br/>Test, Wellness, Subscription,<br/>Promotion, Payment, DigitalGoods"]
        end

        subgraph shared["Shared gRPC services"]
            profile["Profile<br/>[Container]<br/>Identity mapping, persona"]
            trans["Translation<br/>[Container]"]
            notif["Notification<br/>[Container: hosted separately]"]
        end

        subgraph infra["Data stores and messaging"]
            direction LR
            mq[["RabbitMQ<br/>[Message broker]"]]
            pg[("PostgreSQL + pgvector<br/>[Database per service]")]
            scylla[("ScyllaDB<br/>[Feed read model]")]
            redis[("Redis<br/>[Cache, locks,<br/>idempotency keys]")]
        end
    end

    apps -->|"API calls and SignalR hubs<br/>[HTTPS, WebSocket]"| gw
    gw -->|"Proxies requests with scoped token<br/>[HTTP, WebSocket]"| domain
    gw -->|"Resolves alias / patient / user IDs<br/>[gRPC]"| profile

    domain -->|"Identity and persona lookups,<br/>translation, email<br/>[gRPC]"| shared
    chat <-->|"Memory search, session summaries<br/>[gRPC]"| mem
    alias -->|"Validates alias labels<br/>[gRPC]"| mod

    domain -.->|"Publishes and consumes<br/>integration events [AMQP]"| mq
    notif -.->|"Consumes email and<br/>notification events [AMQP]"| mq

    domain -->|"Reads and writes<br/>[EF Core]"| pg
    feed -->|"Reads and writes<br/>[CQL]"| scylla
    domain -->|"Caches, locks<br/>[Redis]"| redis
```

The gateway, the services in the first group, Profile and Translation are defined in `docker-compose-apis.yml`. Notification is not part of that file; Auth's compose configuration points its gRPC client at a separately hosted instance.

## Services

| Service | Responsibility | Main technologies |
|---|---|---|
| **YarpApiGateway** | Single entry point; per-service route files, JWT authentication, token exchange, fixed-window rate limiting | YARP, gRPC client, Redis |
| **Auth** | Registration, login (incl. Firebase ID-token login), JWT issuing, refresh and revocation, device sessions | ASP.NET Core Identity, PostgreSQL, Redis |
| **Profile** | Person, patient and doctor profiles, medical history; owns subject-to-alias/patient/user mappings and serves them and persona snapshots over gRPC | PostgreSQL, gRPC |
| **Alias** | Anonymous public identities, alias labels, follow relationships, audit trail | PostgreSQL, Redis, outbox |
| **Post** | Posts, nested comments, reactions, gifts and tags; moderation state | PostgreSQL, Redis locks, outbox, DDD layers |
| **Feed** | Feed read model of approved posts, follower timelines, ranking and trending | ScyllaDB, Redis |
| **AIModeration** | Moderates post content and alias labels | Gemini, PostgreSQL, gRPC |
| **Media** | Media uploads, image moderation, AI-generated reward stickers | Azure Blob Storage, Sightengine, Gemini |
| **ChatBox** | AI companion sessions: routing, tool selection, generation, rolling session summaries | Gemini, SignalR, PostgreSQL |
| **UserMemory** | Long-term memory store and semantic search; session reward points and sticker claims | PostgreSQL + pgvector, Quartz, Gemini embeddings |
| **RealtimeHub** | Pushes notifications, reward and progress updates to connected clients | SignalR, MassTransit consumers |
| **Test** | DASS-21 psychological self-assessments | PostgreSQL |
| **Wellness** | Wellness modules, challenges, mood journaling | PostgreSQL, outbox |
| **Subscription** | Service packages and user subscriptions; calls Promotion over gRPC | PostgreSQL, gRPC client |
| **Promotion** | Promo and gift codes, exposed as a gRPC service | gRPC, PostgreSQL |
| **Payment** | Payment links and webhooks for subscriptions and bookings | PayOS, VNPay |
| **DigitalGoods** | Digital goods catalogue and inventory grants | PostgreSQL, outbox |
| **Translation** | Content translation exposed over gRPC | Gemini, PostgreSQL |
| **Notification** | In-app, push and email notifications; merges repeated alerts; gRPC email API | PostgreSQL, Firebase Cloud Messaging, SMTP |

> [!NOTE]
> `Conversation`, `Scheduling`, `LifeStyles`, `Billing`, `Image` and `Wallet` also exist under `Services/`, but they are not part of `docker-compose-apis.yml` or the CI build matrix.

## Key Workflows

Each workflow starts with the design intent as reflected in the code, followed by the implemented behaviour.

### 1. Token exchange

**Intent.** Social services should be able to identify an author without being able to link that alias to a patient record. Only the gateway and Profile handle the subject-to-alias/patient mapping, so it happens once per request at the edge.

**Trade-off.** Every authenticated request depends on the gateway and on Profile for ID resolution. Two caches keep that cost off the hot path: Redis for ID lookups (1-hour sliding expiration) and an in-memory cache for minted tokens (15 minutes, the token lifetime).

```mermaid
sequenceDiagram
    autonumber
    participant App as Client app
    participant GW as API Gateway
    participant R as Redis
    participant P as Profile
    participant S as Destination service

    App->>GW: HTTPS request (Bearer client JWT)
    GW->>GW: Read subject reference, match rules for destination
    alt No rule matches the destination
        GW->>S: Forward with original token
        S-->>GW: Response
        GW-->>App: Response
    else Rules match
        loop Each required ID, in parallel, 10 s timeout
            GW->>R: Get cached ID
            R-->>GW: Hit or miss
            opt Cache miss
                GW->>P: Resolve ID by subject reference (gRPC)
                P-->>GW: ID
                GW->>R: Cache ID (1 h sliding)
            end
        end
        alt A required ID is missing
            GW-->>App: 403 PROFILE_INCOMPLETE
        else All IDs resolved
            GW->>GW: Mint RS256 token for this audience (cached 15 min)
            GW->>S: Forward with scoped token
            S-->>GW: Response
            GW-->>App: Response
        end
    end
```

### 2. AI companion: multi-stage LLM pipeline

**Intent.** A single large prompt would pay for retrieval and tool reasoning on every message. Splitting the turn into a cheap router call followed by optional steps keeps simple messages cheap and fast.

**Trade-off.** A tool or retrieval turn makes several sequential LLM and gRPC calls, so its latency is higher than a plain reply.

```mermaid
%%{init: {"flowchart": {"padding": 4, "rankSpacing": 30, "nodeSpacing": 25}}}%%
flowchart TB
    subgraph admission["Step 1 · Admission"]
        direction LR
        start(["User message"]) --> dup{"Throttled?"}
        dup -->|yes| slow(["Return:<br/>sending too fast"])
        dup -->|no| lock{"Lock<br/>acquired?"}
        lock -->|no| busy(["Error:<br/>try again later"])
        lock -->|yes| outA(["▶ Step 2<br/>Routing and events"])
    end

    subgraph routing["Step 2 · Routing and events"]
        direction LR
        inA(["From step 1<br/>Admission ▶"]) --> router["Router LLM<br/>intent + flags"]
        router --> isTool{"Tool<br/>intent?"}
        isTool -->|yes| tool["Tool-selector LLM<br/>DASS-21 or music"]
        isTool -->|no| save{"Save<br/>flag?"}
        tool --> save
        save -->|yes| pubMem["Publish<br/>UserMemoryCreated"]
        save -->|no| pubProc["Publish<br/>MessageProcessed"]
        pubMem --> pubProc
        pubMem -.-> mq[["RabbitMQ"]]
        pubProc -.-> mq
        pubProc --> outB(["▶ Step 3<br/>Context assembly"])
    end

    subgraph context["Step 3 · Context assembly"]
        direction LR
        inB(["From step 2<br/>Routing ▶"]) --> needMem{"Personal<br/>memory?"}
        needMem -->|yes| rag["SearchMemories<br/>gRPC · pgvector"]
        needMem -->|no| needKb{"Team<br/>knowledge?"}
        rag --> needKb
        needKb -->|yes| kb["Inject project<br/>knowledge file"]
        needKb -->|no| compose["Compose<br/>instruction"]
        kb --> compose
        compose --> outC(["▶ Step 4<br/>Reply"])
    end

    subgraph answer["Step 4 · Reply"]
        direction LR
        inC(["From step 3<br/>Context ▶"]) --> reply["Response LLM"]
        reply --> persist["Persist reply"]
        persist --> needSum{"14 new<br/>messages?"}
        needSum -->|yes| summarize["Summarizer LLM<br/>update summary"]
        needSum -->|no| done(["Return reply<br/>+ tool action"])
        summarize --> done
    end

    admission --> routing --> context --> answer
```

Read each step left to right. The terminal at the end of each step names the step that follows; that step starts at its "From step" terminal.

Decision conditions, as implemented in `MessageProcessor`:

| Decision | Condition |
|---|---|
| Throttled | The same text is already pending, or the session already has 2 pending messages |
| Lock acquired | The per-session lock is obtained within 15 s |
| Tool intent | Router intent is `TOOL_CALLING` |
| Save flag | Router sets `SaveNeeded` and returns a memory payload |
| Personal memory | Intent is `RAG_PERSONAL_MEMORY`, or the router sets `UsePersonalMemory` |
| Team knowledge | Intent is `RAG_TEAM_KNOWLEDGE`, or the router sets `UseTeamKnowledge` |
| 14 new messages | At least 14 messages since the session's last summary |

The router uses `gemini-2.5-flash-lite` and replies use `gemini-2.5-flash`. UserMemory consumes `UserMemoryCreated`, embeds the memory with `gemini-embedding-001` (3072 dimensions by default) and stores it with emotion, relationship and topic tags. The session lock is held in process memory, so it serialises messages per ChatBox instance.

### 3. Event-driven post moderation

**Intent.** Moderation calls an external LLM and can be slow, so it runs outside the create-post request. The outbox makes the hand-off reliable: the post and its `PostCreated` event are committed together.

**Trade-off.** Delivery is at-least-once and the post becomes public only after moderation completes, so consumers must tolerate duplicates and clients see a pending state first.

```mermaid
sequenceDiagram
    autonumber
    participant App as Client app
    participant Post as Post API
    participant DB as Post database
    participant OP as Post outbox processor
    participant MQ as RabbitMQ
    participant Mod as AIModeration
    participant Gem as Gemini API
    participant Feed

    App->>Post: Create post (via gateway)
    Post->>DB: Save post (pending moderation) + outbox row, one transaction
    Post-->>App: Post ID, moderation status Pending
    loop Every 5 s
        OP->>DB: Read up to 100 unprocessed rows
        OP-)MQ: Publish PostCreated
        OP->>DB: Mark rows processed
    end
    MQ-)Mod: PostCreated
    Mod->>Gem: Moderate title and content
    Gem-->>Mod: Verdict
    Mod-)MQ: Publish PostModerationEvaluated
    par Post updates its aggregate
        MQ-)Post: PostModerationEvaluated
        Post->>DB: Approve or reject post
    and Feed updates its read model
        MQ-)Feed: PostModerationEvaluated
        Feed->>Feed: If approved, add post to ScyllaDB replica tables
    end
```

AIModeration publishes its verdict directly rather than through an outbox.

### 4. Selective build and deployment

**Intent.** Rebuilding and restarting every container for a one-service change is slow and widens the blast radius. Both workflows therefore work from the set of changed services.

```mermaid
%%{init: {"flowchart": {"padding": 4, "rankSpacing": 30, "nodeSpacing": 25}}}%%
flowchart TB
    subgraph build["Workflow 1 · Build & Push Microservices (GHCR)"]
        direction LR
        trigger(["Push to main<br/>or manual run"]) --> manual{"Manual<br/>run?"}
        manual -->|yes| all["Select all<br/>services"]
        manual -->|no| shared{"Shared file<br/>changed?"}
        shared -->|yes| all
        shared -->|no| map["Map changed<br/>paths to services"]
        map --> any{"Any<br/>match?"}
        any -->|no| skip(["Skip build"])
        any -->|yes| runner{"Self-hosted<br/>runner free?"}
        all --> runner
        runner -->|yes| selfHosted["Build on<br/>self-hosted runner"]
        runner -->|no| hosted["Build on<br/>ubuntu-latest"]
        selfHosted --> push["Push images to GHCR<br/>SHA · main · latest"]
        hosted --> push
    end

    subgraph deploy["Workflow 2 · Deploy to VPS"]
        direction LR
        completed(["Build workflow<br/>completed"]) --> ok{"Build<br/>succeeded?"}
        ok -->|no| stop(["Skip deploy"])
        ok -->|yes| ssh["SSH to VPS: git pull,<br/>diff previous..new"]
        manualDeploy(["Manual run<br/>deploy_all, image_tag"]) --> ssh
        ssh --> force{"deploy_all?"}
        force -->|yes| every["Select all<br/>services"]
        force -->|no| sharedDeploy{"Shared file<br/>changed?"}
        sharedDeploy -->|yes| every
        sharedDeploy -->|no| changed["Map changed<br/>paths to services"]
        every --> up["Pull images from GHCR<br/>compose up -d per service"]
        changed --> up
    end

    build -.->|"workflow_run event"| deploy
```

"Shared file" means `Directory.Packages.props`, `nuget.config` or `.dockerignore` for the build workflow, and `Directory.Packages.props`, `nuget.config`, `docker-compose-apis.yml` or anything under `BuildingBlocks` for the deploy workflow. The self-hosted runner is used only when it is online and not busy. Deployments triggered by the build use the build's commit SHA as the image tag; manual runs use the `image_tag` input (default `main`).

## Technology Stack

| Area | Technologies |
|---|---|
| Backend | C#, .NET 9, ASP.NET Core, Carter (minimal APIs), MediatR (CQRS with validation, logging and idempotency behaviours), FluentValidation, EF Core, Mapster |
| Gateway & communication | YARP reverse proxy, gRPC (Protobuf), JWT (RSA keys) |
| Messaging | RabbitMQ via MassTransit, transactional outbox |
| Data | PostgreSQL (one database per service), pgvector, ScyllaDB (Feed), Redis (cache, locks, idempotency) |
| AI | Google Gemini (routing, generation, moderation, translation, image generation), Gemini embeddings |
| Real-time | SignalR (ChatBox hub, RealtimeHub) |
| Integrations | PayOS, VNPay, Firebase, Azure Blob Storage, Sightengine |
| Background jobs | Quartz.NET, hosted `BackgroundService` workers |
| Observability | Serilog, ASP.NET Core health checks (`/health` on each service) |
| Delivery | Docker, Docker Compose, GitHub Actions, GitHub Container Registry |
| Testing | xUnit, FluentAssertions, NSubstitute / Moq, AutoFixture, Testcontainers (PostgreSQL) |

## Repository Structure

```text
.
├── .github/workflows/            # Build, deploy and config-sync pipelines
└── PsychologySupport/
    ├── ApiGateways/
    │   └── YarpApiGateway/       # Gateway, token exchange, per-service route files
    ├── BuildingBlocks/
    │   ├── BuildingBlocks/       # CQRS, DDD primitives, pipeline behaviours, idempotency
    │   ├── BuildingBlocks.Messaging/  # MassTransit setup and shared integration events
    │   └── BuildingBlocks.Observability/
    ├── Services/                 # One folder per service (API or API/Application/Domain/Infrastructure)
    ├── Directory.Packages.props  # Central NuGet package versions
    ├── PsychologySupport.sln
    ├── docker-compose-apis.yml   # Gateway, services and Redis
    ├── docker-compose-databases.yml  # ScyllaDB and MongoDB
    └── docker-compose.build.yml  # Image build definitions
```

## Getting Started

> [!IMPORTANT]
> Configuration files (`appsettings*.json`) are excluded by `.gitignore` and are synced to the server by dedicated workflows. To run the system you must create them yourself; the repository does not ship working configuration or credentials.

### Prerequisites

- .NET 9 SDK
- Docker with Docker Compose
- External dependencies not defined in the compose files: **PostgreSQL** (with the `pgvector` extension for UserMemory) and **RabbitMQ**
- Accounts / keys for the integrations you want to use (Gemini, PayOS, VNPay, Firebase, Azure Blob Storage, Sightengine)

### Configuration

Each service reads standard ASP.NET Core configuration. Keys referenced in code include:

- `ConnectionStrings:<Service>Db` (e.g. `PostDb`, `UserMemoryDb`, `AuthDb`) and `ConnectionStrings:Redis`
- `MessageBroker:Host`, `MessageBroker:UserName`, `MessageBroker:Password`
- `GrpcSettings:*` (URLs of gRPC dependencies, e.g. `GrpcSettings:PiiUrl`)
- `Jwt:*`, `GeminiConfig:*`, `PayOS:*`, `Vnpay:*`, `Firebase:*`, `AzureBlobStorage:*`, `Sightengine:*`

The compose file also mounts an HTTPS development certificate from `~/.aspnet/https` and RSA keys from each service's `RSAKeys/` folder.

### Run with Docker Compose

```bash
cd PsychologySupport

# Shared network used by all compose files
docker network create emoease-net

# ScyllaDB (Feed) and MongoDB; requires VPS_PUBLIC_IP in the environment
docker compose -f docker-compose-databases.yml up -d

# Build images (DOCKER_REGISTRY and IMAGE_TAG come from your .env file)
docker compose -f docker-compose.build.yml build

# Start the gateway, services and Redis
docker compose --env-file .env -f docker-compose-apis.yml up -d
```

The gateway listens on port `8080`. Each service exposes `/health`, and the gateway proxies it as `/<service>-service/health` (for example `/aimoderation-service/health`).

### Run a single service locally

```bash
cd PsychologySupport
dotnet run --project Services/Post/Post.API
```

EF Core migrations live in each service's `Migrations` folders and are applied with the standard `dotnet ef database update` workflow.

## Testing

Automated tests currently cover the **Post** service:

| Project | Scope |
|---|---|
| `Post.Tests.Domain.Unit` | Aggregate and value-object rules |
| `Post.Tests.Application.Unit` | Command handlers |
| `Post.Tests.Integration` | Post lifecycle flows, including outbox writes |
| `Post.Tests.Contract` | Integration event contracts |

```bash
cd PsychologySupport
dotnet test Services/Post/Post.Domain.UnitTests/Post.Tests.Domain.Unit.csproj
dotnet test Services/Post/Post.Application.UnitTests/Post.Tests.Application.Unit.csproj
```

The test projects are not part of `PsychologySupport.sln`, so they are run by project path. Integration and contract tests use Testcontainers and require a running Docker daemon.

## Architecture Documentation

For service-by-service walkthroughs, data models and additional diagrams, see the **[DeepWiki documentation](https://deepwiki.com/anht04/psychology-support-microservices)**. Where DeepWiki and the source code differ, the source code is authoritative.
