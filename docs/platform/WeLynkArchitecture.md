# The Cortex Architecture

## WeLynk Platform Technical Documentation

**Version:** 2.0 (Public Edition)
**Classification:** Public Technical Documentation
**Audience:** Engineers, Researchers, and Technical Community

> **Note:** This is a public version of our architecture documentation. Security-sensitive implementation details, fraud prevention thresholds, and infrastructure specifics have been redacted. The document focuses on architectural concepts and design philosophy.

---

## Preface

This document describes the Cortex Architecture - the foundational design powering WeLynk, a social connection platform serving users aged 16 and above. The architecture is named for its resemblance to the cerebral cortex: a central kernel orchestrating specialized agents, each responsible for distinct cognitive functions, communicating through well-defined pathways.

This is not implementation documentation. You will find no code here. Instead, this document explains the conceptual foundations, mathematical models, physical simulations, and design philosophies that make WeLynk function. An engineer from any major technology company should be able to understand exactly how this system operates after reading this document.

---

## Table of Contents

1. [Philosophy and Design Principles](#1-philosophy-and-design-principles)
2. [The Kernel and Agent Model](#2-the-kernel-and-agent-model)
3. [Layer Hierarchy and Information Flow](#3-layer-hierarchy-and-information-flow)
4. [The Lynks Economy](#4-the-lynks-economy)
5. [ALMA: Adaptive Learning Matching Algorithm](#5-alma-adaptive-learning-matching-algorithm)
6. [Game Runtime Architecture](#6-game-runtime-architecture)
7. [Game Physics and Client Interpolation](#7-game-physics-and-client-interpolation)
8. [Direct Connection Architecture](#8-direct-connection-architecture)
9. [Game Security and Anti-Abuse](#9-game-security-and-anti-abuse)
10. [Audio Control Enforcement](#10-audio-control-enforcement)
11. [Authentication and Session Management](#11-authentication-and-session-management)
12. [Real-Time Messaging](#12-real-time-messaging)
13. [Presence System](#13-presence-system)
14. [Voice and Video Calls](#14-voice-and-video-calls)
15. [The Store and Item Rendering](#15-the-store-and-item-rendering)
16. [Creator Studio](#16-creator-studio)
17. [User Safety Systems](#17-user-safety-systems)
18. [Security Architecture](#18-security-architecture)
19. [Analytics Engine](#19-analytics-engine)
20. [Infrastructure and Distribution](#20-infrastructure-and-distribution)
21. [Appendix: Mathematical Foundations](#appendix-mathematical-foundations)

---

## 1. Philosophy and Design Principles

### 1.1 The Priority Hierarchy

Every engineering decision in WeLynk follows a strict priority order:

```
Security > Correctness > Architecture > Maintainability > Performance
```

This ordering is absolute. A fast but insecure system is worthless. A performant but incorrect system causes harm. A quick hack that violates architecture creates technical debt that compounds. Only after security, correctness, and architectural integrity are satisfied do we consider maintainability, and only then performance.

### 1.2 The Distributed-First Mindset

WeLynk operates on a fundamental assumption: **there is no single server**. Every design decision assumes multiple workers processing requests simultaneously, with no shared memory between them.

This mental model prevents an entire class of bugs. When an engineer thinks "I'll store this in a dictionary for quick lookup," the distributed-first mindset immediately triggers: "Which worker's dictionary? What happens when the next request hits a different worker?"

The consequence: all shared state lives in Redis. All counters use atomic Lua scripts. All rate limits are distributed. There are no exceptions.

### 1.3 The Trust Boundary Model

WeLynk defines explicit trust boundaries:

```
UNTRUSTED                    TRUST BOUNDARY                    TRUSTED
    │                              │                              │
    │  ┌─────────────────┐         │         ┌─────────────────┐  │
    │  │  User Input     │─────────┼────────▶│  Validated Data │  │
    │  │  External APIs  │         │         │  Internal State │  │
    │  │  Game Code      │         │         │  Agent Results  │  │
    │  │  Uploaded Media │         │         │  Database       │  │
    │  └─────────────────┘         │         └─────────────────┘  │
    │                              │                              │
```

Everything crossing the trust boundary undergoes validation. User input is sanitized. External API responses are verified. Game code runs in isolation. Uploaded media is scanned. Only after crossing this boundary does data become trusted for internal operations.

### 1.4 Defense in Depth

No single security measure is sufficient. WeLynk implements security at every layer:

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1: Network                                               │
│  TLS 1.3, WAF rules, DDoS protection, IP reputation             │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2: Gateway                                               │
│  Authentication, rate limiting, input validation                │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3: Application                                           │
│  Authorization, business rule validation, safety checks         │
├─────────────────────────────────────────────────────────────────┤
│  Layer 4: Data                                                  │
│  Parameterized queries, encryption at rest, audit logging       │
└─────────────────────────────────────────────────────────────────┘
```

An attacker must breach all layers to cause harm. Compromising one layer finds another waiting.

---

## 2. The Kernel and Agent Model

### 2.1 Conceptual Overview

The Cortex Architecture centers on a **SystemKernel** that orchestrates **Agents**. This design mirrors how an operating system kernel manages hardware through device drivers, or how the brain's cortex coordinates specialized regions.

The SystemKernel is an **internal infrastructure coordinator** that is NEVER imported by agents, domain APIs, or backends directly. It manages:
- Thread pools and async task coordination
- Intelligent request batching (BatchCollector)
- Circuit breaking for failing systems
- Performance monitoring (LoadMetricsTracker)
- Dynamic connection management and load balancing
- Background tasks (cache warming, cleanup)
- Memory optimization

```
                         ┌─────────────────────┐
                         │    SystemKernel     │
                         │   (Internal Only)   │
                         │                     │
                         │  Thread Pools       │
                         │  Circuit Breakers   │
                         │  Load Metrics       │
                         │  Memory Optimizer   │
                         └──────────┬──────────┘
                                    │
                         ┌──────────┴──────────┐
                         │   Agent Registry    │
                         │  (Module-level vars)│
                         │                     │
                         │  _database_api      │
                         │  _cache_api         │
                         │  _safety_engine     │
                         │  _matching_engine   │
                         │  ...                │
                         └──────────┬──────────┘
                                    │
           ┌────────────────────────┼────────────────────────┐
           │                        │                        │
           ▼                        ▼                        ▼
    ┌─────────────┐          ┌─────────────┐          ┌─────────────┐
    │  Database   │          │    Cache    │          │   Safety    │
    │    Agent    │          │    Agent    │          │    Agent    │
    └─────────────┘          └─────────────┘          └─────────────┘
           │                        │                        │
           ▼                        ▼                        ▼
    ┌─────────────┐          ┌─────────────┐          ┌─────────────┐
    │ PostgreSQL  │          │    Redis    │          │  ML Models  │
    └─────────────┘          └─────────────┘          └─────────────┘
```

### 2.2 The Agent Contract

Each Agent adheres to a strict contract:

1. **Single Responsibility**: An agent handles exactly one infrastructure concern
2. **Stateless Operation**: Agents hold no mutable state between requests
3. **Isolation**: Agents never import or call other agents directly
4. **Dependency Injection**: All dependencies arrive through the constructor
5. **Async by Default**: All operations are asynchronous

The agents in the system are:

| Agent | Responsibility |
|-------|----------------|
| DatabaseAgent | PostgreSQL persistence operations |
| CacheAgent | Redis caching and distributed state |
| SafetyAgent | Content moderation and minor protection |
| MatchingAgent | ALMA user matching algorithm |
| CallAgent | LiveKit voice/video coordination |
| AuthorizationAgent | Role and permission management |
| AnalyticsAgent | Metrics collection and aggregation |

### 2.3 Initialization Lifecycle

The system initializes components in strict phases, with role-based loading (user backend vs admin backend load different agents):

```
┌─────────────────────────────────────────────────────────────────┐
│  STARTUP                                                        │
│                                                                 │
│  Load environment + config                                      │
│  Create SystemKernel (internal infrastructure)                  │
│  Create RedisAsyncClient                                        │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  PHASE 1: Infrastructure Agents (No Dependencies)               │
│                                                                 │
│  DatabaseAPI ────────────▶ Connection pool, repositories        │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  PHASE 2: Low-Level Agents                                      │
│                                                                 │
│  CacheAPI ───────────────▶ Redis cluster connection             │
│  SessionAPI ─────────────▶ WebSocket session tracking           │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  PHASE 2.5: Domain Managers (Depend on Infrastructure)          │
│                                                                 │
│  UserManager(db, cache, session)                                │
│  ChatManager(db, cache)                                         │
│  AuthManager(db, session)                                       │
│  ProfileManager(db, cache)                                      │
│  ... other managers                                             │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  PHASE 3: Domain APIs (Depend on Managers)                      │
│                                                                 │
│  UserAPI(user_manager, profile_manager)                         │
│  ChatAPI(chat_manager)                                          │
│  AuthAPI(auth_manager)                                          │
│  ... other APIs                                                 │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  PHASE 4: Business Agents (User Backend Only)                   │
│                                                                 │
│  IF role == "user":                                             │
│    AnalysisEngine(user_api)                                     │
│    MatchingEngine(analysis, user_api, safety_api)               │
│    CallEngine(calls_api, user_api)                              │
│    GameStudioAPI(games_api, user_api)                           │
│    TypeScriptGameAPI(runtime_bridge)                            │
│                                                                 │
│  IF role == "admin":                                            │
│    Only authorization + safety (no matching, calls, games)      │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  PHASE 5: Late Binding (Circular Dependencies)                  │
│                                                                 │
│  user_api.set_safety_engine(safety_engine)                      │
│  safety_api.set_safety_engine(safety_engine)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Each phase completes before the next begins. If any agent fails initialization, the system halts rather than operating in a degraded state. This fail-fast behavior ensures problems are detected at startup, not runtime.

### 2.4 The Registry Pattern

The kernel maintains a registry of initialized agents. When a Domain API or Manager needs an agent, it requests it from the registry rather than creating it:

```
┌────────────────┐     ┌──────────────┐     ┌─────────────────┐
│   Manager      │────▶│   Registry   │────▶│  Agent Instance │
│   needs cache  │     │   lookup     │     │  (singleton)    │
└────────────────┘     └──────────────┘     └─────────────────┘
```

This ensures:
- Only one instance of each agent exists per worker
- Agents share connection pools efficiently
- Dependencies are explicit and traceable
- Testing can substitute mock agents easily

---

## 3. Layer Hierarchy and Information Flow

### 3.1 The Five Layers

Information flows through WeLynk in a strictly defined hierarchy:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  LAYER 1: GATEWAY                                               │
│  ─────────────────                                              │
│  Entry point for all external requests                          │
│                                                                 │
│  Responsibilities:                                              │
│  • HTTP route handling                                          │
│  • WebSocket connection management                              │
│  • Request validation (schema enforcement)                      │
│  • Authentication (JWT verification)                            │
│  • Rate limiting enforcement                                    │
│                                                                 │
│  Examples: user_backend routers, admin_backend routers          │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                              │                                  │
│                              ▼                                  │
│  LAYER 2: DOMAIN API                                            │
│  ───────────────────                                            │
│  Public interface for each business domain                      │
│                                                                 │
│  Responsibilities:                                              │
│  • Define the contract for domain operations                    │
│  • Delegate to appropriate managers                             │
│  • Aggregate results from multiple managers                     │
│  • Transform data for gateway consumption                       │
│                                                                 │
│  Critical Rule: NO BUSINESS LOGIC                               │
│  Domain APIs delegate only; they contain no decisions           │
│                                                                 │
│  Examples: UserAPI, ChatAPI, GamesAPI, SafetyAPI                │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                              │                                  │
│                              ▼                                  │
│  LAYER 3: MANAGER                                               │
│  ───────────────                                                │
│  Business logic implementation                                  │
│                                                                 │
│  Responsibilities:                                              │
│  • Implement business rules and decisions                       │
│  • Orchestrate multiple services/operations                     │
│  • Handle domain-specific validation                            │
│  • Manage transactions and consistency                          │
│                                                                 │
│  Examples: UserManager, GameLobbyManager, ChatManager           │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                              │                                  │
│                              ▼                                  │
│  LAYER 4: SERVICE / OPERATION                                   │
│  ────────────────────────────                                   │
│  Reusable logic units                                           │
│                                                                 │
│  Services: Stateless utilities (hashing, validation, etc.)      │
│  Operations: Single-purpose workflows (user creation, etc.)     │
│                                                                 │
│  Examples: ProfilePictureService, UserCreationOperation         │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                              │                                  │
│                              ▼                                  │
│  LAYER 5: AGENT                                                 │
│  ──────────────                                                 │
│  Infrastructure abstraction                                     │
│                                                                 │
│  Responsibilities:                                              │
│  • Abstract infrastructure details                              │
│  • Manage connections and pooling                               │
│  • Translate domain requests to infrastructure calls            │
│                                                                 │
│  Examples: DatabaseAgent, CacheAgent, SafetyAgent               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 The Forbidden Paths

Certain call patterns are strictly forbidden and enforced by `lint_architecture.py` (56+ rules):

```
FORBIDDEN PATTERNS
══════════════════

1. Gateway ──X──▶ Manager (skip Domain API)
   WHY: Domain API provides the contract; skipping it couples
        gateway directly to implementation details
   LINT CHECK: #21 - Domain API receives DatabaseAPI directly

2. Gateway ──X──▶ Agent (skip all layers)
   WHY: Business logic would leak into gateway;
        security checks would be bypassed
   LINT CHECK: #50 - Agent runtime imports from system_core

3. Domain API contains business logic
   WHY: Domain APIs become untestable monoliths;
        managers cannot be reused
   LINT CHECK: #24 - API method calls other API methods
   LINT CHECK: #47 - API method calls multiple managers

4. Agent ──X──▶ Agent (runtime import)
   WHY: Creates hidden dependencies; breaks isolation;
        makes testing and mocking difficult
   LINT CHECK: #51 - Business agent imports another business agent

5. Service ──X──▶ Manager (upward call)
   WHY: Creates circular dependencies; violates hierarchy;
        makes reasoning about data flow impossible
   LINT CHECK: #30 - Manager location not in managers/ subdirectory


IMPORT RULES BY LAYER
═════════════════════

┌────────────────────────────────────────────────────────────────┐
│  Layer       │  CAN Import           │  CANNOT Import          │
├────────────────────────────────────────────────────────────────┤
│  Gateway     │  shared/              │  database_system        │
│              │  TYPE_CHECKING APIs   │  agents directly        │
│              │                       │  other gateways         │
├────────────────────────────────────────────────────────────────┤
│  Domain API  │  own managers         │  business logic         │
│              │  shared/              │  other domains          │
│              │  TYPE_CHECKING agents │  system_core internals  │
├────────────────────────────────────────────────────────────────┤
│  Manager     │  own services/ops     │  direct agent calls     │
│              │  shared/              │  other domains          │
│              │  TYPE_CHECKING agents │  HTTP concerns          │
├────────────────────────────────────────────────────────────────┤
│  Service     │  agents (via inject)  │  managers (circular)    │
│              │  shared/              │  other services         │
│              │  TYPE_CHECKING APIs   │  other domains          │
├────────────────────────────────────────────────────────────────┤
│  Agent       │  own files            │  system_core.main       │
│              │  shared/              │  other agents (runtime) │
│              │  TYPE_CHECKING APIs   │  gateways               │
└────────────────────────────────────────────────────────────────┘


TYPE_CHECKING PATTERN (Critical for Avoiding Circular Imports)
══════════════════════════════════════════════════════════════

CORRECT - Forward reference with TYPE_CHECKING:
  from typing import TYPE_CHECKING
  if TYPE_CHECKING:
      from backend.system_core.api.user_api import UserAPI

  class SafetyPolicy:
      def __init__(self, user_api: "UserAPI"):  # Quoted = forward ref
          self.user_api = user_api

WRONG - Runtime import creates circular dependency:
  from backend.system_core.api.user_api import UserAPI  # Runtime!

  class SafetyPolicy:
      def __init__(self):
          self.user_api = UserAPI()  # Instantiation outside main.py
```

### 3.3 Request Flow Example

Consider a user updating their profile picture:

```
┌─────────────────────────────────────────────────────────────────┐
│  Step 1: HTTP Request Arrives                                   │
│                                                                 │
│  POST /api/v1/users/me/profile-picture                          │
│  Authorization: Bearer <jwt_token>                              │
│  Content-Type: multipart/form-data                              │
│  Body: [image data]                                             │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 2: Gateway Layer (user_profile_router)                    │
│                                                                 │
│  • Extract JWT from Authorization header                        │
│  • Verify signature, check expiration                           │
│  • Extract user_id from token claims                            │
│  • Validate file type (must be image)                           │
│  • Check file size (within configured limit)                    │
│  • Check rate limit (within configured limit)                   │
│  • Call Domain API                                              │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 3: Domain API Layer (UserAPI)                             │
│                                                                 │
│  • Receive validated request                                    │
│  • Delegate to ProfileManager.update_picture()                  │
│  • Return result to gateway                                     │
│                                                                 │
│  Note: No logic here - pure delegation                          │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 4: Manager Layer (ProfileManager)                         │
│                                                                 │
│  • Check user exists and is active                              │
│  • Call SafetyAgent to scan image for prohibited content        │
│  • If CSAM detected: block, report to NCMEC, suspend user       │
│  • Call ProfilePictureService to process image                  │
│  • Call DatabaseAgent to update user record                     │
│  • Call CacheAgent to invalidate cached profile                 │
│  • Return new profile picture URL                               │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 5: Service Layer (ProfilePictureService)                  │
│                                                                 │
│  • Resize image to standard dimensions                          │
│  • Generate thumbnail                                           │
│  • Compress to AVIF format                                      │
│  • Upload to S3 bucket                                          │
│  • Return CDN URL                                               │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 6: Agent Layer                                            │
│                                                                 │
│  SafetyAgent:                                                   │
│  • Send image to PhotoDNA service                               │
│  • Compare hash against known CSAM database                     │
│  • Return safety verdict                                        │
│                                                                 │
│  DatabaseAgent:                                                 │
│  • Execute UPDATE query with parameterized values               │
│  • Return affected row count                                    │
│                                                                 │
│  CacheAgent:                                                    │
│  • Delete user profile cache                                    │
│  • Publish invalidation event                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. The Lynks Economy

### 4.1 Currency Overview

Lynks is the virtual currency of WeLynk. Unlike cryptocurrencies or complex token systems, Lynks follows a straightforward model designed for simplicity and fraud resistance.

**Exchange Rate:**
```
[Exchange rate redacted - proprietary]
```

This rate is fixed and non-negotiable. There is no market, no trading between users, and no speculation. Lynks are purchased, earned, or granted - and spent.

### 4.2 Earning Methods

Users acquire Lynks through four channels:

```
┌─────────────────────────────────────────────────────────────────┐
│  METHOD 1: In-App Purchase (IAP)                                │
│                                                                 │
│  Direct purchase through iOS App Store and Google Play          │
│                                                                 │
│  [Specific pricing tiers redacted - proprietary]                │
│                                                                 │
│  Features:                                                      │
│    • Multiple tier options with volume bonuses                  │
│    • Special starter pack for new users                         │
│                                                                 │
│  Security: Server-side receipt verification with Apple/Google   │
│  Idempotency via transaction ID prevents duplicate grants       │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  METHOD 2: Subscription Daily Grant                             │
│                                                                 │
│  Premium subscribers receive automatic daily grants             │
│                                                                 │
│  Grant: Daily Lynks (transaction type: subscription_daily)      │
│  Requirement: Must open app to claim (anti-hoarding)            │
│  [Specific amounts redacted - proprietary]                      │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  METHOD 3: Referral Program                                     │
│                                                                 │
│  Reward for bringing verified new users                         │
│                                                                 │
│  Reward Structure:                                              │
│    • Referred user: Lynks upon completing quests                │
│    • Referrer: Lynks (if phone verified)                        │
│    • If referrer phone not verified: reward status "pending"    │
│    [Specific amounts redacted - proprietary]                    │
│                                                                 │
│  Requirements:                                                  │
│    • Referred user must complete onboarding quests              │
│    • Lifetime cap on referral rewards per referrer              │
│                                                                 │
│  [Specific quest details redacted - proprietary]                │
│                                                                 │
│  See Section 4.4 for fraud prevention details                   │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  METHOD 4: Promotional Grants                                   │
│                                                                 │
│  Special events, competitions, and milestones                   │
│  Admin-issued grants for special circumstances                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 Spending Mechanisms

Lynks are spent exclusively within the WeLynk ecosystem:

```
┌─────────────────────────────────────────────────────────────────┐
│  STORE ITEMS                                                    │
│                                                                 │
│  Categories: Profile Frames, Message Frames, Backgrounds,       │
│              Sticker Packs, Custom Emojis                       │
│                                                                 │
│  [Specific price ranges redacted - proprietary]                 │
│                                                                 │
│  Pricing tiers:                                                 │
│    • Regular price: price_lynks (all users)                     │
│    • Subscriber price: subscriber_price_lynks (discount)        │
│    • Premium items: is_premium=true requires subscription       │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  GIFTING                                                        │
│                                                                 │
│  Users can gift items to others                                 │
│  The purchaser pays full Lynks price                            │
│  Recipient receives item in inventory                           │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  GAME SPENDING                                                  │
│                                                                 │
│  In-game purchases via Games SDK                                │
│  Transaction type: game_spend                                   │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  CREATOR PURCHASES                                              │
│                                                                 │
│  Purchasing user-generated store items                          │
│  Revenue split between creator and platform                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘


SUBSCRIPTION / PREMIUM
══════════════════════

Single tier subscription model (iOS App Store, Google Play):

Subscription lifecycle:
  1. User subscribes via app store
  2. Webhook received (Apple/Google)
  3. activate_subscription() updates user
  4. Welcome email sent
  5. is_subscriber = true

Renewal:
  • Auto-renewal webhook triggers activation
  • Renewal email instead of welcome

Cancellation:
  • User cancels in app store
  • cancel_subscription() called
  • Grace period: access until subscription_expires_at
  • After expiry: premium features removed

Premium-exclusive features:
  ┌───────────────────────────────────────────────────────────┐
  │  Feature                  │  Non-subscriber behavior      │
  ├───────────────────────────────────────────────────────────┤
  │  Animated profile picture │  Static images only           │
  │  Name style/text effects  │  Default styling              │
  │  Premium store items      │  Cannot purchase              │
  │  Subscriber discounts     │  Pay full price               │
  │  Daily Lynks grant        │  No daily grant               │
  └───────────────────────────────────────────────────────────┘

When subscription expires:
  • Animated PFP converted to static (or removed)
  • Name style reset to NULL
  • Premium items remain in inventory (already purchased)
  • Cannot purchase new premium items
```

### 4.4 Fraud Prevention

The Lynks system implements multiple fraud prevention mechanisms using Redis-backed atomic operations with Lua scripts (critical for distributed multi-worker architecture).

```
FRAUD PREVENTION PHILOSOPHY
═══════════════════════════

Design Principles:
  • Fail-closed: If fraud check unavailable → REJECT transaction
  • Atomic operations: All checks use Lua scripts for race-condition safety
  • Multi-signal detection: No single indicator triggers action alone
  • Graduated response: Severity levels determine enforcement action

Detection Categories:
  • Transaction velocity limiting (unusual spending patterns)
  • Referral ring detection (circular referral schemes)
  • Device/IP velocity limiting (multi-account abuse)
  • Behavioral anomaly detection (bot-like patterns)

Enforcement:
  • Low severity: Flag for review, allow transaction
  • Medium severity: Flag + temporary restrictions
  • High severity: Block transaction + account review
  • Critical severity: Immediate account suspension

All fraud signals are logged with severity, metadata, and timestamps
for pattern analysis and manual review queues.

[Specific thresholds and detection patterns redacted for security]
```

---

## 5. ALMA: Adaptive Learning Matching Algorithm

### 5.1 Historical Context

ALMA began as an ambitious machine learning project. The original system:

- Trained on conversation datasets from Reddit, Omegle, and research corpora
- Used transformer-based embeddings to understand conversation quality
- Predicted match success probability with unprecedented accuracy
- **Cost: [redacted]** in compute at production scale

The high cost was unsustainable. ALMA was completely redesigned as a **pure behavioral matching system** - no ML, no embeddings, no interests. The algorithm matches users based on observable behavior patterns, achieving comparable results at negligible cost.

### 5.2 The Four-Signal Behavioral Model

ALMA uses four behavioral signals with fixed weights that sum to 1.0:

```
SIGNAL WEIGHTS
══════════════

┌───────────────────────────────────────────────────────────────┐
│  Signal               │  Weight  │  What It Measures          │
├───────────────────────────────────────────────────────────────┤
│  Depth Match          │   w₁     │  Message length similarity │
│  Reliability Pairing  │   w₂     │  Ghost/show-up patterns    │
│  Age Match            │   w₃     │  Age proximity (safety)    │
│  Experience Match     │   w₄     │  Session count proximity   │
└───────────────────────────────────────────────────────────────┘
[Specific weights redacted - proprietary algorithm]

Total behavioral score:
S(u₁, u₂) = w₁×depth + w₂×reliability + w₃×age + w₄×experience
```

### 5.3 Signal Formulas

**Depth Match (Message Length Similarity)**

Pairs users who communicate at similar depth levels:

```
Let len_a, len_b = message_len_ewma for each user

             max(len_a, len_b)
ratio = ─────────────────────────
             min(len_a, len_b)

                        1.0
depth_score = ───────────────────────────
               1.0 + k × ln(ratio)
[Coefficient k and example outputs redacted - proprietary algorithm]

Principle:
  • Identical lengths → perfect match
  • Higher ratio → lower compatibility
  • One is 0, other >0 → mismatch penalty [value redacted]
  • Both are 0         → high score (both minimal writers)
```

**Reliability Pairing (Behavioral Patterns)**

Matches users with similar reliability profiles:

```
reliability_score = 1 - (w₁×show_up_diff + w₂×ghost_diff + w₃×early_exit_diff)
                        [Specific weights redacted - proprietary algorithm]

where:
  show_up_diff    = |u₁.show_up_rate - u₂.show_up_rate|
  ghost_diff      = |u₁.ghost_rate - u₂.ghost_rate|
  early_exit_diff = |u₁.early_exit_rate - u₂.early_exit_rate|

Range: [0.0, 1.0]
  • 1.0 = Identical reliability profiles
  • 0.0 = Completely mismatched (one reliable, one unreliable)
```

**Age Match (Gaussian Decay)**

Different decay rates for minors (stricter) vs adults (looser):

```
                    ⎧  exp(-age_diff² / σ²_minor)   if either user is minor
age_score(u₁, u₂) = ⎨
                    ⎩  exp(-age_diff² / σ²_adult)   if both are adults

[Specific decay parameters and example outputs redacted - proprietary algorithm]

Key principle:
  • Minors use stricter decay (smaller age differences penalized more heavily)
  • Adults use looser decay (larger age differences tolerated)
  • Score decays exponentially with age difference
```

**Experience Match (Session Count Proximity)**

Pairs newcomers with newcomers, veterans with veterans:

```
                                    |sessions_a - sessions_b|
experience_score = 1 - min(1.0, ────────────────────────────────)
                                            N
[Denominator N redacted - proprietary algorithm]

Examples:
  • Identical session counts  → 1.0 (perfect match)
  • Moderate gap              → 0.5 (partial match)
  • Large gap                 → 0.0 (veteran vs newcomer)
[Specific session thresholds redacted - proprietary algorithm]
```

### 5.4 Confidence Blending

New users lack behavioral data. ALMA smoothly transitions from demographic to behavioral matching:

```
CONFIDENCE CALCULATION
══════════════════════

                sessions_count
confidence = min(1.0, ─────────────────)
                           N

where N = sessions required for full confidence [threshold redacted]

BLENDED SCORE
═════════════

final_score = (1 - confidence) × demographic_score + confidence × behavioral_score

where:
  demographic_score = age_match (0-1.0)
  behavioral_score  = weighted sum of depth + reliability + experience

EFFECT ON MATCHING
══════════════════

Sessions | Confidence | Weight Split
─────────┼────────────┼────────────────────────
    0    │     0%     │ 100% demographic (age)
  few    │    low     │  Mostly demographic, some behavioral
  some   │    med     │  Balanced mix
  many   │   100%     │ 100% behavioral (age ignored)
[Specific session thresholds redacted - proprietary algorithm]
```

### 5.5 EWMA: Exponentially Weighted Moving Average

All behavioral metrics use EWMA for recency weighting:

```
EWMA FORMULA
════════════

new_ewma = α × current_ewma + (1-α) × new_value

where α = decay factor [value redacted]

This means:
  • Higher weight on new observations vs historical average
  • Older observations decay exponentially
  • Recent behavior matters more than distant past

TRACKED METRICS
═══════════════

┌─────────────────────────────────────────────────────────────────┐
│  Metric                │  Cap        │  Purpose                 │
├─────────────────────────────────────────────────────────────────┤
│  message_len_ewma      │  [capped]   │  Depth indicator         │
│  response_gap_ewma_ms  │  [capped]   │  Conversation pace       │
│  ghost_rate            │  [0, 1]     │  Reliability signal      │
│  show_up_rate          │  [0, 1]     │  Session attendance      │
│  early_exit_rate       │  [0, 1]     │  Commitment level        │
└─────────────────────────────────────────────────────────────────┘
```

### 5.6 Queue Architecture

ALMA uses a database-backed queue with atomic matching:

```
QUEUE OPERATIONS (Atomic SQL)
═════════════════════════════

1. ENTER QUEUE
   INSERT user with: age, is_minor, behavior_score, entered_at
   ON CONFLICT DO NOTHING (prevent duplicates)

2. FIND BEST MATCH (Single Transaction)

   a) Build exclusion set for seeker:
      - Blocked relationships (bidirectional)
      - Friend relationships
      - Report relationships (bidirectional)
      - Previous match history
      - Legacy chat partners

   b) Find candidate:
      SELECT * FROM queue
      WHERE NOT excluded
        AND age_constraint_satisfied
      ORDER BY
        ABS(candidate.behavior_score - seeker.behavior_score) ASC,  -- PRIMARY
        candidate.entered_at ASC  -- SECONDARY (FIFO fairness)
      FOR UPDATE SKIP LOCKED  -- Atomic claim
      LIMIT 1

   c) Atomic state update:
      UPDATE both users SET status = 'matched'
      (Single transaction with row-level locking)

3. COMPLETE MATCH
   Record in match_history (prevents re-matching)
   Remove both from queue

FAIRNESS GUARANTEES
═══════════════════

• Behavioral similarity prioritized (closest scores)
• FIFO secondary sort (longer wait = higher priority when scores similar)
• Bidirectional exclusion validation
• Match history prevents repeated pairings
• Distributed semaphore limits concurrent matches
```

### 5.7 Minor Protection (Hard Enforcement)

Minor safety is enforced at the SQL level - not application logic:

```
ABSOLUTE RULE: Minors match ONLY with other minors

SQL CONSTRAINT
══════════════

WHERE (
  -- Adults can only match other adults
  (NOT seeker.is_minor AND NOT candidate.is_minor)

  -- Minors can ONLY match other minors
  OR (seeker.is_minor AND candidate.is_minor)
)

CONCRETE EXAMPLES
═════════════════

Age 16 (minor) + Age 17 (minor)  →  ✓ ALLOWED
Age 16 (minor) + Age 18 (adult)  →  ✗ BLOCKED (even though 18 is young)
Age 17 (minor) + Age 25 (adult)  →  ✗ BLOCKED
Age 18 (adult) + Age 24 (adult)  →  ✓ ALLOWED
Age 18 (adult) + Age 16 (minor)  →  ✗ BLOCKED

There is NO age gap tolerance for minor-adult matching.
The boundary at 18 is absolute.

VALIDATION AT QUEUE ENTRY
═════════════════════════

is_minor = (age < 18)

VALIDATE: is_minor == (age < 18)
If mismatch → InvalidParametersError (prevents data corruption)
```

### 5.8 New User Affinity System

Experienced users are scored on how well they treat newcomers:

```
NEW USER DEFINITION
═══════════════════

A user is "new" if sessions_count < threshold [threshold redacted]

AFFINITY SCORE (for experienced users)
══════════════════════════════════════

affinity = w₁×depth_rate + w₂×(1 - ghost_rate) + w₃×friend_rate
[Specific weights redacted - proprietary algorithm]

where (tracking interactions with new users only):
  depth_rate   = % of deep conversations with newcomers
  ghost_rate   = % of sessions where user ghosted newcomers
  friend_rate  = % of new users they friended

AFFINITY TIERS
══════════════

  High score = Excellent newcomer mentor
  Medium score = Acceptable for new users
  Low score = Avoid matching with newcomers
[Specific tier thresholds redacted - proprietary algorithm]

APPLICATION
═══════════

When confidence blending for a new user:
  demographic_score = w₁×age_match + w₂×new_user_affinity [weights redacted]
```

### 5.9 Bot Fallback

When no human match is found:

```
BOT MATCHING RULES
══════════════════

• Bots do NOT use behavioral scoring (random selection)
• Bots must match user's age + is_minor constraints
• Bot matching only triggers after human_first_wait timeout
• User remains in queue after bot match (can still match humans)
• Bots cannot initiate searches (security check)
```

---

## 6. Game Runtime Architecture

### 6.1 Design Philosophy

Games in WeLynk run in a separate runtime for critical reasons:

1. **Security Isolation**: Game code cannot access databases or user data directly
2. **Language Optimization**: TypeScript/Node.js excels at real-time event processing
3. **Failure Isolation**: A crashing game cannot bring down the platform
4. **Resource Control**: Games have strict CPU/memory/network limits
5. **Latency Optimization**: Direct browser-to-runtime connections bypass Python

### 6.2 Architecture Overview

**Critical Design Decision**: State updates and actions flow DIRECTLY between browser and runtime, bypassing Python entirely. Python only handles session lifecycle (creation, termination) and platform events (lobby changes).

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT BROWSER                          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Game Bundle (React + TypeScript)                         │  │
│  │  • Renders game UI at 60fps                               │  │
│  │  • Uses entity interpolation (NOT prediction)             │  │
│  │  • Receives 30Hz state snapshots                          │  │
│  │  • Sends actions directly to runtime                      │  │
│  └───────────────────────────────────────────────────────────┘  │
└────────────────────┬──────────────────────┬─────────────────────┘
                     │                      │
    Session creation │                      │ Direct game traffic
    (one-time)       │                      │ (all gameplay)
                     │                      │
                     ▼                      │
┌─────────────────────────────────────────┐ │
│            USER_BACKEND (Python)        │ │
│  ┌───────────────────────────────────┐  │ │
│  │  Game Gateway                     │  │ │
│  │  • Creates sessions (POST /api)   │  │ │
│  │  • Returns signed game token      │  │ │
│  │  • Receives lifecycle events      │  │ │
│  │  • Updates database on game end   │  │ │
│  └───────────────────────────────────┘  │ │
└─────────────────────────────────────────┘ │
                                            │
                     ┌──────────────────────┘
                     │ Direct WebSocket
                     │ [internal WebSocket endpoint]
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    GAME_RUNTIME (Node.js)                       │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Main Thread                                              │  │
│  │  • HTTP Server (session API)                              │  │
│  │  • WebSocket Server (Python bridge)                       │  │
│  │  • Direct Browser WebSocket Server                        │  │
│  │  • Session Worker Manager                                 │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                  │
│           ┌──────────────────┼──────────────────┐               │
│           │                  │                  │               │
│           ▼                  ▼                  ▼               │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐   │
│  │  Worker Thread  │ │  Worker Thread  │ │  Worker Thread  │   │
│  │  (Session 1)    │ │  (Session 2)    │ │  (Session 3)    │   │
│  │                 │ │                 │ │                 │   │
│  │  Game: Pong     │ │  Game: C4       │ │  Game: Word     │   │
│  │  Isolated V8    │ │  Isolated V8    │ │  Isolated V8    │   │
│  │  Heap           │ │  Heap           │ │  Heap           │   │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘   │
│                                                                 │
│  True process isolation: One crash cannot affect others         │
│  Hard timeout: worker.terminate() after configured limit        │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 Game SDK Architecture

The SDK is **completely game-agnostic** - it provides infrastructure only, not game-specific logic:

```
SDK COMPONENTS
══════════════

SERVER-SIDE (@welynk/game-sdk):
┌─────────────────────────────────────────────────────────────────┐
│  GameContext<TState>        Main API for game logic             │
│                                                                 │
│  Event handlers:                                                │
│    ctx.on('init')           Game initialization                 │
│    ctx.on('action')         Player action received              │
│    ctx.on('tick')           60Hz game loop (deltaTime)          │
│    ctx.on('timer')          Timer fired                         │
│    ctx.on('playerJoin')     Player joined                       │
│    ctx.on('playerReady')    Player ready (finished loading)     │
│    ctx.on('playerLeave')    Player left (after grace)           │
│    ctx.on('playerReconnect') Player reconnected within grace    │
│    ctx.on('cleanup')        Session about to be destroyed       │
│    ctx.on('hostChanged')    Host player changed                 │
│    ctx.on('lobbyAutoUnlisted') Platform unlisted the session    │
│                                                                 │
│  State management:                                              │
│    ctx.state                Current public state (read-only)    │
│    ctx.setState(updates)    Update public state → syncs clients │
│    ctx.serverState          Server-only state (never sent)      │
│    ctx.setServerState(updates)  Update server-only state        │
│    ctx.setStateFilter(fn)   Per-player state filtering          │
│                                                                 │
│  Communication:                                                 │
│    ctx.broadcast(event, data)     Send to all players           │
│    ctx.sendToPlayer(id, event, data) Send to specific player    │
│    ctx.reject(reason)             Reject current action         │
│                                                                 │
│  Sub-APIs:                                                      │
│    ctx.timer               Scheduled callbacks (start/cancel)   │
│    ctx.random              Seeded deterministic RNG             │
│    ctx.lynks               In-game currency operations          │
│    ctx.ads                 Ad placement (rewarded, interstitial)│
│    ctx.playerData          Persistent per-player storage        │
│    ctx.lobby               Lobby visibility (list/unlist/close) │
│                                                                 │
│  Tick info (realtime games):                                    │
│    ctx.tickNumber           Current tick number                 │
│    ctx.gameTime             Seconds elapsed since start         │
│                                                                 │
│  Physics:                                                       │
│    createWorld()            Physics world (Planck.js-based)     │
│    world.addCircle()        Add circle body                     │
│    world.addRectangle()     Add rectangle body                  │
│    world.step(dt)           Advance physics simulation          │
│    world.onCollision()      Collision callback                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

CLIENT-SIDE (@welynk/game-sdk/react):
┌─────────────────────────────────────────────────────────────────┐
│  useGame<TState>()          Main hook for state and actions     │
│    .state                   Current game state (filtered)       │
│    .players                 Player list                         │
│    .myPlayerId              This player's ID                    │
│    .sendAction(action)      Send action to server               │
│    .requestLeave()          Request to leave game               │
│    .gameStatus              Current game status                 │
│                                                                 │
│  useGameWithTiming<T>()     Like useGame + timing metadata      │
│    .serverTime              Server timestamp from last update   │
│    .tickNumber              Server tick number                  │
│                                                                 │
│  useGameEvent(name, cb)     Listen to server broadcasts         │
│  useOptimisticValue()       Local prediction for responsive UI  │
│  useSafeArea()              Device safe area insets             │
│  useDiagnostics()           Network/frame diagnostic metrics    │
│                                                                 │
│  <Smooth>                   60fps rendering component           │
│    - Bypasses React state batching                              │
│    - Direct DOM manipulation via entity interpolation           │
│    - Selector extracts position: (state) => ({x, y, rotation})  │
│                                                                 │
│  useGameLoop(callback)      Direct render loop access           │
│  useAnimationFrame(cb)      requestAnimationFrame wrapper       │
│                                                                 │
│  useMusic(id)               Background music control            │
│  useAudio(id)               Sound effect playback               │
│  useAmbient(id)             Ambient/looping audio               │
│  AudioProvider              Audio context management            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.4 Room and Lobby Model

Games organize players through a room/lobby system:

```
LOBBY LIFECYCLE
═══════════════

┌──────────────┐
│   WAITING    │  Players can join
│              │  Host configures game
└──────┬───────┘  Min players not met
       │
       │ (min players reached + host starts)
       │
       ▼
┌──────────────┐
│   STARTING   │  Countdown period
│              │  Late joins allowed
└──────┬───────┘  Final configuration
       │
       │ (countdown complete)
       │
       ▼
┌──────────────┐
│   PLAYING    │  Game in progress
│              │  Actions validated
└──────┬───────┘  State synchronized
       │
       │ (win condition or forfeit)
       │
       ▼
┌──────────────┐
│   FINISHED   │  Results displayed
│              │  Stats recorded
└──────────────┘  Room cleaned up
```

### 6.4 State Synchronization Model

Games use an authoritative server model:

```
CLIENT                           SERVER                          CLIENT
(Player 1)                    (Game Runtime)                   (Player 2)
    │                              │                               │
    │   action: place_piece(3,4)   │                               │
    ├─────────────────────────────▶│                               │
    │                              │                               │
    │                         ┌────┴────┐                          │
    │                         │VALIDATE │                          │
    │                         │ • Is it │                          │
    │                         │   P1's  │                          │
    │                         │   turn? │                          │
    │                         │ • Is    │                          │
    │                         │   (3,4) │                          │
    │                         │   valid?│                          │
    │                         └────┬────┘                          │
    │                              │                               │
    │    state_update: {...}       │     state_update: {...}       │
    │◀─────────────────────────────┼──────────────────────────────▶│
    │                              │                               │
    │  Full authoritative state    │   Full authoritative state    │
    │  sent to both players        │   sent to both players        │
    │                              │                               │
```

The server is the single source of truth. Clients send actions; the server validates and broadcasts the resulting state. Clients never modify state directly.

### 6.5 Tick Rate and Update Frequency

WeLynk uses **fixed, non-configurable rates** for all games:

```
┌─────────────────────────────────────────────────────────────────┐
│  FIXED PLATFORM RATES (Cannot Be Changed By Games)              │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Server Tick Rate:     60 Hz (16.67ms per tick)         │    │
│  │  Snapshot Emission:    30 Hz (every other tick)         │    │
│  │  Client Render Rate:   60 Hz (requestAnimationFrame)    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  Why 30Hz snapshots with 60Hz ticks?                            │
│    • 30fps is sufficient for visual smoothness (movies = 24fps) │
│    • Reduces bandwidth by 50%                                   │
│    • Client interpolation creates smooth 60fps rendering        │
│    • Physics still runs at 60Hz for accuracy                    │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  TICK TIMING PRECISION                                          │
│                                                                 │
│  Uses setImmediate() instead of setInterval():                  │
│    • setInterval can be delayed 30-150ms by GC/event loop       │
│    • setImmediate runs every event loop iteration               │
│    • Tracks accumulated time to prevent "catch-up death spiral" │
│    • Maximum catchup ticks per frame (configured)               │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  ACTION PROCESSING MODES                                        │
│                                                                 │
│  Games declare action handling via actionProfiles:              │
│                                                                 │
│  DISCRETE (Turn-based, card games):                             │
│    • Actions queued and processed in order                      │
│    • Rate limited, with action queue                            │
│    • Good for logic that must run in exact sequence             │
│                                                                 │
│  STREAM (Paddle movement, drawing):                             │
│    • Latest action replaces previous                            │
│    • Higher rate limit (for smooth input)                       │
│    • Coalesces updates (group every 16ms)                       │
│    • Good for continuous positional input                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. Game Physics and Client Interpolation

### 7.1 Physics Simulation

Real-time games use physics simulation based on classical mechanics. The system uses Planck.js (Box2D port) with these fundamental equations:

**Newton's Second Law:**
```
F = ma

where:
  F = net force vector (Newtons)
  m = mass (kilograms)
  a = acceleration vector (m/s²)

In discrete time steps:
  a = F / m
  v(t+Δt) = v(t) + a × Δt
  p(t+Δt) = p(t) + v(t+Δt) × Δt

where:
  Δt = time step (typically 1/60 second)
  v = velocity vector
  p = position vector
```

**Collision Detection (AABB):**
```
Two axis-aligned bounding boxes A and B collide if:

A.min.x ≤ B.max.x  AND  A.max.x ≥ B.min.x
AND
A.min.y ≤ B.max.y  AND  A.max.y ≥ B.min.y
```

**Collision Response (Elastic):**
```
For two objects with masses m₁, m₂ and velocities v₁, v₂:

After collision:
v₁' = v₁ - (2m₂/(m₁+m₂)) × ((v₁-v₂)·(x₁-x₂)/|x₁-x₂|²) × (x₁-x₂)
v₂' = v₂ - (2m₁/(m₁+m₂)) × ((v₂-v₁)·(x₂-x₁)/|x₂-x₁|²) × (x₂-x₁)

where x₁, x₂ are position vectors
```

### 7.2 Client-Side Interpolation

Network latency creates a fundamental problem: the client's view is always behind the server's reality. Interpolation smooths this gap.

**The Interpolation Buffer:**
```
Client maintains a buffer of recent server states:

Buffer = [(t₁, S₁), (t₂, S₂), (t₃, S₃), ...]

where tᵢ = server timestamp, Sᵢ = game state at that time

Render time is offset from current time:
render_time = current_time - interpolation_delay

Typical delay: 100ms (adjustable based on network conditions)
```

**Linear Interpolation:**
```
For two states S₁ at t₁ and S₂ at t₂, render state at time t:

α = (t - t₁) / (t₂ - t₁)

For each object property:
  position(t) = position(S₁) × (1-α) + position(S₂) × α
  rotation(t) = lerp_angle(rotation(S₁), rotation(S₂), α)

Angle interpolation (handles wraparound):
lerp_angle(a, b, α) = a + shortest_arc(a, b) × α
shortest_arc(a, b) = ((b - a + π) mod 2π) - π
```

### 7.3 Entity Interpolation (NOT Client Prediction)

WeLynk deliberately avoids client-side prediction in favor of **entity interpolation**:

```
WHY NO CLIENT PREDICTION?
═════════════════════════

Client-side prediction is complex:
  • Must simulate opponent actions
  • Must replicate server physics exactly
  • Must handle RNG seeds
  • Reconciliation code is error-prone
  • Mispredictions cause visual "snapping"

Entity interpolation is simple:
  • Just render the past
  • No simulation needed
  • No reconciliation needed
  • Always visually smooth


ENTITY INTERPOLATION
════════════════════

The client maintains a buffer of recent server state snapshots:

Buffer = [(t₁, S₁), (t₂, S₂), (t₃, S₃), ..., (tₙ, Sₙ)]
         └─────────────────────────────────────────────┘
                    Recent snapshots (configured buffer size)

Render at fixed delay behind real-time:
  render_time = current_time - interpolation_delay
  interpolation_delay = [configured default, adjustable per-game]

Find bracketing snapshots and interpolate:
  Snapshots: [{x:100, t:0}, {x:120, t:33}, {x:140, t:66}]
  Now = 82ms, render_time = 32ms
  Bracket: t:0 → t:33
  α = (32 - 0) / (33 - 0) = 0.97
  x = 100 × (1-0.97) + 120 × 0.97 = 119.4

Result:
  • Server sends 30Hz snapshots
  • Client renders at 60fps
  • All motion appears smooth
  • No prediction complexity
```

### 7.4 Lag Compensation

For competitive fairness, the server considers what each player saw when they acted:

```
LAG COMPENSATION
════════════════

When Player A shoots at time t_client:
  1. A's client sends: {action: "shoot", target_pos: (x,y), client_time: t}
  2. Server receives at t_server
  3. Server estimates A's network latency: RTT/2
  4. Server rewinds game state to (t_server - RTT/2)
  5. Server checks if shot would hit at rewound state
  6. If hit: apply damage
  7. Broadcast result to all players

This ensures: what you see is what you get (WYSIWYG)
Trade-off: Player being shot may feel hit "unfairly" due to their own latency
```

### 7.5 Ping Pong Physics Example

The Ping Pong game demonstrates these principles:

```
BALL PHYSICS
════════════

Ball properties:
  mass = 0.1 kg (light, fast response)
  radius = 10 pixels
  max_velocity = 800 pixels/second

Paddle collision:
  When ball hits paddle:
  1. Reflect velocity: v.y = -v.y
  2. Add spin based on paddle velocity:
     spin_factor = paddle.velocity.x × 0.3
     v.x += spin_factor
  3. Increase speed slightly: v *= 1.05
  4. Clamp to max: v = clamp(v, -max_velocity, max_velocity)

Wall collision:
  When ball hits side wall:
  v.x = -v.x × 0.98  (slight energy loss)

State update (server, 60Hz):
  ball.position += ball.velocity × (1/60)

Client interpolation delay: [configured] (fast game requires low delay)
Client prediction: none (entity interpolation only, see Section 7.3)
```

---

## 8. Direct Connection Architecture

### 8.1 Why Two Runtimes?

WeLynk uses Python for business logic and Node.js for game execution. This separation is intentional:

| Concern | Python | Node.js |
|---------|--------|---------|
| Business Logic | Excellent | Awkward |
| Type Safety (runtime) | Good | Limited |
| Async Database Access | Native | Requires care |
| Real-time Events | Adequate | Excellent |
| Game Development | Unusual | Natural fit |
| Third-party Game SDKs | Rare | Abundant |

Rather than routing all game traffic through Python, WeLynk uses a **Direct Connection** model where browsers connect directly to the Node.js game runtime. Python handles only lifecycle and economy operations through a separate control-plane channel.

### 8.2 Direct Connection Model

```
DIRECT CONNECTION ARCHITECTURE
══════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│  BROWSER (Client)                                               │
│                                                                 │
│  ┌───────────────────────────────────────────────┐              │
│  │  Game SDK (GameProvider)                       │              │
│  │                                               │              │
│  │  • Connects directly to game runtime via WS   │              │
│  │  • Sends player actions                       │              │
│  │  • Receives state snapshots at 30Hz           │              │
│  │  • Handles reconnection with exponential      │              │
│  │    backoff on disconnect                      │              │
│  │  • Accepts token refresh from runtime         │              │
│  └───────────────────────┬───────────────────────┘              │
│                          │                                      │
└──────────────────────────┼──────────────────────────────────────┘
                           │
                           │ Direct WebSocket
                           │ /game/{sessionId}?token=xxx
                           │ (JWT-authenticated)
                           │
┌──────────────────────────┼──────────────────────────────────────┐
│                          │                                      │
│  ┌───────────────────────▼───────────────────────┐              │
│  │  Direct Connection Handler                     │              │
│  │                                               │              │
│  │  • Verifies JWT token on connect              │              │
│  │  • Routes actions to correct game session     │              │
│  │  • Streams state snapshots to client          │              │
│  │  • Detects dead connections via ping/pong     │              │
│  │  • Proactively refreshes tokens before expiry │              │
│  └───────────────────────────────────────────────┘              │
│                                                                 │
│                      NODE.JS (game_runtime)                     │
│                                                                 │
│  ┌───────────────────────▲───────────────────────┐              │
│  │  Control-Plane Handler (separate /ws endpoint) │              │
│  │                                               │              │
│  │  • Session create/destroy                     │              │
│  │  • Player join/leave tracking                 │              │
│  │  • Economy verification (Lynks)               │              │
│  │  • Matchmaking coordination                   │              │
│  └───────────────────────┬───────────────────────┘              │
│                          │                                      │
└──────────────────────────┼──────────────────────────────────────┘
                           │
                           │ Internal WebSocket (/ws)
                           │ (shared-secret authenticated)
                           │
┌──────────────────────────┼──────────────────────────────────────┐
│                          │                                      │
│  ┌───────────────────────▼───────────────────────┐              │
│  │  MultiRuntimeBridge                            │              │
│  │                                               │              │
│  │  • Sends lifecycle events to runtime          │              │
│  │  • Receives game outcome events               │              │
│  │  • Handles economy/Lynks verification         │              │
│  │  • Monitors control-plane connection health   │              │
│  └───────────────────────────────────────────────┘              │
│                                                                 │
│                      PYTHON (user_backend)                      │
└─────────────────────────────────────────────────────────────────┘
```

The key insight: **game traffic (actions, state snapshots, events) flows directly between browser and Node.js**, never touching Python. This eliminates a full network hop for every player action and state update, reducing latency and freeing Python to handle business logic.

### 8.3 Two Connection Types

```
CONNECTION SEPARATION
═════════════════════

1. DIRECT CONNECTIONS (Browser ↔ Game Runtime)
   ──────────────────────────────────────────

   Purpose: Real-time game traffic
   Auth: Short-lived JWT (generated by Python backend)
   Traffic: Player actions, state snapshots, game events, broadcasts
   Latency: Minimal (single hop)

   Token lifecycle:
     • Python generates game token on session join
     • Browser presents token on WebSocket connect
     • Runtime verifies token independently
     • Runtime proactively refreshes token before expiry
     • Client SDK handles token refresh transparently


2. CONTROL-PLANE CONNECTION (Python ↔ Game Runtime)
   ─────────────────────────────────────────────────

   Purpose: Lifecycle and economy operations
   Auth: Shared secret (internal network only)
   Traffic: Session create/destroy, join/leave, Lynks verification

   Control-plane messages (Python → Node):
     • session_create    - Create new game session
     • session_destroy   - Destroy game session
     • player_join       - Register player in session
     • player_leave      - Remove player from session
     • lynks_result      - Economy transaction result
     • ping              - Health check

   Control-plane messages (Node → Python):
     • game_ended        - Game finished (results/stats)
     • lynks_request     - Economy transaction request
     • player_data       - Persistent data operations
     • room_status       - Room lifecycle event
     • pong              - Health check response
```

### 8.4 Health Monitoring

Both connection types implement health checking:

```
DIRECT CONNECTION HEALTH (Browser ↔ Runtime)
════════════════════════════════════════════

Runtime periodically pings each browser connection:
  • WebSocket ping/pong at configured interval
  • Connections not responding are marked dead
  • Dead connections trigger player leave (with grace period)
  • Prevents ghost players from stale connections

CONTROL-PLANE HEALTH (Python ↔ Runtime)
═══════════════════════════════════════

Python periodically pings the runtime:

  Health check parameters:
    [Specific values redacted for security]
    Includes: ping interval, timeout, max retries, retry delay
```

### 8.5 Reconnection Logic

```
CLIENT RECONNECTION (Browser → Runtime)
═══════════════════════════════════════

On WebSocket disconnect (unintentional):
  1. Client SDK detects connection loss
  2. Exponential backoff reconnection begins
     • Base delay: [configured]
     • Max delay: [configured]
     • Max attempts: [configured]
  3. On reconnect: token re-verified, state resynced
  4. Player appears to never have left (if within grace period)

On intentional close (game end, leave):
  • No reconnection attempted


CONTROL-PLANE RECONNECTION (Python ↔ Runtime)
═════════════════════════════════════════════

During RECONNECTING:
  • Control-plane events (join/leave/clock sync) are queued
  • No new games can be started
  • Direct runtime gameplay continues independently

On successful reconnection:
  • Queued control-plane events are replayed in order
  • Backend/session status resyncs with live runtime state
```

---

## 9. Game Security and Anti-Abuse

### 9.1 Threat Model

Third-party games pose unique security challenges:

```
THREAT CATEGORIES
═════════════════

1. RESOURCE EXHAUSTION
   • Infinite loops consuming CPU
   • Memory leaks filling RAM
   • Excessive network requests

2. DATA THEFT
   • Attempting to access user data
   • Reading other games' state
   • Exfiltrating to external servers

3. PLATFORM ABUSE
   • Sending spam through game chat
   • Displaying inappropriate content
   • Manipulating game outcomes for gambling

4. DENIAL OF SERVICE
   • Crashing the runtime
   • Blocking other games
   • Exhausting connection limits
```

### 9.2 Sandboxing Model

Games run in isolated sandboxes:

```
ISOLATION LAYERS
════════════════

┌─────────────────────────────────────────────────────────────────┐
│  LAYER 1: NETWORK ISOLATION                                     │
│                                                                 │
│  Game runtime runs on isolated Docker network                   │
│  ALLOWED connections:                                           │
│    • user_backend (control-plane WebSocket)                     │
│    • Browser clients (direct game WebSocket, token-auth)        │
│  BLOCKED connections:                                           │
│    • Database (PostgreSQL)                                      │
│    • Cache (Redis)                                              │
│    • External internet                                          │
│    • Other internal services                                    │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 2: PROCESS ISOLATION                                     │
│                                                                 │
│  Each game room runs with:                                      │
│    • Separate V8 isolate (memory isolation)                     │
│    • CPU time limits (configurable per-tick max)                │
│    • Memory limits (configurable per-room max)                  │
│    • No filesystem access                                       │
│    • No child process spawning                                  │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 3: API RESTRICTION                                       │
│                                                                 │
│  Games can only use approved SDK APIs:                          │
│    ✓ getGameState()                                             │
│    ✓ getPlayers()                                               │
│    ✓ sendAction()                                               │
│    ✓ broadcastMessage()                                         │
│    ✗ require('fs')                                              │
│    ✗ require('net')                                             │
│    ✗ process.env                                                │
│    ✗ eval()                                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 9.3 Resource Limits

Strict, non-configurable limits prevent resource abuse:

```
RESOURCE QUOTAS (All Non-Configurable)
══════════════════════════════════════

Per Player (Rate Limiting):
┌─────────────────────────────────────────────────────────────────┐
│  Action Type       │  Limit           │  Action on Exceed       │
├─────────────────────────────────────────────────────────────────┤
│  Discrete actions  │  [configured]    │  Action rejected        │
│  Stream actions    │  [configured]    │  Coalesced/dropped      │
│  Per-type tracking │  Yes             │  Per (player+actionType)│
└─────────────────────────────────────────────────────────────────┘

Per Game Session (Worker Thread):
┌─────────────────────────────────────────────────────────────────┐
│  Resource          │  Limit           │  Action on Exceed       │
├─────────────────────────────────────────────────────────────────┤
│  Action timeout    │  Configured      │  Action marked timeout  │
│  Hard timeout      │  Configured      │  worker.terminate()     │
│  Memory (soft)     │  Configured      │  Warning issued         │
│  Memory (hard)     │  Configured      │  Session force-ended    │
│  Per-worker memory │  Configured      │  V8 terminates isolate  │
│  State size        │  Configured      │  Emission blocked       │
└─────────────────────────────────────────────────────────────────┘

Per Runtime Instance:
┌─────────────────────────────────────────────────────────────────┐
│  Resource          │  Limit           │  Action on Exceed       │
├─────────────────────────────────────────────────────────────────┤
│  Active sessions   │  Configured      │  New sessions rejected  │
└─────────────────────────────────────────────────────────────────┘

[Specific resource limits redacted for security]


TIMEOUT ENFORCEMENT (True Worker Termination)
═════════════════════════════════════════════

Why worker threads matter for timeouts:

  Shared event loop (old approach):
    setTimeout(() => { /* this may never fire if game blocks */ })
    Game infinite loop → blocks health checks → blocks ALL games

  Worker threads (current approach):
    worker.terminate() → V8 forcefully killed
    Game infinite loop → only that worker dies
    Other games unaffected, health checks continue

This is TRUE timeout enforcement that cannot be circumvented.
```

### 9.4 Validation Rules

All game actions are validated:

```
VALIDATION PIPELINE
═══════════════════

Player action arrives
        │
        ▼
┌───────────────────┐
│  1. FORMAT CHECK  │ Is it valid JSON? Required fields present?
└─────────┬─────────┘
          │ pass
          ▼
┌───────────────────┐
│  2. SIZE CHECK    │ Within configured size limit?
└─────────┬─────────┘
          │ pass
          ▼
┌───────────────────┐
│  3. RATE CHECK    │ Under rate limit for this player?
└─────────┬─────────┘
          │ pass
          ▼
┌───────────────────┐
│  4. AUTH CHECK    │ Is player in this room? Is it their turn?
└─────────┬─────────┘
          │ pass
          ▼
┌───────────────────┐
│  5. GAME RULES    │ Is this action valid per game logic?
└─────────┬─────────┘
          │ pass
          ▼
    ACTION APPLIED
```

---

## 10. Audio Control Enforcement

### 10.1 The Problem

Games cannot be trusted with audio. A malicious game could:
- Play sounds at maximum volume to harass users
- Play audio during calls to disrupt communication
- Continue playing audio after the game ends
- Bypass user mute settings

### 10.2 Platform Audio Control

The platform, not games, controls audio:

```
AUDIO CONTROL HIERARCHY
═══════════════════════

┌─────────────────────────────────────────────────────────────────┐
│  LEVEL 1: PLATFORM MASTER CONTROL                               │
│                                                                 │
│  Platform settings override everything:                         │
│    • Master mute (mutes all audio)                              │
│    • Master volume (caps all audio)                             │
│    • Focus mode (only current activity plays audio)             │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  LEVEL 2: ACTIVITY PRIORITY                                     │
│                                                                 │
│  Priority order (higher = plays, lower = muted):                │
│    1. System notifications (highest)                            │
│    2. Active call audio                                         │
│    3. Foreground game audio                                     │
│    4. Background game audio (muted by default)                  │
│    5. Ambient sounds (lowest)                                   │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  LEVEL 3: USER PREFERENCES                                      │
│                                                                 │
│  User can adjust within platform limits:                        │
│    • Game volume slider (0-100%)                                │
│    • Mute specific games                                        │
│    • Mute during calls                                          │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  LEVEL 4: GAME REQUESTS                                         │
│                                                                 │
│  Games can only REQUEST audio, not control it:                  │
│    game.playSound("explosion", volume: 0.8)                     │
│                                                                 │
│  Platform decides actual playback:                              │
│    actual_volume = request_volume × user_pref × platform_cap    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 10.3 Technical Enforcement

```
ENFORCEMENT MECHANISM
═════════════════════

Game SDK provides audio API that routes through platform:

┌─────────────────┐          ┌─────────────────┐
│  GAME CODE      │          │  PLATFORM       │
│                 │          │                 │
│  playSound()  ──┼─────────▶│  AudioManager  │
│                 │  request │    │            │
│                 │          │    ▼            │
│                 │          │  Check:         │
│                 │          │  • User muted?  │
│                 │          │  • In call?     │
│                 │          │  • Volume cap   │
│                 │          │    │            │
│                 │          │    ▼            │
│                 │          │  Play or ignore │
└─────────────────┘          └─────────────────┘

Games CANNOT:
  • Access Web Audio API directly
  • Create audio elements
  • Modify volume after request
  • Detect if audio was actually played

This prevents:
  • Volume manipulation attacks
  • Audio timing attacks
  • Mute bypass attempts
```

---

## 11. Authentication and Session Management

### 11.1 Token Architecture

WeLynk uses a dual-token JWT system with specific-purpose tokens:

```
TOKEN TYPES
═══════════

┌─────────────────────────────────────────────────────────────────┐
│  ACCESS TOKEN                                                   │
│                                                                 │
│  Purpose: Authenticate API requests                             │
│  Lifetime: Short-lived (minutes)                                │
│  Storage: Memory only (never persisted)                         │
│                                                                 │
│  Contains: User identity, session reference, roles,             │
│            age classification, and standard JWT claims          │
│                                                                 │
│  [Specific claim structure redacted for security]               │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  REFRESH TOKEN                                                  │
│                                                                 │
│  Purpose: Obtain new access tokens                              │
│  Lifetime: Long-lived (days/weeks)                              │
│  Storage: HTTP-only secure cookie                               │
│                                                                 │
│  Contains: User identity, family tracking for rotation,         │
│            and standard JWT claims                              │
│                                                                 │
│  [Specific claim structure redacted for security]               │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  MEDIA SERVER TOKEN (for calls)                                 │
│                                                                 │
│  Purpose: Authenticate media server connections                 │
│  Lifetime: Short-lived (minutes)                                │
│                                                                 │
│  Contains: User identity, room permissions, and unique          │
│            token ID for per-token revocation                    │
│                                                                 │
│  [Specific claim structure redacted for security]               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 11.2 Token Lifecycle

```
AUTHENTICATION FLOW
═══════════════════

User Login
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. CREDENTIAL VALIDATION                                       │
│                                                                 │
│  • Verify password hash (Argon2id)                              │
│  • Check account status (active, suspended, etc.)               │
│  • Verify email is verified (non-temporary accounts)            │
│  • Verify 2FA if enabled                                        │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. TOKEN GENERATION                                            │
│                                                                 │
│  • Create new session in Redis                                  │
│  • Generate short-lived access token                            │
│  • Generate long-lived refresh token                            │
│  • Store refresh token family in database                       │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. TOKEN DELIVERY                                              │
│                                                                 │
│  • Access token: Response body (client stores in memory)        │
│  • Refresh token: HTTP-only cookie (Secure, SameSite=Strict)    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘


TOKEN REFRESH (with Distributed Locking)
════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│  1. ACQUIRE DISTRIBUTED LOCK                                    │
│                                                                 │
│  Key: [key pattern redacted]                          │
│  TTL: Short safety timeout                                      │
│                                                                 │
│  Purpose: Prevent race conditions when same refresh token       │
│  is used concurrently from multiple tabs/devices.               │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. VALIDATE REFRESH TOKEN                                      │
│                                                                 │
│  • Verify JWT signature                                         │
│  • Check expiration                                             │
│  • Verify family is not revoked                                 │
│  • Check user not deleted/inactive                              │
│  • Check revocation status via Redis (fail-closed on error)     │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. ROTATE TOKENS                                               │
│                                                                 │
│  • Revoke old refresh token                                     │
│  • Generate new refresh token (same family)                     │
│  • Generate new access token                                    │
│  • Release distributed lock                                     │
│                                                                 │
│  CRITICAL: If old token reused after rotation:                  │
│  → Entire family is revoked (potential theft detected)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘


TOKEN REVOCATION (Fail-Closed)
══════════════════════════════

Storage: Redis-backed distributed cache
Broadcast: Redis Pub/Sub channel

[Specific key patterns and data structure redacted for security]

CRITICAL: If revocation check errors → token refresh REJECTED
(Fail-closed strategy for security)
```

### 11.3 Password Security

```
PASSWORD HASHING
════════════════

Algorithm: Argon2id (memory-hard, resistant to GPU attacks)

Properties:
  • Memory-hard: Requires significant RAM, resists GPU/ASIC attacks
  • Configurable: Memory, time, and parallelism parameters tuned for security
  • Salted: Unique random salt per password
  • Slow by design: Intentionally slow verification to resist brute force

Why Argon2id?
  • Winner of the Password Hashing Competition
  • Combines Argon2i (side-channel resistant) and Argon2d (GPU-resistant)
  • OWASP recommended for new applications

[Specific parameters redacted for security]
```

### 11.4 Session Management

```
SESSION STORAGE (Redis)
═══════════════════════

[Specific key patterns redacted for security]
TTL: Extended on activity

Value structure:
  • User identifier
  • Timestamps (creation, last activity)
  • Device information (hashed for privacy)
  • Session status


SESSION LIMITS
══════════════

Per user:
  • Maximum concurrent sessions enforced
  • Oldest session evicted when limit exceeded
  • User can view and revoke sessions manually

Security triggers (all sessions revoked):
  • Password change
  • Security alert acknowledged
  • Admin action
```

---

## 12. Real-Time Messaging

### 12.1 WebSocket Event Model

```
EVENT NAMING CONVENTION
═══════════════════════

Format: {domain}:{entity}:{action} or {domain}:{action}

Direction indicators:
  C2S (Client → Server): Imperative verb (send, edit, delete)
  S2C (Server → Client): Past tense (sent, edited, deleted)
  Bidirectional: Same name both ways (typing)

CHAT DOMAIN EVENTS
══════════════════

Server-to-Client (S2C):
  chat:received         New message broadcast
  message:edited        Edit notification
  message:deleted       Deletion notification
  message:read          Read receipt
  reaction:added        Reaction added
  reaction:removed      Reaction removed
  chat:restored         Chat restored after deletion

Client-to-Server (C2S):
  chat:send             Send message
  message:edit          Edit message
  message:delete        Delete message
  message:mark_read     Mark as read
  reaction:add          Add reaction
  reaction:remove       Remove reaction
```

### 12.2 Message Delivery Pipeline

```
MESSAGE FLOW (with Cache Invalidation)
══════════════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│  STEP 1: CLIENT SENDS                                           │
│                                                                 │
│  WebSocket Event: chat:send                                     │
│  Payload: WebSocketSendMessagePayloadDict                       │
│    chat_id, content, reply_to (optional), attachments           │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 2: BLOCK CHECK (Fail-Closed)                              │
│                                                                 │
│  PRIVATE chats:                                                 │
│    If sender blocked recipient OR recipient blocked sender      │
│    → Message BLOCKED                                            │
│                                                                 │
│  GROUP chats:                                                   │
│    Message sent, but filtered at broadcast time                 │
│    (blocked users won't receive)                                │
│                                                                 │
│  On any error: FAIL-CLOSED (message blocked, not sent)          │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 3: SANITIZATION & LIMITS                                  │
│                                                                 │
│  XSS Protection: HTML sanitizer                                 │
│  Message length limits:                                         │
│    • Free users: [tier limit]                                   │
│    • Subscribers: [tier limit]                                  │
│                                                                 │
│  Encryption: Message encrypted with chat DEK                    │
│  (Data Encryption Key retrieved at system_core layer)           │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 4: PERSISTENCE                                            │
│                                                                 │
│  • Generate message ID                                          │
│  • Store in database with is_sent=false                         │
│  • Update chat last_activity                                    │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 5: CACHE INVALIDATION (BEFORE broadcast)                  │
│                                                                 │
│  CRITICAL ORDER: Invalidate BEFORE broadcast                    │
│                                                                 │
│  • Invalidate chat session cache                                │
│  • Invalidate all participant chat list caches                  │
│  • If recipient had deleted chat → restore it                   │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 6: BROADCAST                                              │
│                                                                 │
│  Via Redis Pub/Sub (reaches all workers):                       │
│    • Emit chat:received to all chat members                     │
│    • Respects block relationships (filtered per-recipient)      │
│                                                                 │
│  For offline members:                                           │
│    • Queue push notification (respects blocks, fail-closed)     │
│    • Increment unread counter                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘


READ RECEIPTS
═════════════

When user marks messages as read:
  1. Database updated
  2. Event `message:read` emitted to all OTHER participants
     (sender is excluded)
  3. `message:status:updated` sent back to reader
     (clears unread_count in their sidebar)

REACTIONS
═════════

Add reaction:
  1. Database records reaction
  2. Cache invalidation (chat session)
  3. Emit `reaction:added` to all participants

Remove reaction:
  1. Database removes reaction
  2. Cache invalidation
  3. Emit `reaction:removed` to all participants

MESSAGE EDITING
═══════════════

  1. Only content updated, is_edited=true flag set
  2. Cache invalidated IMMEDIATELY after DB write
  3. Emit `message:edited` (sender excluded from broadcast)

MESSAGE DELETION
════════════════

  1. Message marked deleted in DB
  2. Cache invalidated BEFORE broadcast (critical fix)
  3. Emit `message:deleted` to all participants
  4. Deleted messages never rendered (replies show "deleted")
```

### 12.3 Cross-Worker Broadcasting

```
MULTI-WORKER ARCHITECTURE
═════════════════════════

Problem:
  User A connected to Worker 1
  User B connected to Worker 3
  How does A's message reach B?

Solution: Redis Pub/Sub

┌──────────┐     ┌──────────┐      ┌──────────┐
│ Worker 1 │     │ Worker 2 │      │ Worker 3 │
│ (User A) │     │          │      │ (User B) │
└────┬─────┘     └────┬─────┘      └────┬─────┘
     │                │                 │
     │  PUBLISH       │                 │
     │  chat:room:xyz │                 │
     ├───────────────▶│◀───────────────┤
     │                │                 │
     │         ┌──────┴──────┐          │
     │         │    REDIS    │          │
     │         │   Pub/Sub   │          │
     │         └──────┬──────┘          │
     │                │                 │
     │  SUBSCRIBE     │   SUBSCRIBE     │
     │  chat:room:xyz │   chat:room:xyz │
     │◀───────────────┼───────────────▶│
     │                │                 │
     │                │   Deliver to    │
     │                │   User B        │

Each worker:
  1. Subscribes to channels for connected users' chats
  2. Publishes messages to relevant channels
  3. Receives messages for local delivery

For bot users:
  Events published to PSS bot event queue instead
```

### 12.4 Chat Status and Grace Period

```
CHAT STATUS VALUES
══════════════════

INACTIVE      Chat has no recent activity
GRACE_PERIOD  Time-limited window after match (allows messaging)
ACTIVE        Active conversation

GRACE PERIOD RULES
══════════════════

After a match:
  • Non-friends have limited time to message (grace period)
  • After grace period: must be friends to continue messaging
  • Grace period extends with each message during the window
```

---

## 13. Presence System

### 13.1 Status State Machine

WeLynk separates **presence status** (connection state) from **activity status** (what the user is doing):

```
PRESENCE STATES (Connection-Based)
══════════════════════════════════

┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│    ┌──────────┐    start searching     ┌────────────┐            │
│    │  ONLINE  │──────────────────────▶│ SEARCHING  │            │
│    └────┬─────┘◀──────────────────────└────────────┘            │
│         │         stop searching                                 │
│         │                                                        │
│         │ join call                                               │
│         ▼                                                        │
│    ┌──────────┐                                                  │
│    │ IN_CALL  │                                                  │
│    └────┬─────┘                                                  │
│         │ leave call                                              │
│         │                                                        │
│         └─────────────────┬──────────────────────────────────────│
│                           │                                      │
│                           │ disconnect                           │
│                           ▼                                      │
│                    ┌──────────────┐                               │
│                    │ GRACE_PERIOD │  (still appears online)       │
│                    └──────┬───────┘                               │
│                           │                                      │
│              reconnect    │    grace expires                      │
│              ┌────────────┤                                      │
│              │            ▼                                      │
│              │       ┌──────────┐                                │
│              └──────▶│ OFFLINE  │                                │
│                      └──────────┘                                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

Presence status definitions:
  ONLINE       - Connected via WebSocket
  SEARCHING    - Actively searching for a match
  IN_CALL      - Currently in a voice/video call
  GRACE_PERIOD - Disconnected but still appearing online
  OFFLINE      - Not connected (grace period expired)


ACTIVITY STATES (What the User is Doing)
════════════════════════════════════════

  IDLE      - Online but inactive
  ACTIVE    - Actively using the app
  IN_GAME   - Playing a game
  IN_CHAT   - Viewing a chat
  IN_CALL   - In a voice/video call

Activity status is derived from user actions (sending messages,
playing games, etc.) and is independent of presence status.
```

### 13.2 Presence Architecture

```
PRESENCE DATA FLOW
══════════════════

On WebSocket Connect:
  1. Set user status to ONLINE in session_api (Redis)
  2. Add user to online sorted set: users:online
  3. Broadcast presence:updated to friends (respecting privacy)
  4. Subscribe user to friend presence updates

On Activity:
  1. Activity update called on message send
  2. Activity types: "game" (with game info), "call"
  3. Stored in Redis cache
  4. Activity cleared when game/call ends

On WebSocket Disconnect:
  1. User marked offline in session_api
  2. Offline broadcast SCHEDULED (not sent immediately)
  3. Grace period marker set with configurable TTL
  4. User APPEARS online during grace period

Grace Period Expiration:
  1. Background task checks for expired grace period markers
  2. Sends presence update (offline) to friends
  3. Removes grace period marker

Reconnect During Grace Period:
  1. Clear pending offline broadcast
  2. Grace period marker deleted
  3. User continues appearing online (no disconnect notification)


REDIS STRUCTURE
═══════════════

[Specific key patterns redacted for security]

Data stored:
  • Online user tracking (sorted set)
  • Current activity state (game/call)
  • Grace period markers
  • Privacy check caching with instant invalidation
```

### 13.3 Privacy and Visibility

```
MUTUAL PRIVACY CHECKS
═════════════════════

For presence to be visible, BOTH users must allow it.

Settings (all require mutual consent):
  • show_online_status:    Both must have setting = True
  • hide_read_receipts:    Both must have setting = False
  • hide_typing_indicator: Both must have setting = False

Cache Pattern:
  [Specific key patterns redacted for security]
  TTL: Short duration for privacy sensitivity

Instant Invalidation:
  On privacy setting change: O(K) invalidation via index

FAIL-CLOSED:
  On error checking privacy → assume hidden (safe default)


PRESENCE EVENT
══════════════

Event: presence:updated
Payload:
  user_id: string
  online: boolean
  last_active: ISO string (only if offline)


STALE USER CLEANUP
══════════════════

Background task periodically checks:
  • Users marked online with no WebSocket SIDs
  • Excludes user simulator bots from cleanup
  • Sets truly stale users offline
```

### 13.4 Subscription Model

```
PRESENCE SUBSCRIPTIONS
══════════════════════

Who sees whose presence:
  • Friends see each other (respecting mutual privacy)
  • Chat members see each other while chat is open
  • Game players see each other during game

Privacy Rules:
  • Blocked users never see each other's presence
  • Minors' presence visible only to friends
  • Mutual privacy settings control visibility (see 13.3)

Subscription limits:
  • Max presence subscriptions per user (configured limit)
  • Subscriptions auto-expire after configurable period of no interest
  • Presence updates batched (rate limited to client)
```

---

## 14. Voice and Video Calls

### 14.1 SFU Architecture

WeLynk uses a Selective Forwarding Unit (SFU) model via LiveKit (self-hosted):

```
SFU vs MESH COMPARISON
══════════════════════

Mesh (peer-to-peer):
  Each participant sends to every other participant
  For N users: N×(N-1) streams
  4 users = 12 streams
  Bandwidth scales O(N²) - infeasible for groups

SFU (server-mediated):
  Each participant sends to server once
  Server forwards to other participants
  For N users: N upload + N×(N-1) download (server handles)
  Client bandwidth scales O(N) - much better

┌─────────┐     ┌─────────┐     ┌─────────┐
│ User A  │     │ User B  │     │ User C  │
└────┬────┘     └────┬────┘     └────┬────┘
     │               │               │
     │  1 upload     │  1 upload     │  1 upload
     │               │               │
     └───────────────┼───────────────┘
                     │
              ┌──────┴──────┐
              │   LiveKit   │
              │     SFU     │
              └──────┬──────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
     │  2 downloads  │  2 downloads  │  2 downloads
     │               │               │
     ▼               ▼               ▼
┌─────────┐     ┌─────────┐     ┌─────────┐
│ User A  │     │ User B  │     │ User C  │
│ sees B,C│     │ sees A,C│     │ sees A,B│
└─────────┘     └─────────┘     └─────────┘


LIVEKIT CONFIGURATION
═════════════════════

Server Architecture:
  • Production: Secure WebSocket (wss://)
  • Development: Local WebSocket

Room Configuration:
  • Room prefix pattern for call isolation
  • Maximum participants per room enforced
  • Auto-cleanup of empty/abandoned rooms

[Specific URLs and timeout values redacted for security]
```

### 14.2 Call Signaling Flow

```
CALL INITIATION
═══════════════

┌─────────────────────────────────────────────────────────────────┐
│  1. CALLER INITIATES                                            │
│                                                                 │
│  POST /api/v1/calls/start                                       │
│  Body: { recipient_id, call_type: "video" | "audio" }           │
│                                                                 │
│  Server:                                                        │
│    • Create call record in database                             │
│    • Create LiveKit room via API                                │
│    • Generate room token with unique JTI                        │
│      Unique token ID for per-token revocation                   │
│                                                                 │
│  Rate limited to prevent abuse                                  │
│  Temporary accounts have lifetime call limits                   │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. RECIPIENT NOTIFIED                                          │
│                                                                 │
│  WebSocket: call:incoming                                       │
│  Payload:                                                       │
│    call_id: "call_xyz"                                          │
│    caller: { id, display_name, avatar_url }                     │
│    call_type: "video"                                           │
│                                                                 │
│  Ring timeout: Configurable per environment                     │
│  If not answered → call marked MISSED                           │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
               ┌────────────┴────────────┐
               ▼                         ▼
┌─────────────────────────┐   ┌─────────────────────────┐
│  ACCEPT                 │   │  DECLINE                │
│                         │   │                         │
│  POST .../accept        │   │  POST .../decline       │
│  Server generates       │   │  Caller notified        │
│  room token             │   │  Call record updated    │
│  Both join LiveKit room │   │                         │
└───────────┬─────────────┘   └─────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. MEDIA EXCHANGE                                              │
│                                                                 │
│  Both clients connect to LiveKit room with tokens               │
│  LiveKit validates: signature, TTL, grants                      │
│  Webhook notifies backend: participant_joined                   │
│                                                                 │
│  State machine: RINGING → CONNECTING → ACTIVE                   │
│  Connection timeout: Configurable for WebRTC negotiation        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘


CALL END SECURITY
═════════════════

When call ends (user calls /calls/end):
  1. Event call:ended broadcast
  2. ALL LiveKit tokens revoked (all participants disconnected)
  3. Room lifecycle ends

SECURITY P1-7 FIX:
  If a token is compromised, attacker can only rejoin within token lifetime window.
  On call end, ALL participants are force-disconnected from room.
  Prevents eavesdropping on "ended" calls.
```

### 14.3 Smart Disconnect Detection

```
PROBLEM
═══════

LiveKit webhook reports "participant disconnected"
But user may still be online (just had a network hiccup)

SOLUTION: Heartbeat + Grace Period
══════════════════════════════════

1. Check if user is online via WebSocket presence

2. If online: User had network hiccup
   → Keep them in call, await LiveKit reconnect

3. If offline: Ping user with call:heartbeat:request

4. User must respond with call:heartbeat within timeout

5. If responds: Give grace period to reconnect to media server

6. If no response after configured attempts: Remove from call


HEARTBEAT CONFIGURATION
═══════════════════════

Parameters (configurable per environment):
  • Heartbeat ping interval (production vs test)
  • Response timeout before marking unresponsive
  • Maximum ping attempts before removal
  • Network grace period for reconnection
  • Stale threshold based on connection health

[Specific values redacted for security]


SINGLE PARTICIPANT BEHAVIOR
═══════════════════════════

When call goes from 2 → 1 participant:
  • Timeout starts for reconnection window
  • If no one rejoins → call ends
  • Timer resets if participant count fluctuates

This mimics Discord behavior: allows brief disconnects
without immediately ending the call.
```

### 14.4 WebSocket Events (Call Domain)

```
SERVER-TO-CLIENT (S2C)
══════════════════════

call:incoming              Incoming call notification
call:answered              Call answered by recipient
call:declined              Call declined
call:cancelled             Call cancelled by caller
call:ended                 Call ended
call:participant:joined    Participant joined room
call:participant:left      Participant left room
call:participant:reconnecting  User attempting to reconnect
call:participant:reconnected   Participant reconnected
call:state:restored        Call state restored after server restart
call:device:switched       User switched camera/microphone
call:active                Call is now active (all parties connected)
call:missed                Call was missed (ring timeout)
call:signal                SDP/ICE candidate signaling
call:reconnect_needed      Client should reconnect (network issue)
call:token:refreshed       Token refreshed during long call
call:heartbeat:request     Server requesting heartbeat

CLIENT-TO-SERVER (C2S)
══════════════════════

call:heartbeat             Heartbeat response
  Payload: { call_id, request_id, timestamp }
```

### 14.5 Adaptive Bitrate Model

```
QUALITY ADAPTATION
══════════════════

LiveKit implements simulcast with 3 quality layers:

┌─────────────────────────────────────────────────────────────────┐
│  Layer   │  Resolution   │  Bitrate      │  When Selected       │
├─────────────────────────────────────────────────────────────────┤
│  High    │  HD           │  [configured] │  Good bandwidth      │
│  Medium  │  SD           │  [configured] │  Moderate bandwidth  │
│  Low     │  Low-res      │  [configured] │  Poor bandwidth      │
└─────────────────────────────────────────────────────────────────┘

Selection algorithm:

Let B = available bandwidth
Let Q = {high, medium, low}
Let bitrate(q) = required bitrate for quality q

Selected quality = max { q ∈ Q : bitrate(q) ≤ B × headroom_factor }

The headroom factor [value redacted] provides margin for:
  • Audio (typically 32-64 Kbps)
  • Control signaling
  • Bandwidth fluctuations


TOKEN REFRESH DURING LONG CALLS
═══════════════════════════════

LiveKit tokens expire after a configured duration.
For long calls exceeding token lifetime:
  • Event: call:token:refreshed (Priority 3, server-pushed)
  • Endpoint: /calls/refresh-token
  • Allows seamless token rotation without disconnecting
```

---

## 15. The Store and Item Rendering

### 15.1 Database Schema

```
STORE TABLES
════════════

store_items (Primary Catalog)
─────────────────────────────
  item_id             UUID (PK)
  creator_id          String (FK to users, NULL for system items)
  name                String (max 200)
  slug                String (max 200, URL-friendly)
  description         Text
  item_type           String: background, profile_background, pfp_frame,
                              message_border, sticker_pack, emoji_pack
  category            String (filtering: "gradients", "patterns", "glow")
  tags                JSONB (array of search keywords)
  price_lynks         Integer (default 0)
  is_premium          Boolean (requires subscription)
  subscriber_price_lynks  Integer (NULL=use regular, 0=free)
  thumbnail_url       Text (CloudFront URL)
  asset_url           Text (CloudFront URL)
  config              JSONB (rendering configuration)
  config_hash         String (32 chars, SHA256 for sync detection)
  status              String: draft, pending_review, approved, rejected
  is_active           Boolean (visible in catalog)
  purchase_count      Integer (analytics)
  rejection_reason    Text (admin feedback if rejected)
  created_at          DateTime
  updated_at          DateTime

  UNIQUE: (slug, item_type, creator_id)

user_inventory (Purchases)
──────────────────────────
  inventory_id        UUID (PK)
  user_id             String (FK)
  item_id             UUID (FK to store_items)
  purchased_at        DateTime
  purchase_price      Integer (price at time of purchase)
  transaction_id      UUID (FK to lynk_transactions, audit trail)
  gifted_by_user_id   String (FK, NULL if purchased)
  gift_message        Text
  gifted_at           DateTime

  UNIQUE: (user_id, item_id) - Each user owns item at most once
```

### 15.2 Item Types and Rendering

```
STORE ITEM TYPES
════════════════

┌─────────────────────────────────────────────────────────────────┐
│  Type               │  Canvas           │  Rendering Method    │
├─────────────────────────────────────────────────────────────────┤
│  background         │  Chat messages    │  CSS gradient/image  │
│  profile_background │  Profile page     │  CSS gradient/image  │
│  pfp_frame          │  Avatar overlay   │  CSS or PNG overlay  │
│  message_border     │  Message bubble   │  9-slice CSS         │
│  sticker_pack       │  Message panel    │  Individual PNGs     │
│  emoji_pack         │  Message panel    │  Individual PNGs     │
└─────────────────────────────────────────────────────────────────┘
```

### 15.3 Item Configuration Schemas

```
PFP FRAME CONFIGURATION
═══════════════════════

Overlay Type (PNG on avatar):
{
  "renderType": "overlay",
  "frameUrl": "[cdn-url]/store/assets/{id}/frame.png",
  "size": 1.5  // 0.8x to 2.0x multiplier
}

Border Type (CSS only, no image):
{
  "renderType": "border",
  "borderColor": "#FF0000",
  "borderWidth": 4,
  "borderRadius": 20
}

Glow Type (CSS only, no image):
{
  "renderType": "glow",
  "glowColor": "#00FF00",
  "glowIntensity": 0.8,
  "glowRadius": 15
}


MESSAGE BORDER CONFIGURATION (9-Slice)
══════════════════════════════════════

{
  "renderType": "border-image",
  "borderImage": "[cdn-url]/store/assets/{id}/frame.png",
  "borderImageSlice": [20, 20, 20, 20],   // [top, right, bottom, left] pixels
  "borderImageWidth": [20, 20, 20, 20],   // CSS border width
  "borderImageOverlap": [0, 0, 0, 0],     // Seam adjustment
  "borderImageRepeat": "stretch",          // stretch | repeat | round
  "minWidth": 100,
  "minHeight": 100,
  "borderRadius": 8,
  "shadow": {
    "color": "rgba(0,0,0,0.3)",
    "blur": 8,
    "spread": 2,
    "offsetX": 0,
    "offsetY": 4
  }
}


STICKER/EMOJI PACK CONFIGURATION
════════════════════════════════

{
  "renderType": "sticker-pack",
  "stickers": [
    {
      "id": "sticker-1",
      "file": "stickers/happy-face.png",  // Relative to asset base
      "keywords": ["happy", "emoji", "face"]
    }
  ]
}

{
  "renderType": "emoji-pack",
  "emojis": [
    {
      "id": "emoji-1",
      "file": "emojis/smile.png",
      "shortcode": ":smile:"  // Used in messages
    }
  ]
}
```

### 15.4 The 9-Slice Technique

```
9-SLICE RENDERING
═════════════════

Image divided into 9 regions:

┌───────┬─────────────────────┬───────┐
│  TL   │        TOP          │  TR   │
│  (1)  │        (2)          │  (3)  │
├───────┼─────────────────────┼───────┤
│       │                     │       │
│ LEFT  │       CENTER        │ RIGHT │
│  (4)  │        (5)          │  (6)  │
│       │                     │       │
├───────┼─────────────────────┼───────┤
│  BL   │       BOTTOM        │  BR   │
│  (7)  │        (8)          │  (9)  │
└───────┴─────────────────────┴───────┘

Scaling rules:
  • Corners (1,3,7,9): Never scaled, maintain exact pixels
  • Top/Bottom (2,8): Scale horizontally only
  • Left/Right (4,6): Scale vertically only
  • Center (5): Scale both directions

CSS generated by frontend:
  border-style: solid;
  border-width: {top}px {right}px {bottom}px {left}px;
  border-image-source: url({sprite.png});
  border-image-slice: {top} {right} {bottom} {left};
  border-image-width: {top}px {right}px {bottom}px {left}px;
  border-image-repeat: stretch;
```

### 15.5 CDN and Asset Delivery

```
CLOUDFRONT DISTRIBUTION
═══════════════════════

Base URL: [cdn-url]/

URL patterns:
  Thumbnails: [cdn-url]/store/thumbnails/{item_id}.png
  Assets:     [cdn-url]/store/assets/{item_id}/frame.png

For public assets (store items):
  • Direct CloudFront URL (no signing needed)
  • Edge caching with configured TTL for item details
  • Shorter TTL for catalog lists (refreshed more frequently)

For private assets:
  • Presigned S3 URL with TTL


STORAGE HIERARCHY
═════════════════

store-assets/
├── thumbnails/
│   └── {item_id}.png
└── assets/
    └── {item_id}/
        ├── frame.png
        └── (additional assets)
```

### 15.6 Purchase Transaction Flow

```
ATOMIC PURCHASE OPERATION
═════════════════════════

1. Validate item exists, is_active, not already owned
2. Get wallet (includes subscription status)
3. Determine price:
   • Subscriber? Use subscriber_price_lynks (if set)
   • Otherwise use price_lynks
4. Check balance ≥ price

5. ATOMIC OPERATION (with idempotency key):
   • Deduct Lynks from wallet
   • Create transaction log entry
   • Idempotency key prevents duplicate charges on retry

6. Add to user_inventory
7. Increment purchase_count (non-critical)
8. Generate signed receipt token
9. Store receipt in database
10. Send confirmation email (async)

REFUND SAFETY:
  If inventory add fails after deduction → automatic refund


GIFT FLOW
═════════

Same as purchase, but:
  • Validates sender & recipient are friends
  • Checks neither has blocked the other
  • Uses sender's subscription status for pricing
  • Deducts from sender's balance
  • Adds to recipient's inventory
  • Creates chat system message with gift notification
  • Emits WebSocket: store:gift:received
  • Stores: gifted_by_user_id, gift_message, gifted_at
```

### 15.7 Item Lifecycle

```
STATUS FLOW
═══════════

    ┌─────────┐
    │  draft  │  Creator working on item
    └────┬────┘
         │
         │ Submit for review
         ▼
┌────────────────┐
│ pending_review │  Awaiting admin approval
└───────┬────────┘
        │
   ┌────┴────┐
   │         │
   ▼         ▼
┌────────┐  ┌──────────┐
│approved│  │ rejected │  (rejection_reason stored)
│        │  │          │
│ LIVE   │  │ Feedback │
│ IN     │  │ provided │
│ STORE  │  │ to       │
│        │  │ creator  │
└────────┘  └──────────┘


CACHING
═══════

TTL values:
  • Individual items: [configured TTL]
  • Catalog lists: [configured TTL]
  • Cache invalidation on purchases/updates
```

---

## 16. Creator Studio

### 16.1 Platform Vision

Creator Studio is WeLynk's unified platform for user-generated content:

```
CREATOR STUDIO OVERVIEW
═══════════════════════

                    ┌─────────────────────────────────┐
                    │        CREATOR STUDIO           │
                    │    (Currently in Beta)          │
                    └───────────────┬─────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               ▼
           ┌───────────────┐               ┌───────────────┐
           │     GAMES     │               │  STORE ITEMS  │
           │               │               │               │
           │ • Multiplayer │               │ • Frames      │
           │ • Turn-based  │               │ • Stickers    │
           │ • Real-time   │               │ • Backgrounds │
           │ • Physics     │               │ • Effects     │
           └───────────────┘               └───────────────┘

Value proposition:
  • Creators: Reach WeLynk's user base, earn revenue
  • Users: More content variety, community creations
  • Platform: Engagement, retention, content diversity
```

### 16.2 Submission and Review Pipeline

```
CONTENT SUBMISSION FLOW
═══════════════════════

┌─────────────────────────────────────────────────────────────────┐
│  STEP 1: CREATION                                               │
│                                                                 │
│  Games:                                                         │
│    • Use Game SDK (TypeScript/React)                            │
│    • Test locally with development tools                        │
│    • Package with manifest.json                                 │
│                                                                 │
│  Store Items:                                                   │
│    • Use Frame Editor tool (9-slice, overlays)                  │
│    • Upload assets in required formats                          │
│    • Configure metadata (name, price, tags)                     │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 2: AUTOMATED CHECKS                                       │
│                                                                 │
│  Games:                                                         │
│    • Static analysis (no forbidden APIs)                        │
│    • Resource usage simulation                                  │
│    • Security sandbox verification                              │
│                                                                 │
│  Store Items:                                                   │
│    • File format validation                                     │
│    • Size/resolution requirements                               │
│    • PhotoDNA scan (no prohibited content)                      │
│                                                                 │
│  Result: PASS → continue | FAIL → feedback to creator           │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 3: HUMAN REVIEW                                           │
│                                                                 │
│  Moderator checks:                                              │
│    • Content appropriateness                                    │
│    • Quality standards                                          │
│    • Policy compliance                                          │
│    • No copyright infringement                                  │
│                                                                 │
│  Result: APPROVED | REJECTED (with feedback) | REVISIONS NEEDED │
│                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 4: PUBLICATION                                            │
│                                                                 │
│  On approval:                                                   │
│    • Content goes live in browse/search                         │
│    • Creator notified                                           │
│    • Analytics tracking begins                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 16.3 Revenue Model

```
CREATOR REVENUE SHARING
═══════════════════════

When users purchase creator content:

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Revenue is split between creator and platform                  │
│                                                                 │
│  Creator payout:                                                │
│    • Accumulate until minimum threshold                         │
│    • Monthly payout cycle                                       │
│    • Converted to USD at standard rate                          │
│                                                                 │
│  [Specific percentages and thresholds redacted - proprietary]   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Creator analytics dashboard shows:
  • Total sales (units and revenue)
  • Daily/weekly/monthly trends
  • Geographic distribution
  • Popular items ranking
```

---

## 17. User Safety Systems

### 17.1 Four-Level Safety Architecture

```
SAFETY PIPELINE
═══════════════

Every piece of user content passes through this 4-level pipeline:

┌─────────────────────────────────────────────────────────────────┐
│  LEVEL 1: CLIENT-SIDE (UX only - no security value)             │
│                                                                 │
│  Purpose: Improve user experience, reduce server load           │
│  Limitation: Completely bypassable - server enforces all rules  │
│                                                                 │
│  • Input length limits (tier-based)                             │
│  • Basic format validation                                      │
│  • Rate limiting feedback                                       │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  LEVEL 2: GATEWAY (Request Validation)                          │
│                                                                 │
│  Purpose: Enforce contracts, authenticate, rate limit           │
│                                                                 │
│  • Schema validation (Pydantic)                                 │
│  • JWT authentication verification                              │
│  • Distributed rate limiting (Redis + Lua atomic ops)           │
│  • Request size limits                                          │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  LEVEL 3: SAFETY ENGINE (Content Analysis)                      │
│                                                                 │
│  Purpose: Analyze content for policy violations                 │
│                                                                 │
│  Core orchestrator: Centralized moderation service              │
│  Returns verdict: ACCEPT | ACCEPT_WITH_FLAG | REJECT            │
│                                                                 │
│  TEXT MODERATION (12+ specialized detectors):                   │
│    • Language filter (profanity, explicit)                      │
│    • Bullying/harassment detector                               │
│    • Grooming pattern detector                                  │
│    • Age falsification detector                                 │
│    • Self-harm detection                                        │
│    • Extortion/blackmail detector                               │
│    • Scam detector                                              │
│    • Psychological manipulation detector                        │
│    • Link intelligence (malware, phishing)                      │
│    • Injection detector (SQL, XSS, command)                     │
│    • Emoji pattern filter (spam detection)                      │
│    • Low-effort spam filter                                     │
│                                                                 │
│  IMAGE MODERATION:                                              │
│    • CSAM detection (PhotoDNA + hash database)                  │
│    • AWS Rekognition (nudity, violence, drugs, hate symbols)    │
│    • GIF/sticker safety (Tenor API integration)                 │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  LEVEL 4: HUMAN REVIEW (Flagged Content Queue)                  │
│                                                                 │
│  Purpose: Handle edge cases, appeals, complex decisions         │
│                                                                 │
│  • Flagged content queue (ACCEPT_WITH_FLAG → admin review)      │
│  • Report investigation                                         │
│  • Ban appeal processing                                        │
│  • Policy edge case decisions                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 17.2 Five-Level Intensity System

```
CONTEXT-AWARE FILTER INTENSITY
══════════════════════════════

Filter strictness varies by relationship and age context:

Level    │ When Applied                               │ Behavior
─────────┼────────────────────────────────────────────┼──────────────────────
REJECT   │ Missing user age (data integrity failure)  │ Request rejected
(-1)     │                                            │
─────────┼────────────────────────────────────────────┼──────────────────────
LEVEL 0  │ Self-chat, adults+friends, minors+friends  │ CSAM only
UNREST.  │                                            │ No profanity filter
─────────┼────────────────────────────────────────────┼──────────────────────
LEVEL 1  │ Two adults, NOT friends                    │ Profanity rejected
BASELINE │                                            │ Mild harassment flagged
─────────┼────────────────────────────────────────────┼──────────────────────
LEVEL 2  │ Two minors NOT friends, all-minor groups,  │ Profanity + harassment
MEDIUM   │ cross-age game chat                        │ rejected, stricter
─────────┼────────────────────────────────────────────┼──────────────────────
LEVEL 3  │ Public profile fields (bio, username)      │ Maximum filter for
HIGH     │                                            │ adult visibility to minors
─────────┼────────────────────────────────────────────┼──────────────────────
LEVEL 4  │ Adult ↔ Minor interactions (any direction) │ Strictest: grooming,
MAXIMUM  │                                            │ manipulation rejected

Automatic triggers:
  • Group chat has_minor + has_adult → LEVEL 4
  • Game chat with mixed ages → LEVEL 2
  • Profile updates (public fields) → LEVEL 3 always
```

### 17.3 Minor Protection System

```
MINOR PROTECTION (Ages 16-17)
═════════════════════════════

AGE STORAGE & VERIFICATION
──────────────────────────
Age stored as String(50) in users table
Minor: 16 ≤ age ≤ 17
Adult: age ≥ 18
Minimum platform age: 16

MATCHING ENFORCEMENT (SQL-Level, No Exceptions)
───────────────────────────────────────────────
Minors can ONLY match with other minors (16-17)
Adults can ONLY match with other adults (18+)
Any adult-minor pair: IMMEDIATE REJECT

Enforcement logic:

  Minor = ages 16-17
  Adult = ages 18+

  Cross-age matching (minor ↔ adult): IMMEDIATE REJECT

Fail-closed: Unknown age → BLOCK match (not allow)


AGE CHANGE PROTECTION (Attack Prevention)
─────────────────────────────────────────
Attack: Age manipulation to circumvent minor protection

Defenses:
  • Age change cooldown enforced
  • Adult → Minor changes blocked with flagging
  • Age transitions trigger appropriate relationship termination
  • Significant changes logged for audit

[Specific thresholds and enforcement details redacted for security]


PROACTIVE SAFETY ALERTS
───────────────────────
When suspicious patterns detected, empowering alerts sent to minor users.

Alert categories include:
  • Age falsification detection
  • Contact information sharing
  • Financial requests
  • Cross-age interaction warnings
  • Combined suspicious pattern detection

[Specific triggers and thresholds redacted for security]

All alerts are empowerment-focused, providing report/block actions.


AGE FALSIFICATION DETECTOR
──────────────────────────
Multi-method detection system for identifying adults posing as minors:

Detection Categories:
  • Confession pattern analysis
  • Life experience indicator detection
  • Language and dialect inconsistency analysis
  • Behavioral timeline analysis
  • Generational communication pattern analysis

Output: Empowering warning to minor with block/report options

[Specific detection patterns redacted for security]
```

### 17.4 PhotoDNA and CSAM Detection

```
CSAM DETECTION ARCHITECTURE
═══════════════════════════

Legal Requirement: 18 USC §2258A (REPORT Act)
Philosophy: FAIL-CLOSED (any component failure → quarantine + ban)


DETECTION FLOW
──────────────

┌─────────────────────────────────────────────────────────────────┐
│  IMAGE UPLOAD                                                   │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  LAYER 1: Local Hash Database (Instant)                     ││
│  │                                                             ││
│  │  • Compute MD5 + SHA-256 of image                           ││
│  │  • Check against known CSAM hashes (O(1) set lookup)        ││
│  │  • Sources: NCMEC exports, IWF lists, PhotoDNA known        ││
│  │  • Match → IMMEDIATE QUARANTINE                             ││
│  └──────────────────────────┬──────────────────────────────────┘│
│                             │                                   │
│                    (if no match)                                │
│                             │                                   │
│                             ▼                                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  LAYER 2: PhotoDNA Gateway (Perceptual Hashing)             ││
│  │                                                             ││
│  │  Standalone service                                         ││
│  │  • Base64 decode image                                      ││
│  │  • Generate PhotoDNA perceptual hash (144-bit fingerprint)  ││
│  │  • Query Microsoft PhotoDNA Database                        ││
│  │  • Configured timeout with fail-closed behavior             ││
│  │  • Concurrency: Physical CPU cores (hash gen is CPU-bound)  ││
│  └──────────────────────────┬──────────────────────────────────┘│
│                             │                                   │
│                    ┌────────┴────────┐                          │
│                    │                 │                          │
│                    ▼                 ▼                          │
│               NO MATCH          MATCH DETECTED                  │
│               (proceed)              │                          │
│                                      ▼                          │
│          ┌───────────────────────────────────────────┐          │
│          │  IMMEDIATE ACTIONS (Fail-Closed)          │          │
│          │                                           │          │
│          │  1. Ban user permanently                  │          │
│          │  2. Terminate all sessions                │          │
│          │  3. Quarantine evidence (encrypted)       │          │
│          │  4. Create NCMEC report record            │          │
│          │  5. Submit to NCMEC CyberTipline          │          │
│          │  6. Log CRITICAL alert                    │          │
│          └───────────────────────────────────────────┘          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘


PERCEPTUAL HASHING (Why PhotoDNA Works)
───────────────────────────────────────
Problem: CSAM is often modified (resized, color-shifted, cropped)
        Standard hashing (MD5/SHA-256) fails - 1 pixel change = new hash

PhotoDNA Algorithm:
  1. Resize image to standard size (64×64)
  2. Convert to grayscale
  3. Compute discrete cosine transform (DCT)
  4. Quantize into binary fingerprint
  5. Output: ~144-bit hash stable across visual modifications

Result: Catches variants of known CSAM even with significant edits


NCMEC CYBERTIPLINE REPORTING
────────────────────────────
Integration with NCMEC CyberTipline API for mandatory reporting.

Report Submission (XML):
  1. Open report with incident data
  2. Upload file details (hash, metadata)
  3. Finalize report

Report Contents:
  • Incident timestamp
  • User identifiers and IP address
  • File details (hash, metadata)
  • Detection method

Retry Logic: Exponential backoff with configurable attempts
Failure Handling: Queued for retry with status tracking

[Specific endpoints and configuration redacted for security]


EVIDENCE QUARANTINE
───────────────────
Development: Local encrypted storage
Production:  Cloud storage with encryption + immutable object retention

Retention: Compliant with legal requirements

Metadata stored:
  • Unique identifiers
  • Detection method
  • Report status
  • Timestamps

[Specific storage configuration redacted for security]


FAIL-CLOSED SCENARIOS
─────────────────────
PhotoDNA timeout           → Quarantine + ban
PhotoDNA gateway error     → Quarantine + ban
Hash calculation failure   → Quarantine + ban
NCMEC submission failure   → Ban (report retried)
Storage write failure      → Ban + alert admin

Principle: Better to ban innocent user than allow CSAM uploader
```

### 17.5 Content Moderation Detectors

```
TEXT DETECTION SERVICES (12 Specialized Detectors)
══════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│  1. LANGUAGE FILTER                                             │
│     Profanity, explicit language detection                      │
│     Context-aware (negation detection, safe contexts)           │
│     Severity varies by intensity level                          │
├─────────────────────────────────────────────────────────────────┤
│  2. BULLYING/HARASSMENT DETECTOR                                │
│     Keywords: "loser", "stupid", "ugly", "kys"                  │
│     Grooming patterns in adult-to-minor conversations           │
│     Social media probing ("what's your snap")                   │
│     Risky behavior patterns (drugs, underage drinking)          │
├─────────────────────────────────────────────────────────────────┤
│  3. EXTORTION/BLACKMAIL DETECTOR                                │
│     Threats, demands for money/nudes                            │
│     Isolation tactics ("meet alone", "parents aren't home")     │
├─────────────────────────────────────────────────────────────────┤
│  4. SELF-HARM DETECTION                                         │
│     Suicide mentions, cutting/injury patterns                   │
│     Crisis escalation indicators                                │
├─────────────────────────────────────────────────────────────────┤
│  5. SCAM DETECTOR                                               │
│     Financial fraud patterns                                    │
│     Romance scam detection                                      │
│     Contributes to heuristic risk score                         │
├─────────────────────────────────────────────────────────────────┤
│  6. PSYCHOLOGICAL MANIPULATION DETECTOR                         │
│     Trust exploitation ("you can trust me")                     │
│     Gaslighting patterns                                        │
│     Isolation reinforcement                                     │
├─────────────────────────────────────────────────────────────────┤
│  7. LINK INTELLIGENCE SYSTEM                                    │
│     Malware domain detection                                    │
│     Adult content blocking (for minors)                         │
│     Phishing evaluation                                         │
│     Legal block list (HTTP 451: darkweb, CSAM sites)            │
│     URL shortener risk assessment                               │
├─────────────────────────────────────────────────────────────────┤
│  8. INJECTION DETECTOR                                          │
│     SQL injection patterns                                      │
│     Command injection detection                                 │
│     XSS payload blocking                                        │
│     Always: REJECT + strike penalty                             │
├─────────────────────────────────────────────────────────────────┤
│  9. GRACE PERIOD FILTER (Early Conversation Window)             │
│     Omegle-style garbage: "M 20 horny", "asl?"                  │
│     Age/sex/location solicitation                               │
│     Immediate rejection with strike penalty                     │
├─────────────────────────────────────────────────────────────────┤
│  10. LOW-EFFORT STARTER FILTER                                  │
│      One-word openers ("hi", "hey")                             │
│      Copy-paste spam detection                                  │
│      Activates grace period                                     │
├─────────────────────────────────────────────────────────────────┤
│  11. CONVERSATION PATTERN DETECTOR (Cross-Message)              │
│      Grooming sequences: info → secrecy → isolation → meeting   │
│      Trust manipulation patterns                                │
│      Age compliments ("mature for your age")                    │
│      Redis-backed history (survives restarts)                   │
├─────────────────────────────────────────────────────────────────┤
│  12. ESCALATION DETECTOR (Temporal Analysis)                    │
│      Tracks "hey" → "how old?" → "snap?" progression            │
│      Multi-message sequence analysis                            │
│      Grace period time-based thresholds                         │
└─────────────────────────────────────────────────────────────────┘


HEURISTIC RISK SCORING
──────────────────────
Multiple detectors contribute to composite risk score:

Risk Categories (weighted by severity):
  • Scam patterns
  • Manipulation tactics
  • Spam indicators
  • Self-harm content
  • Extortion attempts
  • Bullying behavior
  • Grooming patterns

Actions based on composite score:
  • High score: REJECT (block message)
  • Medium score: FLAG (admin review)
  • Low score: ACCEPT

[Specific weights and thresholds redacted for security]


IMAGE DETECTION SERVICES
════════════════════════

1. CSAM Detection (PhotoDNA + Hash DB) - See section 17.4

2. AWS Rekognition (Production Only)
   Detects 13+ categories with configurable confidence thresholds:

   Detection Categories:
   • Explicit Nudity & Sexual Activity
   • Suggestive content
   • Violence & Gore
   • Weapons
   • Self-Harm indicators
   • Drug-related content
   • Hate Symbols & Extremism

   [Specific confidence thresholds redacted for security]

   Development: Bypassed with heuristics

3. GIF/Sticker Safety
   Tenor API integration
   Known unsafe GIF ID blacklist
   Per-user GIF blocking
```

### 17.6 Anti-Bot and Anti-Spam System

```
SPAM DETECTION ARCHITECTURE
═══════════════════════════

GRACE PERIOD SYSTEM (Time-Limited New Match Protection)
───────────────────────────────────────────────────────
Purpose: Catch Omegle-style bot spam before moving to next victim

Detection Layer 1 - Immediate Patterns:
  • Age/sex declarations: "M 30", "f18", "asl?"
  • Sexual solicitation: "horny", "dtf", "send pics"
  • Social media harvesting: "snap?", "got insta?", "discord?"
  • OnlyFans/monetization: "onlyfans", "fansly"
  • Payment requests: "cashapp", "venmo", "paypal"
  • Zero-width character obfuscation

Single match = IMMEDIATE REJECT

Detection Layer 2 - Multi-Message Escalation:
  Tracks gradual escalation across messages:

  Monitored Patterns (scored by risk):
  • Age questions
  • Location questions
  • Personal info asks
  • Social media asks
  • Contact info asks
  • Marketing/promotional content
  • Payment solicitations

  Features:
  • Early message multiplier (stricter on new conversations)
  • Cumulative scoring with caps
  • Threshold-based actions (REJECT/FLAG/ACCEPT)

  [Specific scoring values redacted for security]


PREDATORY BEHAVIOR DETECTION (Cross-Message)
────────────────────────────────────────────
Analyzes message history for grooming patterns:

Detected Pattern Categories:
  • Rapid personal questions (information gathering)
  • Secrecy building ("just between us" patterns)
  • Isolation tactics (attempts to meet alone)
  • Trust manipulation (false rapport building)
  • Age-inappropriate compliments

Risk Assessment:
  • Individual patterns scored by severity
  • Dangerous combinations trigger multipliers
  • Combined score determines action (flag/reject)

[Specific scoring thresholds redacted for security]


SPAM ENFORCEMENT POLICY
───────────────────────
Trigger: Low-effort filter exceeds threshold in time window with multiple targets

Action Escalation:
  • 1st violation: Short temporary timeout
  • 2nd violation: Longer temporary timeout
  • 3rd+ violation: Permanent ban

Report-Based Enforcement:
  Trigger: Multiple unique users report same user for spam
  Same escalation as above

[Specific thresholds redacted for security]


ANTI-OBFUSCATION MEASURES
─────────────────────────
Zero-width character removal:
  U+200B (zero-width space)
  U+200C (zero-width non-joiner)
  U+200D (zero-width joiner)
  U+FEFF (BOM)
  U+2060 (word joiner)

Unicode normalization: NFKC + NFKD
Homoglyph conversion: 0→o, 1→i, 3→e, 4→a, 5→s, 7→t, @→a, $→s
Fuzzy matching: RapidFuzz (configurable threshold)
```

### 17.7 Report and Strike System

```
USER REPORTING FLOW
═══════════════════

Report Categories (11 types):
  INAPPROPRIATE_CONTENT, INAPPROPRIATE_BEHAVIOR, SPAM, HARASSMENT,
  IMPERSONATION, CHAT_MESSAGE, USER_PROFILE, SPAM_SCAM, FAKE_PROFILE,
  HATE_SPEECH, OTHER

Data Captured:
  • Report ID (UUID)
  • Reporter ID + age
  • Reported user ID + age
  • Report type + reason text (NOT filtered - must quote offensive content)
  • Timestamp

Auto-Actions When Minor Reports Adult:
  1. Auto-unfriend reported user
  2. Evaluate minor safety flags
  3. Trigger spam enforcement checks


REPORT STATUS LIFECYCLE
───────────────────────

  PENDING → INVESTIGATING → IN_REVIEW → RESOLVED
                                        ├─ RESOLVED_NO_ACTION
                                        └─ RESOLVED_ACTION_TAKEN


STRIKE SYSTEM (Redis-Backed)
════════════════════════════

Why Redis?
  • Atomic Lua scripts prevent race conditions (multi-worker)
  • Ephemeral data with TTL (strikes auto-expire)
  • O(1) lookups during chat
  • Cluster-safe with hash tags

Strike Escalation:
  • Low strike count: Short temporary ban
  • Medium strike count: Longer temporary ban
  • High strike count: Permanent ban + admin review

Properties:
  • Strikes expire after inactivity period (cooldown)
  • Rate limiting prevents malicious strike flooding
  • Graduated response based on violation severity

[Specific thresholds redacted for security]


BAN STATUS API
──────────────
Returns ChatBanStatusDict:
  {
    is_banned: bool
    is_permanent: bool
    expires_at: Optional[float]     (Unix timestamp)
    seconds_remaining: Optional[int]
    message: str                    (human-readable)
  }

Ban Duration Messages:
  Days:    "You are banned for X day(s)..."
  Hours:   "You are banned for X hour(s)..."
  Minutes: "You are banned for X minute(s)..."


APPEAL PROCESS
══════════════

User-Initiated Appeal:
  1. User submits: user_id, ban_reason, appeal_reason, context
  2. Validation: Must be banned, no pending appeal
  3. Status: PENDING

Admin Review:
  1. Access pending appeals queue (paginated)
  2. View user info (username, email, account age, minor status)
  3. Decision: APPROVED or DENIED
  4. Add notes
  5. Optional: Auto-unban if approved

Appeal Outcomes:
  APPROVED: Appeal reviewed, decision approved
    • User optionally unbanned (ban flag removed)
    • Strike count remains (but ban lifted)

  DENIED: Appeal reviewed, decision denied
    • Ban remains in effect
    • User can resubmit (after cooldown)


CROSS-AGE FRIEND REQUEST TRACKING
─────────────────────────────────
Purpose: Detect adults mass-friending minors

Behavior:
  • Tracks friend requests from adults to minors
  • Flags users exceeding threshold for admin review
  • Time-windowed tracking with automatic expiry

[Specific thresholds redacted for security]
```

### 17.8 Rate Limiting Architecture

```
DISTRIBUTED RATE LIMITING
═════════════════════════

Challenge: Multiple workers (5+), shared rate limits
Solution: Redis + Lua scripts for atomic operations

SLIDING WINDOW ALGORITHM
────────────────────────
Let W = window size (configured)
Let L = limit (configured)
Let T = current timestamp

For each request:
  1. Remove entries older than (T - W)
  2. Count remaining entries
  3. If count < L: allow, add T to set
  4. If count >= L: reject

All steps execute atomically in Lua:

  ZREMRANGEBYSCORE key 0 (now - window)  -- prune old
  count = ZCARD key                       -- count
  if count < limit then
    ZADD key now now                      -- record
    EXPIRE key window                     -- TTL
    return ALLOWED
  else
    return REJECTED
  end


LYNK VELOCITY LIMITING (In-App Currency)
────────────────────────────────────────
Purpose: Prevent rapid-fire transaction abuse

Detection Methods:
  • Per-minute transaction limits
  • Rapid-fire pattern detection (short burst windows)
  • Graduated response (flag → review → block)

Properties:
  • Atomic check-and-increment via Lua scripts
  • Fail-closed: Redis errors reject transaction
  • Flagged users tracked with auto-expiring keys

[Specific thresholds redacted for security]


WEBSOCKET MESSAGE RATE LIMITING
───────────────────────────────
Parameters per endpoint:
  max_requests: Maximum per window
  window_seconds: Time window
  burst_size: Allow bursts
  backoff_multiplier: Exponential on violations
  max_backoff: Cap on delay

Exponential Backoff:
  delay = min(window × multiplier^violations, max_backoff)


FAIL-CLOSED GUARANTEES
──────────────────────
All rate limiting fails-closed:
  Redis connection error → Reject request
  Invalid key format     → Reject request
  Script failure         → Reject request

Rationale: Brief blocking during outage < allowing abuse
```

---

## 18. Security Architecture

The Cortex Architecture implements a defense-in-depth security model with six
distinct layers. Each layer operates independently—if one layer fails, subsequent
layers continue to provide protection. The core security maxim: protect minors
from predatory exploitation first, then address all other threats.

### 18.1 Defense-in-Depth Architecture

```
THE SIX-LAYER SECURITY MODEL
════════════════════════════

Request Flow Through Security Layers:

    ┌───────────────────────────────────────────────────────────────┐
    │  LAYER 1: GATEWAY/MIDDLEWARE (First Line)                     │
    │                                                               │
    │  Access Protection                                            │
    │    • IP/origin validation                                     │
    │    • Bot detection (scanner, fuzzer, sqlmap patterns)         │
    │    • User-agent analysis                                      │
    │    • Request size limits (configured per-endpoint)             │
    │                                                               │
    │  Security Headers                                             │
    │    • CSP: default-src 'self', script-src 'self'               │
    │    • X-Frame-Options: DENY                                    │
    │    • X-Content-Type-Options: nosniff                          │
    │    • HSTS: max-age=31536000 (1 year)                          │
    │                                                               │
    │  If fails → 403/451 returned, request never reaches app       │
    └───────────────────────────────────────────────────────────────┘
                                    ↓
    ┌───────────────────────────────────────────────────────────────┐
    │  LAYER 2: AUTHENTICATION/AUTHORIZATION                        │
    │                                                               │
    │  Brute Force Protection                                       │
    │    • Limited attempts per time window per account             │
    │    • Progressive delays (exponential backoff)                 │
    │    • CAPTCHA trigger after failures                           │
    │    • Email notification on lockout                            │
    │                                                               │
    │  Session Security                                             │
    │    • High-entropy session identifiers                         │
    │    • Short-lived access tokens                                │
    │    • Device fingerprint binding (soft check)                  │
    │    • Geographic velocity detection                            │
    │                                                               │
    │  If fails → User locked out or requires verification          │
    └───────────────────────────────────────────────────────────────┘
                                    ↓
    ┌───────────────────────────────────────────────────────────────┐
    │  LAYER 3: INPUT VALIDATION                                    │
    │                                                               │
    │  Content Length Limits                                        │
    │    • All user inputs have enforced maximum lengths            │
    │    • Field-specific limits (messages, usernames, bios)        │
    │                                                               │
    │  Pattern Detection                                            │
    │    • SQL injection patterns                                   │
    │    • Cross-site scripting (XSS) patterns                      │
    │    • Command injection patterns                               │
    │                                                               │
    │  If fails → Request rejected, suspicious input logged         │
    └───────────────────────────────────────────────────────────────┘
                                    ↓
    ┌───────────────────────────────────────────────────────────────┐
    │  LAYER 4: CONTENT MODERATION (Safety Engine)                  │
    │                                                               │
    │  Text Safety (12 detectors)                                   │
    │  Image Safety (AWS Rekognition + PhotoDNA)                    │
    │  Link Intelligence (safe/adult/dangerous/phishing)            │
    │                                                               │
    │  Outcomes:                                                    │
    │    • ACCEPT → Content allowed                                 │
    │    • ACCEPT_WITH_FLAG → Allowed, flagged for review           │
    │    • REJECT → Blocked, strikes recorded                       │
    │                                                               │
    │  (See Section 17 for detailed Safety Systems)                 │
    └───────────────────────────────────────────────────────────────┘
                                    ↓
    ┌───────────────────────────────────────────────────────────────┐
    │  LAYER 5: BEHAVIORAL ANALYSIS                                 │
    │                                                               │
    │  Conversation Pattern Detection                               │
    │    • Grooming escalation patterns                             │
    │    • Age falsification tracking                               │
    │    • Cross-chat pattern analysis                              │
    │                                                               │
    │  Strike Tracking (Redis + Lua atomic)                         │
    │    • Per-user strike accumulation                             │
    │    • Automatic ban escalation                                 │
    │                                                               │
    │  If fails → Temp ban, escalation for review                   │
    └───────────────────────────────────────────────────────────────┘
                                    ↓
    ┌───────────────────────────────────────────────────────────────┐
    │  LAYER 6: RELATIONSHIP ENFORCEMENT                            │
    │                                                               │
    │  Interaction Checks                                           │
    │    • Interaction permission → Block/trust verification        │
    │    • Call permission → Safety status check                    │
    │    • Game permission → Game access verification               │
    │                                                               │
    │  Matching Exclusion                                           │
    │    • Candidate filtering for safe matching                    │
    │    • Excludes blocked, flagged, previously matched            │
    │                                                               │
    │  If fails → Interaction denied, users isolated                │
    └───────────────────────────────────────────────────────────────┘

Cascade Failure Design:
═══════════════════════

If Layer N fails, Layer N+1 continues independently.

Example: Layer 3 bypassed (SQL injection in input)
  → Layer 4 still sanitizes content (catches <script>)
  → Layer 5 still tracks behavior (catches patterns)
  → Database layer uses parameterized queries (final defense)

Philosophy: Fail-closed. When uncertain, restrict. Deny by default.
```

### 18.2 Threat Model and Attack Surface

```
THREAT CATEGORIES
═════════════════

┌─────────────────────────────────────────────────────────────────┐
│  PREDATORY THREATS (CRITICAL PRIORITY)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Attack: Adults posing as minors to groom/exploit               │
│                                                                 │
│  Defenses:                                                      │
│    • Age Falsification Detector (5 methods)                     │
│    • Conversation Pattern Detector (grooming escalation)        │
│    • Minor Protection System (empowering warnings)              │
│    • Matching Safety Checks (age gap analysis)                  │
│                                                                 │
│  Detection Signals:                                             │
│    • Adult life references (jobs, apartments, bills)            │
│    • Language pattern inconsistencies                           │
│    • Grooming speed (rushing to inappropriate topics)           │
│    • Outdated slang usage                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  INJECTION ATTACKS                                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SQL Injection                                                  │
│    Attack: Malicious SQL in user input                          │
│    Defense: Parameterized queries (asyncpg $1, $2 placeholders) │
│             Semantic validation detects SQL keywords            │
│             No string concatenation for data values             │
│                                                                 │
│  Command Injection                                              │
│    Attack: Shell commands in user input                         │
│    Defense: List-based subprocess args (never shell=True)       │
│             User input never reaches subprocess                 │
│             Whitelisted service names for admin ops             │
│                                                                 │
│  XSS (Cross-Site Scripting)                                     │
│    Attack: Malicious scripts in user content                    │
│    Defense: HTML sanitization (bleach library)                  │
│             CSP headers (no inline scripts)                     │
│             Unicode normalization (homograph prevention)        │
│             Zero-width character removal                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  SESSION ATTACKS                                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CSRF (Cross-Site Request Forgery)                              │
│    Attack: Unauthorized actions via user's session              │
│    Defense: Synchronizer Token Pattern (HMAC-SHA256 signed)     │
│             SameSite=Strict cookies (production)                │
│             Origin/Referer header validation                    │
│             Mobile exempt (Bearer token auth)                   │
│                                                                 │
│  Session Hijacking                                              │
│    Attack: Stealing user session tokens                         │
│    Defense: HTTP-only cookies (no JS access)                    │
│             Secure flag (HTTPS only)                            │
│             Token rotation on refresh                           │
│             Device fingerprint soft-binding                     │
│                                                                 │
│  Session Fixation                                               │
│    Attack: Force victim to use attacker's session               │
│    Defense: Session regeneration on login                       │
│             Old session immediately invalidated                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  ACCESS CONTROL ATTACKS                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  IDOR (Insecure Direct Object Reference)                        │
│    Attack: Access other users' data by guessing IDs             │
│    Defense: Centralized access validator on every access        │
│             256-bit random IDs (not sequential)                 │
│             Ownership validation in queries                     │
│             Return 404 (not 403) to prevent enumeration         │
│                                                                 │
│  Privilege Escalation                                           │
│    Attack: Gaining unauthorized admin permissions               │
│    Defense: Role hierarchy (SUPER_ADMIN > ADMIN > USER)         │
│             Role checks at gateway layer                        │
│             Audit logging of all admin actions                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

ATTACK SURFACE ANALYSIS
═══════════════════════

Exposed Endpoint Categories:

  Authentication (Pre-auth, no CSRF)
    • Login, registration, password reset

  User Data (Authenticated)
    • Profile operations, reporting

  Messaging (Real-time)
    • Chat messages via REST and WebSocket

  Media (Size-limited)
    • File uploads with size limits
    • Access-controlled file serving

  Admin (Isolated backend)
    • Separate origin validation

[Specific endpoint paths redacted for security]
```

### 18.3 SQL Injection Prevention

```
QUERY CONSTRUCTION MODEL
════════════════════════

Rule: 100% parameterized queries. No exceptions.

┌─────────────────────────────────────────────────────────────────┐
│  CORRECT: Parameterized query                                   │
│                                                                 │
│    query = "SELECT * FROM users WHERE user_id = $1"             │
│    await conn.execute(query, user_id)                           │
│                                                                 │
│  The user_id is passed as a separate argument, never            │
│  interpolated into the SQL string.                              │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  FORBIDDEN: String concatenation                                │
│                                                                 │
│    query = f"SELECT * FROM users WHERE user_id = {user_id}"     │
│                                                                 │
│  This pattern is banned throughout the codebase.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Dynamic Query Building (When Necessary):
────────────────────────────────────────

For admin filtering, column names can be dynamic (they're schema,
not data), but values must always use placeholders:

  allowed_fields = {"username", "email", "status"}  # Whitelist

  if field not in allowed_fields:
      raise ValueError("Invalid field")

  # Column name from whitelist, value parameterized
  query = f"SELECT * FROM users WHERE {field} = $1"
  await conn.execute(query, value)

Defense-in-Depth: Semantic Validation
─────────────────────────────────────

Even with parameterized queries, input is scanned for SQL patterns:

  Checks include:
    • SQL keywords (SELECT, INSERT, DROP, UNION, etc.)
    • Comment syntax
    • Logic operators
    • Time-based injection patterns
    • Schema traversal attempts

This provides:
  • Better error messages to users
  • Early rejection before database round-trip
  • Logging of attempted injection for security monitoring
```

### 18.4 XSS Prevention

```
HTML SANITIZATION PIPELINE
══════════════════════════

All user-generated content passes through multi-layer sanitization:

┌─────────────────────────────────────────────────────────────────┐
│  LAYER 1: HTML SANITIZATION (bleach library)                    │
│                                                                 │
│  Allowed Tags (Whitelist):                                      │
│    b, i, u, s, em, strong, code, pre, kbd, mark, sub, sup       │
│    ul, ol, li, blockquote, p, br, hr, h3-h6, a, span, div       │
│                                                                 │
│  Allowed Attributes:                                            │
│    href, title (links)                                          │
│    class (validated against pattern)                            │
│    cite (quotes)                                                │
│                                                                 │
│  Allowed Protocols:                                             │
│    http, https, mailto                                          │
│                                                                 │
│  Everything else is STRIPPED.                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 2: UNICODE NORMALIZATION                                 │
│                                                                 │
│  Homograph Attack Prevention:                                   │
│    Normalizes to NFKC form (decompose + compose compatible)     │
│    "pаypal.com" (Cyrillic 'а') → detected as obfuscation        │
│                                                                 │
│  Zero-Width Character Removal:                                  │
│    U+200B (zero-width space)                                    │
│    U+200C (zero-width non-joiner)                               │
│    U+200D (zero-width joiner)                                   │
│    U+2060 (word joiner)                                         │
│    U+FEFF (zero-width no-break space)                           │
│                                                                 │
│  Bidirectional Control Removal:                                 │
│    U+202E (right-to-left override) - Most dangerous             │
│    Other directional markers (LRM, RLM, etc.)                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 3: LINK SECURITY                                         │
│                                                                 │
│  All <a> tags automatically receive:                            │
│    rel="noopener noreferrer"                                    │
│                                                                 │
│  Prevents reverse tabnabbing:                                   │
│    • Linked page cannot access window.opener                    │
│    • Cannot redirect original tab                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 4: FINAL VALIDATION                                      │
│                                                                 │
│  Dangerous Pattern Check (if any survive sanitization):         │
│    • javascript: protocol                                       │
│    • vbscript: protocol                                         │
│    • on\w+\s*= (event handlers)                                 │
│                                                                 │
│  If detected: entire content is HTML-escaped (fail-closed)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

CONTENT SECURITY POLICY
═══════════════════════

CSP Header Directives:

  default-src      'self'              ← Same-origin only
  script-src       'self'              ← No inline scripts
  style-src        'self' 'unsafe-inline' ← React CSS-in-JS
  img-src          'self' data: + CDNs ← Controlled image sources
  media-src        'self' blob:        ← WebRTC blob URLs
  connect-src      'self' wss: https:  ← API + WebSocket
  object-src       'none'              ← Block Flash/Java
  frame-ancestors  'none'              ← Prevent clickjacking

Production-Only:
  upgrade-insecure-requests            ← Auto HTTP→HTTPS
  block-all-mixed-content              ← Block HTTP on HTTPS
```

### 18.5 CSRF Prevention

```
SYNCHRONIZER TOKEN PATTERN
══════════════════════════

CSRF Token Structure:
─────────────────────

[Specific token structure redacted for security]

Properties:
  • High entropy random component
  • Cryptographically signed (HMAC)
  • Context-bound (client fingerprinting)
  • Time-limited validity

Token Lifecycle:
────────────────

  ┌────────────────┐     ┌────────────────┐     ┌────────────────┐
  │   GENERATION   │ →   │    STORAGE     │ →   │   VALIDATION   │
  └────────────────┘     └────────────────┘     └────────────────┘
         │                      │                      │
  On GET request         HTTP-only cookie        Compare cookie
  for authenticated      + response header       with header token
  users                                          (constant-time)

Validation Process:
───────────────────

  1. Extract token from cookie (csrf_token)
  2. Extract token from header (X-CSRF-Token) or JSON body
  3. Constant-time comparison: secrets.compare_digest()
  4. Decode and verify HMAC signature
  5. Check token age within allowed window
  6. Soft-check IP/UA binding (log mismatches)
  7. Verify token version supported

CSRF-Exempt Paths:
──────────────────

  • Pre-auth endpoints (no session to forge)
  • System health/documentation endpoints

[Specific paths redacted for security]

Protected Methods: POST, PUT, DELETE, PATCH

COOKIE CONFIGURATION
════════════════════

Access Token Cookie:
  Name:       access_token
  HttpOnly:   true          ← Prevents XSS token theft
  Secure:     true          ← HTTPS only
  SameSite:   lax           ← CSRF protection + OAuth compat
  Path:       /             ← All routes
  Max-Age:    JWT lifetime + buffer (for refresh)

Refresh Token Cookie:
  Name:       refresh_token
  HttpOnly:   true
  Secure:     true
  SameSite:   lax
  Path:       Scoped to auth routes only
  Max-Age:    Long-lived (matches refresh token lifetime)

CSRF Token Cookie:
  Name:       csrf_token
  HttpOnly:   true
  Secure:     true (production)
  SameSite:   strict (production)  ← Strongest CSRF protection
  Max-Age:    Short-lived (hours)

Mobile Client Bypass:
─────────────────────

Mobile apps use Bearer token authentication, not cookies.
CSRF is a browser-specific attack, so mobile is exempt.

Detection: User-Agent patterns (Dart/, okhttp, CFNetwork)
```

### 18.6 Authentication Security

```
PASSWORD HASHING
════════════════

Algorithm: Argon2id v19 (OWASP 2024+ recommended)

Properties:
  • Memory-hard: High memory cost resists GPU/ASIC attacks
  • Time cost: Multiple iterations for brute force resistance
  • Parallelism: Multi-threaded processing
  • Salt: Unique per password (auto-generated)
  • Output: Binary hash

Why Argon2id?
  • Memory-hard: Resists GPU/ASIC attacks
  • Side-channel resistant: Combines Argon2i + Argon2d
  • Modern standard: Winner of Password Hashing Competition

[Specific parameters redacted for security]

Timing Attack Mitigation:
─────────────────────────

When user doesn't exist, hash comparison still runs against
a dummy hash to prevent timing oracle attacks:

  if user_exists:
      verify(stored_hash, provided_password)
  else:
      verify(DUMMY_HASH, provided_password)  ← Same timing

TOKEN ARCHITECTURE
══════════════════

┌─────────────────────────────────────────────────────────────────┐
│  ACCESS TOKEN (JWT)                                             │
├─────────────────────────────────────────────────────────────────┤
│  Algorithm:  HS256 (HMAC-SHA256)                                │
│  Lifetime:   Short-lived (minutes for permanent users)          │
│              Extended for temporary users                       │
│                                                                 │
│  Claims:                                                        │
│    sub:        User ID                                          │
│    iat:        Issued at                                        │
│    exp:        Expiration                                       │
│    jti:        Unique token ID (UUID v4)                        │
│    session_id: Persists across refreshes                        │
│    aud:        "welynk:users"                                   │
│    iss:        "user_backend"                                   │
│    role:       user | admin | super_admin                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  REFRESH TOKEN                                                  │
├─────────────────────────────────────────────────────────────────┤
│  Format:    256-bit random (secrets.token_urlsafe(32))          │
│  Storage:   SHA256 hash in database (not plaintext)             │
│  Lifetime:  Long-lived (weeks/months)                           │
│                                                                 │
│  Metadata:                                                      │
│    family_id:   Token family for rotation tracking              │
│    session_id:  Links to access tokens                          │
│    ip_address:  Client IP at issuance                           │
│    user_agent:  Client UA at issuance                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

REFRESH TOKEN ROTATION
══════════════════════

Token Family Pattern with Compromise Detection:

  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
  │ Token A │ →  │ Token B │ →  │ Token C │ →  │ Token D │
  └─────────┘    └─────────┘    └─────────┘    └─────────┘
       │              │              │              │
       └──────────────┴──────────────┴──────────────┘
                     Same family_id

Refresh Flow:
  1. Client sends refresh_token (raw 256-bit string)
  2. Server hashes: SHA256(refresh_token)
  3. Lookup in database by hash
  4. Validate: not revoked, not expired, user not revoked
  5. Issue NEW refresh token (same family)
  6. REVOKE old token immediately
  7. Issue new access token

Compromise Detection:
─────────────────────

If two refresh requests use the same token:

  Attacker stole token → uses it → gets new token
  Legitimate user → uses same token → COLLISION

  Detection: Token already revoked when legitimate user tries
  Action: Revoke ENTIRE token family
  Result: Attacker's new token also invalidated, user must re-auth

Distributed Locking:
────────────────────

Redis lock during refresh prevents race conditions:
  Lock key: [key pattern redacted]
  Timeout: Configured (fail-closed if exceeded)
  If locked: Request rejected (fail-closed)
```

### 18.7 IDOR Prevention

```
RESOURCE ACCESS VALIDATION
══════════════════════════

Centralized validator handles all access checks:

  • Profile access validation
  • Chat access validation
  • Admin access validation

PROFILE ACCESS RULES
════════════════════

  Self-access:      Always allowed (view only)

  Other user:       Check target exists and not deleted
                    Check not suspended
                    Apply privacy blocking (limited data if blocked)

  On denial:        Return 404 (not 403)  ← Prevents enumeration

CHAT ACCESS RULES
═════════════════

  Participant check:
    1. Get chat details
    2. Extract participant_ids
    3. Verify requester.id in participant_ids
    4. If not: return 404

  Status checks:
    • Deleted chat? → 404
    • Read-only + write access? → 403
    • Archived? → 403

ADMIN ACCESS RULES
══════════════════

  Role Hierarchy:
    SUPER_ADMIN > ADMIN > USER

  SUPER_ADMIN: All operations
  ADMIN: Resource-specific (no system config)
  USER: No admin access

  All admin attempts logged for audit.

ID GENERATION
═════════════

All resource IDs are cryptographically random, not sequential:

  User IDs:     256-bit random (secrets.token_urlsafe(32))
  Chat IDs:     256-bit random
  Session IDs:  128-bit random (UUID v4)
  Token JTIs:   128-bit random (UUID v4)

Why random?
  • Cannot guess next ID (enumeration prevention)
  • Cannot infer count or order (information leakage prevention)
  • Cannot brute-force (256-bit = 2^256 possibilities)

ERROR MASKING
═════════════

Authorization failures always return 404:

  WRONG: "You don't have permission to access this resource" (403)
         → Reveals resource exists

  RIGHT: "Resource not found or inaccessible" (404)
         → No information leakage
```

### 18.8 Rate Limiting Architecture

```
DISTRIBUTED RATE LIMITING
═════════════════════════

Challenge: Multiple Uvicorn workers must share rate limits.
           In-memory counters fail (Worker A counts N, Worker B counts N = 2N)

Solution: Redis + Lua scripts for atomic operations.

SLIDING WINDOW ALGORITHM
════════════════════════

For each request at time T:
  1. Remove timestamps older than (T - window_size)
  2. Count remaining timestamps
  3. If count < limit: allow, record T
  4. If count >= limit: reject

All steps execute atomically in single Lua script:

┌─────────────────────────────────────────────────────────────────┐
│  Storage: Redis Sorted Set (timestamps as scores)               │
│                                                                 │
│  Atomic Lua Script:                                             │
│  ──────────────────                                             │
│  1. Prune entries older than window                             │
│  2. Count remaining entries                                     │
│  3. If under limit: allow and record timestamp                  │
│  4. If over limit: reject                                       │
│                                                                 │
│  [Specific key patterns and script details redacted]            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

RATE LIMITS BY ACTION
═════════════════════

Rate limits are enforced per action type with varying thresholds:

WebSocket Events:
  • Connection attempts: Limited per second
  • Send message: Limited per second
  • Typing indicator: Limited per second
  • Game actions: Higher limit (real-time needs)
  • File upload: Lower limit (resource-intensive)

HTTP Endpoints:
  • General: Per-endpoint limits
  • Auth: Stricter limits to prevent brute force

Per-Recipient (Group Chat Bombing):
  • Per-user message limits prevent coordinated attacks

[Specific rate limit values redacted for security]

DDOS PROTECTION LAYERS
══════════════════════

Layer 1: CloudFront Edge
  • AWS Shield DDoS protection
  • Geographic distribution
  • Edge caching

Layer 2: Access Protection Middleware
  • Bot detection (scanner, fuzzer, sqlmap patterns)
  • Suspicious User-Agent (below minimum length)
  • Automation indicators

Layer 3: Rate Limiting
  • Per-user limits (authenticated)
  • Progressive delays (not hard blocks)
  • Self-healing stale connection cleanup

Layer 4: Connection Tracking
  • Maximum WebSocket connections per user enforced
  • Connection TTL with periodic heartbeat validation
  • Atomic cleanup via Lua scripts

FAIL-CLOSED STRATEGY
════════════════════

If Redis unavailable during rate limit check:
  → Request rejected (SECURITY > AVAILABILITY)

Rationale: Prevents bypass attacks during Redis outages.
```

### 18.9 Error Sanitization

```
ERROR SANITIZATION PRINCIPLE
════════════════════════════

Internal errors must NEVER reach users.

NEVER expose:                    ALWAYS expose:
  • Stack traces                   • User-friendly message
  • File paths                     • Error code for support
  • Database queries               • Actionable guidance
  • Internal service names
  • Version numbers
  • Configuration values

PATTERN-BASED REDACTION
═══════════════════════

95 sensitive patterns automatically redacted:

  File Paths:
    /app/backend/user.py          → [file_path]
    C:\Users\config.py            → [file_path]

  Database Errors:
    psycopg2.OperationalError     → "Database error"
    column "email" does not exist → "Invalid field"

  Stack Traces:
    Traceback (most recent call): → [removed]
    File "/app/file.py", line 123 → [removed]

  Credentials:
    postgresql://user:pass@host   → [database_url]
    Bearer eyJ...                 → [auth_token]

  Internal IPs:
    192.168.x.x                   → [internal_ip]
    10.x.x.x                      → [internal_ip]

  PII (minors compliance):
    user@example.com              → [email]
    123-45-6789                   → [ssn]

EXCEPTION TYPE MAPPING
══════════════════════

14 exception types → safe generic messages:

  ValidationError        → "Invalid input provided"
  DatabaseError          → "Database operation failed"
  ConnectionError        → "Service temporarily unavailable"
  TimeoutError           → "Operation timed out"
  PermissionError        → "Access denied"
  FileNotFoundError      → "Resource not found"
  ValueError             → "Invalid value provided"
  KeyError               → "Required field missing"

PYDANTIC VALIDATION SANITIZATION
════════════════════════════════

Pydantic returns exact input that failed validation.
This must be stripped to prevent attack payload leakage:

  BEFORE (dangerous):
    {"loc": ["email"], "input": "x' OR '1'='1"}  ← Leaks SQL payload

  AFTER (safe):
    {"loc": ["email"], "msg": "invalid email format"}

PII MASKING FOR LOGS
════════════════════

User IDs and emails masked before logging:

  User ID masking:
    → Hashed identifier (sufficient for correlation)

  Email masking:
    → Partial obfuscation (e.g., "u***@e***.com")

Sufficient for log correlation without PII exposure.
```

### 18.10 Data Encryption

```
FIELD-LEVEL ENCRYPTION
══════════════════════

Encrypted in Database:
  • Chat Data Encryption Keys (DEKs)
  • User PII fields
  • 2FA secrets (separate key)

Algorithm: AES-256-GCM (Authenticated Encryption)
  • 256-bit key
  • 12-byte nonce (per encryption)
  • 16-byte authentication tag
  • Base64 encoding for storage

Key Derivation: PBKDF2 with high iteration count

ENVELOPE ENCRYPTION PATTERN
═══════════════════════════

  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │    Master Key (AWS KMS)                                     │
  │         │                                                   │
  │         │ encrypts                                          │
  │         ↓                                                   │
  │    Data Encryption Key (DEK)                                │
  │         │                                                   │
  │         │ encrypts                                          │
  │         ↓                                                   │
  │    User Data                                                │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘

Benefits:
  • Master key never leaves HSM
  • DEK rotation without re-encrypting all data
  • Per-user or per-resource DEKs possible

KEY MANAGEMENT
══════════════

Production: AWS KMS (HSM-backed)
  • Hardware security module protection
  • Automatic key rotation
  • CloudTrail audit logging
  • IAM-based access control

Development: Local KMS Provider
  • In-memory keys
  • Never used in production

Multi-Version Support:
  • Track current key version
  • Old versions retained for decryption
  • Seamless rotation with overlap period

CACHE ENCRYPTION
════════════════

Redis cache uses separate encryption layer:
  • Algorithm: Fernet (AES-128-CBC + HMAC-SHA256)
  • Encrypts DEKs before caching
  • Graceful degradation: if key not set, sensitive data not cached

TLS CONFIGURATION
═════════════════

All Production Traffic:
  • TLS 1.3 required
  • HTTPS enforcement via HSTS
  • max-age=31536000 (1 year)
  • includeSubDomains
  • preload

Cookie Security:
  • Secure flag: true (production)
  • HttpOnly flag: true
  • SameSite: strict (CSRF cookies)
```

### 18.11 File Upload Security

```
ALLOWED FILE TYPES (WHITELIST)
══════════════════════════════

Images:
  image/jpeg   → .jpg, .jpeg
  image/png    → .png
  image/webp   → .webp
  image/gif    → .gif

Video:
  video/mp4    → .mp4

Everything else: REJECTED

VALIDATION PIPELINE
═══════════════════

┌─────────────────────────────────────────────────────────────────┐
│  STEP 1: FILENAME VALIDATION                                    │
│                                                                 │
│  • Extract basename (strip path components)                     │
│  • Check extension against whitelist                            │
│  • Detect double-extension attacks (.jpg.exe)                   │
│  • Sanitize special characters (alphanumeric only)              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  STEP 2: SIZE VALIDATION                                        │
│                                                                 │
│  Per-Type Limits:                                               │
│    Images: [configured limit]                                   │
│    Video:  [configured limit]                                   │
│                                                                 │
│  Subscriber-Aware:                                              │
│    Free users: [tier limit]                                     │
│    Subscribers: [tier limit]                                    │
│                                                                 │
│  Minimum: [configured] (too small = invalid)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  STEP 3: MAGIC BYTE VALIDATION                                  │
│                                                                 │
│  File Signatures:                                               │
│    JPEG:  \xff\xd8\xff                                          │
│    PNG:   \x89\x50\x4e\x47\x0d\x0a\x1a\x0a                      │
│    GIF:   GIF87a or GIF89a                                      │
│    WebP:  RIFF + WEBP at offset 8                               │
│    MP4:   ftyp at offset 4 with known brands                    │
│                                                                 │
│  Declared MIME type must match detected signature.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  STEP 4: MALICIOUS CONTENT SCAN                                 │
│                                                                 │
│  Patterns checked in file header:                               │
│    <script       javascript:     <iframe                        │
│    <object       <embed          <form                          │
│    eval(         <svg (can contain scripts)                     │
│                                                                 │
│  If detected: REJECTED                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  STEP 5: MEDIA-SPECIFIC VALIDATION                              │
│                                                                 │
│  Images (Pillow):                                               │
│    • Open and validate structure                                │
│    • Check dimensions (within allowed range)                    │
│    • Count frames (limit for GIFs)                              │
│    • Verify format matches MIME                                 │
│                                                                 │
│  Video:                                                         │
│    • Validate MP4 container (ftyp box)                          │
│    • Max duration: [configured limit]                           │
│    • Max resolution: [configured limit]                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  STEP 6: CSAM DETECTION                                         │
│                                                                 │
│  AWS Rekognition + PhotoDNA (async, non-blocking)               │
│                                                                 │
│  On detection:                                                  │
│    • User banned immediately                                    │
│    • Message quarantined                                        │
│    • NCMEC report filed                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

PATH TRAVERSAL PREVENTION
═════════════════════════

5-layer path traversal defense:

  1. Resolve with .resolve().relative_to(base_dir)
     Raises ValueError if path escapes base directory

  2. Reject symlinks explicitly
     if path.is_symlink(): raise 404

  3. Validate file exists
     if not path.exists() or not path.is_file(): raise 404

  4. Length limit
     if len(path) > 500: raise 400

  5. No leading slash
     if path.startswith("/"): raise 400

STORAGE ARCHITECTURE
════════════════════

Development: Local filesystem (data/media/)
Production: AWS S3 + CloudFront CDN

Access Control:
  • Profile pictures: Public if approved
  • Chat media: Authenticated + participant check
  • Group photos: Public (avatar behavior)
  • Store assets: Public, cached [configured TTL]
  • Temp files: Owner only, short-lived cache

All media served through authenticated endpoint:
  GET /api/v1/media/serve/{file_path}

No direct S3 access—all requests go through access control.
```

---

## 19. Analytics Engine

The analytics engine uses Exponentially Weighted Moving Average (EWMA) calculations
to measure behavioral depth over time. Recency-weighted metrics predict user
behavior better than lifetime averages. All metrics are privacy-first: aggregates
over raw data, no message content stored, PII excluded from timeseries.

### 19.1 Engagement Metrics

```
EXPONENTIALLY WEIGHTED MOVING AVERAGE (EWMA)
════════════════════════════════════════════

Core engagement uses EWMA to weight recent behavior more heavily:

  new_ewma = decay × current_ewma + (1 - decay) × new_value

With configurable decay factor:
  • Recent values weighted more heavily
  • Older values decay over sessions
  • EWMA adapts quickly when behavior changes

ENGAGEMENT FACTORS
══════════════════

┌─────────────────────────────────────────────────────────────────┐
│  MESSAGE LENGTH ENGAGEMENT (message_len_ewma)                   │
├─────────────────────────────────────────────────────────────────┤
│  Measures: Conversation depth via average message length        │
│                                                                 │
│  Formula:                                                       │
│    message_len_ewma = α × old_ewma + (1-α) × session_avg_len    │
│    [Decay factor α redacted]                                    │
│                                                                 │
│  Capped at configured maximum (prevents spam burst domination)  │
│  Range: 0 to configured cap                                     │
│                                                                 │
│  Interpretation:                                                │
│    High value → thoughtful conversations                        │
│    Low value → brief exchanges                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  RESPONSE GAP (response_gap_ewma_ms)                            │
├─────────────────────────────────────────────────────────────────┤
│  Measures: Median time between messages (responsiveness)        │
│                                                                 │
│  Formula:                                                       │
│    response_gap_ewma = α × old_ewma + (1-α) × median_gap_ms     │
│    [Decay factor α redacted]                                    │
│                                                                 │
│  Capped at configured maximum to prevent AFK domination         │
│  Range: 0 to configured max milliseconds                        │
│                                                                 │
│  Interpretation:                                                │
│    Low value → highly engaged, quick responses                  │
│    High value → casual, asynchronous conversation               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  SESSION COUNT (sessions_count)                                 │
├─────────────────────────────────────────────────────────────────┤
│  Measures: Lifetime session participation (no decay)            │
│  Used for: Experience matching in ALMA                          │
│  Type: Integer counter                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

SESSION QUALITY CLASSIFICATION
══════════════════════════════

Scored after configured delay from match creation:

  ┌───────────────────────────────────────────────┐
  │  Quality  │  Messages  │  Interpretation      │
  ├───────────────────────────────────────────────┤
  │  HIGH     │  Many      │  Engaged, deep       │
  │  MEDIUM   │  Some      │  Moderate engagement │
  │  LOW      │  Few       │  Light engagement    │
  │  NONE     │  0         │  No conversation     │
  └───────────────────────────────────────────────┘
  [Specific thresholds redacted - proprietary]

RELIABILITY METRICS
═══════════════════

  show_up_rate:     Probability user attends match (default 1.0)
  ghost_rate:       Proportion of sessions ended abruptly
  early_exit_rate:  Rate of premature session termination

These feed into ALMA behavioral matching [weight redacted - proprietary].
```

### 19.2 Retention Calculations

```
DAY-N RETENTION
═══════════════

Definition: Percentage of cohort users who return on Day N.

Formula:
                Users from cohort active on Day N
  R(N) = ─────────────────────────────────────────── × 100%
                   Total users in cohort

Standard Intervals:
  D1  = Day 1 retention  (immediate stickiness)
  D7  = Day 7 retention  (weekly habit formation)
  D30 = Day 30 retention (monthly engagement)
  D90 = Day 90 retention (long-term retention)

Activity Definition:
  User is "active" if ANY of:
    • Chat session created
    • Message sent
    • Game session started
    • Friend request sent/accepted
    • Account login

COHORT ANALYSIS
═══════════════

Weekly cohort bucketing (Monday-Sunday signup weeks):

  ┌──────────────────────────────────────────────────────────────┐
  │  Cohort      │ Size │  D1   │  D7   │  D30  │  D90          │
  ├──────────────────────────────────────────────────────────────┤
  │  2024-01-15  │ 1250 │ 65.2% │ 42.1% │ 18.3% │  8.5%         │
  │  2024-01-22  │ 1180 │ 61.8% │ 39.4% │ 17.1% │  7.9%         │
  │  ...         │ ...  │ ...   │ ...   │ ...   │  ...          │
  └──────────────────────────────────────────────────────────────┘

Interpretation:
  • Steep drop-off → poor initial experience
  • Curve stagnation → stable user base forming
  • High resurrection rate → viral/word-of-mouth growth

CHURN CALCULATION
═════════════════

                Users active last month but inactive this month
  Monthly Churn = ──────────────────────────────────────────────── × 100%
                           Active users last month

Churn Risk Signals:
  At-risk:     No activity for 7+ days
  Churned:     No activity for 14+ days
  Declining:   EWMA values dropping over time

Subscriber churn tracked separately (financial vs engagement).
```

### 19.3 Match Quality Metrics

```
MATCH SUCCESS INDICATORS
════════════════════════

Primary Metrics:

  1. Message Exchange (binary)
     Success if messages > 0

  2. Conversation Initiation Rate
                    Chats with messages
     Init Rate = ─────────────────────── × 100%
                      Total matches

     Healthy range: [target range redacted]

  3. Friending Rate
                    Users who added each other
     Friend Rate = ──────────────────────────── × 100%
                         Total matches

     Indicates match compatibility

INTERACTION SCORING
═══════════════════

Per-match interaction score:

  InteractionScore {
    user_id:            str
    other_user_id:      str
    score_user_to_other: float   (user's quality rating)
    score_other_to_user: float   (other's quality rating)
    chat_session_id:    str
    scored_at:          datetime
  }

Factors:
  • Message count (binary: conversation happened)
  • Conversation duration (how long chat active)
  • Response patterns (via median_response_gap_ms)
  • Mutual engagement (both users contributed)

ALMA Integration:
  Quality scores feed into SVD network analysis.
  Low-quality interactions (no messages) weighted less in embeddings.

SESSION COMPLETION TRACKING
═══════════════════════════

                    Sessions with status='COMPLETED'
  Completion Rate = ─────────────────────────────────────
                    Sessions in {COMPLETED, VOIDED}

  COMPLETED: Normal session end
  VOIDED:    Forcibly ended (timeout, error, abuse filter)

Voided sessions indicate technical issues or safety triggers.
```

### 19.4 Lynks Circulation Tracking

```
TRANSACTION TYPES
═════════════════

  store_purchase:     User → Store Item (spend)
  gift_purchase:      User A → User B (gift spend)
  gift_received:      User B (receive)
  referral_reward:    New user referred (earn)
  subscription_daily: Daily subscription grant
  subscription_premium: Premium purchase

Transaction Model:
  LynkTransaction {
    user_id:      str
    other_user_id: Optional[str]  (for gifts)
    type:         str
    amount:       int             (positive=earn, negative=spend)
    created_at:   datetime
    balance_after: int
  }

EARNING SOURCES
═══════════════

  Sources: IAP Purchase, Subscription, Referral, Game Rewards

  [Specific amounts and rates redacted - proprietary]

ECONOMY HEALTH METRICS
══════════════════════

  total_lynks_in_circulation:  SUM(user_lynk_balance)
  avg_lynk_balance:            Mean balance across users
  lynks_earned_today:          SUM(amount > 0) for today
  lynks_spent_today:           SUM(ABS(amount)) where amount < 0
  net_lynk_flow:               earned - spent

LYNK VELOCITY
═════════════

Measures speed of currency circulation:

                    Total_Transactions_This_Month
  Lynk Velocity = ───────────────────────────────── × 30 days
                   Total_Lynks_in_Circulation

Interpretation:
  Normal range:  Healthy economy (active spending)
  Below normal:  Stagnation (Lynks hoarded)
  Above normal:  Hyperactivity (possible bot farming)
[Specific ranges redacted - fraud detection thresholds]

FRAUD DETECTION
═══════════════

Velocity Limits (Redis + Lua atomic):
  • Transactions per minute limited
  • Rapid-fire detection window
  • Threshold triggers flagging for review

Detection Patterns:
  • Rapid-fire: Multiple transactions in short window
  • Bot farming: Repetitive low-value patterns
  • Distribution: Multi-account coordination (IP/device fingerprint)

[Specific thresholds redacted for security]
```

### 19.5 Platform Health Dashboard

```
ACTIVE USER METRICS (DAU/WAU/MAU)
═════════════════════════════════

  DAU = COUNT(DISTINCT user_id)
        WHERE activity_date >= NOW() - INTERVAL '24 hours'

  WAU = COUNT(DISTINCT user_id)
        WHERE activity_date >= NOW() - INTERVAL '7 days'

  MAU = COUNT(DISTINCT user_id)
        WHERE activity_date >= NOW() - INTERVAL '30 days'

Activity includes: chat, message, game, login, profile update, WebSocket connect

REAL-TIME ACTIVITY RATES
════════════════════════

Events per minute (rolling window):

  messages_per_minute:     Chat messages sent
  media_messages_per_minute: Gifts, images, stickers
  matches_per_minute:      New matches created
  calls_per_minute:        Voice/video calls initiated
  friendships_per_minute:  Friend requests/accepts
  games_per_minute:        In-chat games started

FEATURE ADOPTION
════════════════

Cosmetic Usage:
  • chats_with_message_frames: Users with message borders
  • chats_with_backgrounds: Users with chat backgrounds
  • users_with_pfp_frames: Users with profile frames
  • users_with_name_style: Users with custom names

Friendship Sources:
  • friendships_from_matches: % from match → chat → friend
  • friendships_from_other: % from direct add/invite

PLATFORM HEALTH INDICATORS
══════════════════════════

  ┌─────────────────────────────────────────────────────────────┐
  │  Category         │  Metrics                                │
  ├─────────────────────────────────────────────────────────────┤
  │  User Growth      │  New signups/day, conversion rate       │
  │  Engagement       │  Session duration, return rate          │
  │  Match Health     │  Success rate, friendship conversion    │
  │  Safety Health    │  Reports pending, blocked count         │
  │  Economy Health   │  Lynk velocity, spending trend          │
  └─────────────────────────────────────────────────────────────┘
```

### 19.6 Data Collection and Privacy

```
EVENT TRACKING
══════════════

Core Events Captured:

  Session Events:
    match:created, match:completed, match:voided
    Fields: match_id, duration_seconds, quality_score

  Message Events:
    message:sent, message:edited, message:deleted
    Fields: message_id, length, timestamp, media_type
    NOT captured: message content

  Interaction Events:
    friend:added, friend:removed
    Fields: from_user_id, to_user_id, source

  Economic Events:
    store:purchased, gift:sent
    Fields: user_id, item_id, amount_lynks

AGGREGATION ARCHITECTURE
════════════════════════

Real-Time (Redis):
  • Active user sets: metrics:daily:active:{date}
  • Counters: metrics:counter:{name}:{period}
  • Per-minute rates: computed from rolling window
  • TTL: Daily counters expire at midnight

Batch (PostgreSQL):
  • Hourly, daily, weekly, monthly snapshots
  • Cohort retention calculations
  • Window functions for trend analysis

Timeseries Retention: [configured retention period]

PRIVACY PRINCIPLES
══════════════════

  1. PII Exclusion
     • User IDs excluded from timeseries (aggregates only)
     • No message content stored
     • No conversation transcripts

  2. Aggregation Levels
     • Individual: Minimal PII (for admin investigation)
     • Cohort: No identifying information
     • Platform: Pure aggregates

  3. Data Minimization
     • Track: session count, message count, timing
     • Never track: message content, relationships, preferences
     • Retention: [configured period] raw, indefinite aggregates

  4. Compliance
     • COPPA: Minors excluded from cohort analysis
     • GDPR: User can request analytics deletion
     • Legal hold: Retention policies respect holds
```

---

## 20. Infrastructure and Distribution

The platform runs on AWS with Cloudflare providing edge security and routing.
The architecture uses serverless compute (Fargate) and serverless data (Aurora
Serverless, ElastiCache Serverless) for cost-efficient scaling from zero to
production load.

### 20.1 Network Architecture

```
TRAFFIC FLOW
════════════

┌─────────────────────────────────────────────────────────────────────────┐
│                              INTERNET                                   │
│                                  │                                      │
│          ┌───────────────────────┼───────────────────────┐              │
│          │                       │                       │              │
│          ▼                       ▼                       ▼              │
│   ┌─────────────┐         ┌─────────────┐         ┌─────────────┐       │
│   │   Edge CDN  │         │   Secure    │         │   Static    │       │
│   │   (Pages)   │         │   Tunnel    │         │   Assets    │       │
│   │             │         │             │         │    CDN      │       │
│   │  Frontend   │         │    API      │         │             │       │
│   │   Apps      │         │  Traffic    │         │             │       │
│   └─────────────┘         └──────┬──────┘         └──────┬──────┘       │
│                                  │                       │              │
│                                  ▼                       ▼              │
│                           ┌─────────────┐         ┌─────────────┐       │
│                           │   Private   │         │   Object    │       │
│                           │Load Balancer│         │   Storage   │       │
│                           └──────┬──────┘         └─────────────┘       │
│                                  │                                      │
│              ┌───────────────────┼───────────────┐                      │
│              ▼                   ▼               ▼                      │
│       ┌─────────────┐     ┌─────────────┐ ┌─────────────┐               │
│       │ User API    │     │ Admin API   │ │Game Runtime │               │
│       │ Backend     │     │ Backend     │ │             │               │
│       └──────┬──────┘     └──────┬──────┘ └──────┬──────┘               │
│              │                   │               │                      │
│              └───────────────────┼───────────────┘                      │
│                                  │                                      │
│              ┌───────────────────┼───────────────┐                      │
│              ▼                   ▼               ▼                      │
│       ┌─────────────┐     ┌─────────────┐ ┌─────────────┐               │
│       │  Database   │     │   Cache     │ │   Media     │               │
│       │ (PostgreSQL)│     │  (Redis)    │ │   Storage   │               │
│       │ Serverless  │     │   (TLS)     │ │             │               │
│       └─────────────┘     └─────────────┘ └─────────────┘               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

ROUTING PHILOSOPHY
══════════════════

  • Frontend apps served via edge CDN (static hosting)
  • API traffic routed through secure tunnel (no public IPs)
  • Static assets served via CDN with long cache TTLs
  • Media server accessible for WebRTC connections

Key insight: No public IP for backend services. All API traffic
enters through secure tunnel to private load balancer.

[Specific DNS records and internal routing redacted]
```

### 20.2 Edge Security Configuration

```
SECURE TUNNEL ARCHITECTURE
══════════════════════════

Why Tunnel over Public Load Balancer:
  • No public IP exposure (zero attack surface)
  • DDoS protection at edge
  • Automatic TLS termination
  • IP reputation filtering
  • Bot management

SECURITY SETTINGS
═════════════════

  • Strict SSL mode (origin cert required)
  • Modern TLS versions only
  • Browser integrity checking
  • WAF rules for common attacks

[Specific configuration details redacted]
```

### 20.3 Cloud Services

```
COMPUTE: SERVERLESS CONTAINERS
══════════════════════════════

Architecture: Fargate-style serverless containers

Services:
  • User-facing API backend (multi-instance for HA)
  • Admin API backend
  • Game runtime server
  • Safety scanning services
  • Edge tunnel connector

Benefits:
  • No server management
  • Per-second billing
  • Automatic scaling
  • VPC networking isolation

DATABASE: SERVERLESS POSTGRESQL
═══════════════════════════════

Mode: Serverless with automatic scaling

Features:
  • Multi-AZ (automatic failover)
  • Encryption at rest (AES-256)
  • Automated backups
  • Point-in-time recovery
  • Scales based on connection load

CACHE: SERVERLESS REDIS
═══════════════════════

Protocol: TLS-encrypted connections
Mode:     Serverless (no cluster management)

Features:
  • Auto-scaling compute and memory
  • Cross-AZ replication
  • Sub-millisecond latency
  • No per-node provisioning

STORAGE: OBJECT STORAGE
═══════════════════════

Bucket Categories:
  • Static assets (games, store items, shared libraries)
  • User media (profile pictures, chat attachments)
  • Evidence quarantine (law enforcement compliance)

CDN Configuration:
  • Immutable caching for versioned assets
  • Configurable TTL for dynamic content

[Specific resource names and configurations redacted]
```

### 20.4 VPC Architecture

```
NETWORK TOPOLOGY
════════════════

Architecture: Multi-AZ VPC with public and private subnets

  ┌─────────────────────────────────────────────────────────────┐
  │                           VPC                               │
  │                                                             │
  │  ┌──────────────────────┐  ┌──────────────────────┐         │
  │  │   PUBLIC SUBNET      │  │   PUBLIC SUBNET      │         │
  │  │   (Availability      │  │   (Availability      │         │
  │  │    Zone A)           │  │    Zone B)           │         │
  │  │                      │  │                      │         │
  │  │   NAT Gateway        │  │                      │         │
  │  └──────────────────────┘  └──────────────────────┘         │
  │                                                             │
  │  ┌──────────────────────┐  ┌──────────────────────┐         │
  │  │   PRIVATE SUBNET     │  │   PRIVATE SUBNET     │         │
  │  │   (Compute)          │  │   (Compute)          │         │
  │  │                      │  │                      │         │
  │  │   Container Tasks    │  │   Container Tasks    │         │
  │  │   Database Primary   │  │   Database Replica   │         │
  │  └──────────────────────┘  └──────────────────────┘         │
  │                                                             │
  │  ┌──────────────────────┐  ┌──────────────────────┐         │
  │  │   PRIVATE SUBNET     │  │   PRIVATE SUBNET     │         │
  │  │   (Data)             │  │   (Data)             │         │
  │  │                      │  │                      │         │
  │  │   Cache Nodes        │  │   Cache Nodes        │         │
  │  └──────────────────────┘  └──────────────────────┘         │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘

SECURITY GROUP STRATEGY
═══════════════════════

Principle: Least privilege access between components

  • Load balancer: Accepts only tunnel traffic
  • Backend services: Accept traffic only from load balancer
  • Database: Accepts connections only from backend services
  • Cache: Accepts connections only from backend services
  • Game runtime: Isolated with specific ingress rules
  • Safety services: Internal-only, no external access

Network isolation: Backend services cannot receive direct internet
traffic. All ingress through secure tunnel → load balancer → security group.

[Specific CIDR ranges and security group names redacted]
```

### 20.5 Multi-Worker Architecture

```
WHY MULTIPLE WORKERS?
═════════════════════

Single worker limitations:
  • Python GIL limits CPU parallelism
  • One worker crash = total outage
  • No horizontal scaling

Multi-worker solution:
  • Multiple container tasks for user-facing backend
  • Load balancer distributes requests across tasks
  • One crash doesn't affect others

Critical implication:

    ┌─────────────────────────────────────────────────────────────┐
    │                                                             │
    │  WORKERS DO NOT SHARE MEMORY                                │
    │                                                             │
    │  Task 1 memory ≠ Task 2 memory ≠ Task 3 memory              │
    │                                                             │
    │  User's request 1 → Task 1                                  │
    │  User's request 2 → Task 3 (different!)                     │
    │                                                             │
    │  Any "shared state" must live in Redis                      │
    │                                                             │
    └─────────────────────────────────────────────────────────────┘

What CANNOT be in-memory:
  • Rate limit counters (must be Redis + Lua atomic)
  • Token revocation caches (Redis as source of truth)
  • Session state (Redis)
  • Any shared counters

What CAN be in-memory:
  • Read-only configuration (loaded at startup)
  • Connection pools (per-task)
  • Immutable lookup tables
```

### 20.6 Cache Architecture

```
ELASTICACHE SERVERLESS (REDIS 7+)
═════════════════════════════════

Unlike traditional Redis Cluster with shards, ElastiCache Serverless
abstracts the cluster topology. Application connects to single endpoint.

Connection: TLS required (rediss://)
Authentication: IAM or username/password
ACL: Redis 7+ requires username (default: "default")

CACHE TTL PHILOSOPHY
════════════════════

TTLs are configured based on data sensitivity and update frequency:

  Cache Category             │ TTL Strategy              │ Rationale
  ───────────────────────────┼───────────────────────────┼────────────────
  Security-critical data     │ Short (minutes)           │ Ban/block status
  Presence/status data       │ Short (minutes)           │ Real-time needs
  Permission data            │ Short-medium (minutes)    │ Access control
  Historical/read-only       │ Medium (minutes-hours)    │ Low update rate
  Session data               │ Longer (hours)            │ Stable auth data

[Specific TTL values redacted for security]

CACHE PATTERNS
══════════════

Pattern 1: Cache-Aside (Read)

  1. Check Redis cache
  2. HIT: Return cached data
  3. MISS: Query PostgreSQL → Store in Redis → Return

Pattern 2: Invalidate-on-Write

  1. Write to PostgreSQL (source of truth)
  2. Delete from Redis cache
  3. Next read will repopulate

  CRITICAL: Always DB write first, then invalidate.
            Never reverse (stale data risk).

ATOMIC OPERATIONS (LUA SCRIPTS)
═══════════════════════════════

Lua scripts provide atomicity for distributed state:

  Use Cases:
    • Sliding window rate limiting
    • Fraud/velocity detection
    • Bounded counters
    • Strike system enforcement
    • Connection cleanup

Why Lua?
  • Atomic: No race conditions between check and increment
  • Fast: Executes on Redis server (no round trips)
  • Consistent: All workers see same state
```

### 20.7 CI/CD Pipeline

```
GITHUB ACTIONS DEPLOYMENT
═════════════════════════

Workflow: Deploy to AWS ECS
Trigger:  Push to main (backend/*, games/*)

Pipeline:

  ┌────────────────┐
  │ detect-changes │
  └───────┬────────┘
          │
    ┌─────┴─────┐
    │           │
    ▼           ▼
┌─────────┐ ┌─────────────────┐
│ build   │ │ sync-static-    │
│ images  │ │ assets (S3)     │
└────┬────┘ └────────┬────────┘
     │               │
     ▼               │
┌─────────┐          │
│ run     │          │
│ migrate │          │
└────┬────┘          │
     │               │
┌────┴────┬──────────┼─────────┐
│         │          │         │
▼         ▼          ▼         ▼
┌─────┐ ┌─────┐ ┌────────┐ ┌────────┐
│user │ │admin│ │game    │ │photodna│
│back │ │back │ │runtime │ │server  │
└──┬──┘ └──┬──┘ └────┬───┘ └────┬───┘
   │       │         │          │
   └───────┴─────────┴──────────┘
                     │
                     ▼
              ┌─────────────┐
              │ register-   │
              │ content     │
              └──────┬──────┘
                     │
                     ▼
              ┌─────────────┐
              │ purge       │
              │ cloudflare  │
              └─────────────┘

Key Steps:

  1. Build Docker images → Push to ECR
  2. Run Alembic migrations (one-off ECS task)
  3. Update ECS task definitions with new image
  4. Deploy services (rolling update)
  5. Sync store/game assets to S3
  6. Register content in database
  7. Purge Cloudflare cache

Authentication: GitHub OIDC → AWS IAM Role (no long-lived credentials)
```

---

## Appendix: Mathematical Foundations

### A.1 Vector Operations

```
Cosine Similarity (used in ALMA interest matching):

                 A · B           Σᵢ AᵢBᵢ
cos(θ) = ─────────────── = ─────────────────────
          ‖A‖ × ‖B‖       √(ΣᵢAᵢ²) × √(ΣᵢBᵢ²)

Range: [-1, 1]
  1  = identical direction
  0  = orthogonal (no similarity)
  -1 = opposite direction


Euclidean Distance:

d(A, B) = √(Σᵢ (Aᵢ - Bᵢ)²)

Used for: Position calculations, spatial queries
```

### A.2 Physics Equations

```
Kinematics (used in game physics):

Position:     p(t) = p₀ + v₀t + ½at²
Velocity:     v(t) = v₀ + at
Acceleration: a = F/m (Newton's Second Law)

Discrete integration (game loop):
  v(t+Δt) = v(t) + a × Δt
  p(t+Δt) = p(t) + v(t+Δt) × Δt


Elastic Collision (1D):

v₁' = ((m₁-m₂)v₁ + 2m₂v₂) / (m₁+m₂)
v₂' = ((m₂-m₁)v₂ + 2m₁v₁) / (m₁+m₂)

Conservation of momentum: m₁v₁ + m₂v₂ = m₁v₁' + m₂v₂'
Conservation of energy:   ½m₁v₁² + ½m₂v₂² = ½m₁v₁'² + ½m₂v₂'²
```

### A.3 Interpolation

```
Linear Interpolation (LERP):

lerp(a, b, t) = a + (b - a) × t = a(1-t) + bt

where t ∈ [0, 1]
  t = 0 → result = a
  t = 1 → result = b


Angle Interpolation (handles wraparound):

shortest_angle(from, to) = ((to - from + π) mod 2π) - π

lerp_angle(a, b, t) = a + shortest_angle(a, b) × t


Smoothstep (smooth start and end):

smoothstep(t) = 3t² - 2t³

Derivative is 0 at t=0 and t=1, creating smooth motion
```

### A.4 Probability and Statistics

```
Exponential Backoff (reconnection logic):

delay(n) = min(base × 2ⁿ + random_jitter, max_delay)

where:
  n = retry attempt number
  base = initial delay (e.g., 1 second)
  max_delay = cap (e.g., 30 seconds)
  random_jitter = small random value to prevent thundering herd


Sliding Window Rate Limit:

Let W = window size, L = limit, T = current time

Request allowed if:
  |{timestamp t : t ∈ (T-W, T]}| < L

Probability of limit hit (Poisson approximation):
  If average rate λ requests/window:
  P(hit limit) = 1 - Σₖ₌₀^(L-1) (e^(-λ) × λᵏ / k!)
```

---
