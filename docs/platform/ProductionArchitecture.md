# WeLynk Production Architecture

A social connection platform for adults and minors (16+) featuring chat, calls, games, and AI companion features.

---

## Overview

WeLynk's production architecture prioritizes:

1. **Security** - All traffic through Cloudflare, network isolation for untrusted code
2. **Compliance** - CSAM detection, NCMEC reporting, log retention
3. **Scalability** - Auto-scaling container services, managed databases
4. **Cost Efficiency** - Optimized for startup-friendly costs with linear scaling

### Technology Stack

| Component | Technology | Hosting |
|-----------|------------|---------|
| User Webapp | React/Vite | Cloudflare Pages |
| Admin Webapp | React/Vite | Cloudflare Pages |
| Game Studio | React/Vite | Cloudflare Pages |
| Mobile App | Flutter | App Store / Play Store |
| User Backend | Python/FastAPI | AWS ECS Fargate |
| Admin Backend | Python/FastAPI | AWS ECS Fargate |
| Game Runtime | Node.js/TypeScript | AWS ECS Fargate (isolated) |
| Content Moderation | Python/FastAPI | AWS ECS Fargate (isolated) |
| Database | PostgreSQL 15 | AWS RDS |
| Cache | Redis 7 | AWS ElastiCache |
| Video Calls | LiveKit | AWS EC2 |
| Static Assets CDN | S3 + CloudFront | AWS |

---

## Security Model

### Design Principle: Zero Direct Access

All public traffic flows through Cloudflare. Backend services have no public IP addresses.

Admin APIs are additionally protected by **Cloudflare Access** (Zero Trust), requiring authenticated service tokens for all requests.

```
                         INTERNET
                            │
                            ▼
                    ┌───────────────┐
                    │  CLOUDFLARE   │
                    │               │
                    │  • DDoS       │
                    │  • WAF        │
                    │  • Bot filter │
                    │  • Rate limit │
                    │  • SSL/TLS    │
                    └───────┬───────┘
                            │
                            ▼
                  ┌─────────────────────┐
                  │  Cloudflare Tunnel  │
                  │  (Outbound only)    │
                  └─────────┬───────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ AWS BACKENDS  │  ◄── No public IPs
                    └───────────────┘
```

### Three Layers of Protection

#### Layer 1: Cloudflare Tunnel

Cloudflare Tunnel (`cloudflared`) creates an **outbound-only** connection from AWS to Cloudflare. The backends have no public IP addresses - there's nothing to attack directly.

```
AWS Private Subnet                      Cloudflare Edge
┌─────────────────┐                    ┌─────────────────┐
│                 │   OUTBOUND ONLY    │                 │
│  cloudflared ───┼───────────────────►│ Tunnel Ingress  │
│  daemon         │   (no inbound)     │                 │
└─────────────────┘                    └─────────────────┘

Result: No public ports exposed. Cannot be port scanned or DDoSed directly.
```

#### Layer 2: Mutual TLS (mTLS)

Even if the tunnel were somehow bypassed, the load balancer validates Cloudflare's client certificate. Requests without valid Cloudflare-signed certificates are rejected at the network level.

#### Layer 3: IP Allowlisting

Security groups only allow traffic from Cloudflare's published IP ranges as an additional defense-in-depth measure.

### Why This Cannot Be Spoofed

| Attack Vector | Why It Fails |
|---------------|--------------|
| Direct IP access | No public IPs exist (Tunnel) |
| Spoofed headers | mTLS certificate validation fails |
| Spoofed source IP | Security group blocks non-CF IPs |
| Man-in-the-middle | mTLS requires Cloudflare's private key |
| DNS hijacking | Tunnel doesn't use DNS for origin connection |

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              CLIENTS                                     │
│                                                                          │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │
│   │ Flutter App  │  │ User Webapp  │  │ Admin Webapp │  │ Game Studio │  │
│   │ (iOS/Android)│  │   (React)    │  │   (React)    │  │   (React)   │  │
│   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬──────┘  │
└──────────┼─────────────────┼─────────────────┼─────────────────┼─────────┘
           │                 │                 │                 │
           └────────────────┬┴─────────────────┴┬────────────────┘
                            │                   │
                            ▼                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            CLOUDFLARE                                    │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  DDoS Protection  │  WAF  │  Bot Management  │  Rate Limiting   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌────────────────────────┐         ┌─────────────────────────────┐    │
│   │   STATIC HOSTING       │         │      API PROXY              │    │
│   │   (Cloudflare Pages)   │         │      (Cloudflare Tunnel)    │    │
│   │                        │         │                             │    │
│   │   • Main webapp        │         │   • User API                │    │
│   │   • Admin webapp       │         │   • Admin API               │    │
│   │   • Game Studio        │         │   • Media signaling         │    │
│   └────────────────────────┘         └──────────────┬──────────────┘    │
│                                                     │                    │
└─────────────────────────────────────────────────────┼────────────────────┘
                                                      │
                                          Cloudflare Tunnel (mTLS)
                                                      │
                                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              AWS CLOUD                                   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                         VPC (Private)                            │   │
│   │                                                                  │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │                  TUNNEL SUBNET                           │   │   │
│   │   │                                                          │   │   │
│   │   │   ┌────────────────────────────────────────────────┐     │   │   │
│   │   │   │  CLOUDFLARED DAEMON (ECS)                      │     │   │   │
│   │   │   │  • Outbound tunnel to Cloudflare               │     │   │   │
│   │   │   │  • Routes to internal load balancer            │     │   │   │
│   │   │   └────────────────────────┬───────────────────────┘     │   │   │
│   │   │                            │                             │   │   │
│   │   └────────────────────────────┼─────────────────────────────┘   │   │
│   │                                │                                 │   │
│   │   ┌────────────────────────────┼─────────────────────────────┐   │   │
│   │   │              LOAD BALANCER SUBNET                        │   │   │
│   │   │                            │                             │   │   │
│   │   │   ┌────────────────────────┴───────────────────────┐     │   │   │
│   │   │   │         INTERNAL APPLICATION LOAD BALANCER     │     │   │   │
│   │   │   │                                                │     │   │   │
│   │   │   │   Routes requests to appropriate backend       │     │   │   │
│   │   │   │   service based on path                        │     │   │   │
│   │   │   └───────────────────┬────────────────────────────┘     │   │   │
│   │   │                       │                                  │   │   │
│   │   └───────────────────────┼──────────────────────────────────┘   │   │
│   │                           │                                      │   │
│   │   ┌───────────────────────┼──────────────────────────────────┐   │   │
│   │   │           BACKEND SUBNET                                 │   │   │
│   │   │                       │                                  │   │   │
│   │   │   ┌───────────────────┴───────────────────┐              │   │   │
│   │   │   │                                       │              │   │   │
│   │   │   ▼                                       ▼              │   │   │
│   │   │   ┌─────────────────┐   ┌─────────────────┐              │   │   │
│   │   │   │  USER-BACKEND   │   │  ADMIN-BACKEND  │              │   │   │
│   │   │   │  (ECS Fargate)  │   │  (ECS Fargate)  │              │   │   │
│   │   │   │                 │   │                 │              │   │   │
│   │   │   │  • REST API     │   │  • Dashboard    │              │   │   │
│   │   │   │  • WebSocket    │   │  • Moderation   │              │   │   │
│   │   │   │  • Auth, Chat   │   │  • Analytics    │              │   │   │
│   │   │   │  • Matching     │   │                 │              │   │   │
│   │   │   └────────┬────────┘   └────────┬────────┘              │   │   │
│   │   │            │                     │                       │   │   │
│   │   └────────────┼─────────────────────┼───────────────────────┘   │   │
│   │                │                     │                           │   │
│   │       ┌────────┴─────────────────────┴────────┐                  │   │
│   │       │                                       │                  │   │
│   │       ▼                                       ▼                  │   │
│   │   ┌──────────────────┐              ┌──────────────────┐         │   │
│   │   │   DATA SUBNET    │              │  ISOLATED SUBNETS│         │   │
│   │   │                  │              │                  │         │   │
│   │   │  ┌────────────┐  │              │  ┌────────────┐  │         │   │
│   │   │  │ PostgreSQL │  │              │  │   GAME     │  │         │   │
│   │   │  │   (RDS)    │  │              │  │  RUNTIME   │  │         │   │
│   │   │  └────────────┘  │              │  │ (isolated) │  │         │   │
│   │   │                  │              │  └────────────┘  │         │   │
│   │   │  ┌────────────┐  │              │                  │         │   │
│   │   │  │   Redis    │  │              │  ┌────────────┐  │         │   │
│   │   │  │ ElastiCache│  │              │  │  CONTENT   │  │         │   │
│   │   │  └────────────┘  │              │  │ MODERATION │  │         │   │
│   │   │                  │              │  │ (isolated) │  │         │   │
│   │   └──────────────────┘              │  └────────────┘  │         │   │
│   │                                     │                  │         │   │
│   │                                     └──────────────────┘         │   │
│   │                                                                  │   │
│   └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Network Isolation

### The Problem: Untrusted Code Execution

WeLynk allows users to create and submit games. These games run on our infrastructure. Even with VM sandboxing (V8 isolates), defense-in-depth requires **complete network isolation**.

### Isolation Strategy

The game runtime is placed in a dedicated subnet with strict network controls:

| What Game Runtime CAN Do | What Game Runtime CANNOT Do |
|--------------------------|----------------------------|
| Respond to user-backend requests | Access database directly |
| Make authenticated callbacks to user-backend | Access Redis cache |
| Nothing else | Reach admin backend |
| | Access content moderation service |
| | Make any internet requests |
| | Pivot to other AWS services |

```
Network isolation ensures that even if the VM sandbox is compromised,
the blast radius is strictly limited. No direct access to databases,
caches, admin services, or external networks is possible.
```

### Content Moderation Isolation

The content moderation service handles sensitive image data. It's similarly isolated:

- No database access (can't store images)
- No cache access (can't cache images)
- Limited outbound (only to moderation APIs)
- Images deleted from memory immediately after processing

---

## Game Runtime Scaling

### Architecture

The game runtime supports horizontal scaling for thousands of concurrent games.

```
┌─────────────────────────────────────────────────────────────────────┐
│                          REDIS                                       │
│                                                                      │
│  Stores runtime health status and session routing information        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                    ▲
                    │ manages registry
                    │
┌───────────────────┴─────────────────────────────────────────────────┐
│                       user-backend                                   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Runtime Discovery                                           │    │
│  │  • Discovers runtimes via ECS Service Discovery              │    │
│  │  • Polls health endpoints                                    │    │
│  │  • Tracks health, session counts, memory usage               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Request Router                                              │    │
│  │  • Routes requests to correct runtime                        │    │
│  │  • Load balances new sessions                                │    │
│  │  • Handles runtime failures                                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────┬────────────────────────────┬──────────────────────┘
                  │                            │
                  ▼                            ▼
        ┌─────────────────┐          ┌─────────────────┐
        │  game-runtime   │          │  game-runtime   │
        │    Task 1       │          │    Task 2       │
        │                 │          │                 │
        │  V8 Isolates    │          │  V8 Isolates    │
        │                 │          │                 │
        └─────────────────┘          └─────────────────┘
```

### Key Design Decisions

1. **Game runtime has NO infrastructure access** - It cannot reach Redis, the database, or any other service. User-backend manages all discovery and routing.

2. **Sticky sessions** - Once a game session starts on a runtime, all requests for that session route to the same runtime.

3. **Graceful degradation** - If a runtime dies, those game sessions end. Clients see "game ended" and can start a new game.

4. **Load-based routing** - New sessions go to the runtime with lowest load (weighted by session count and memory usage).

### Scaling

The game runtime scales horizontally based on demand. ECS auto-scaling policies monitor session counts and memory usage to add or remove runtime tasks as needed. This ensures consistent game performance regardless of concurrent user load.

---

## Data Flow Examples

### User Authentication

```
┌─────────┐     ┌────────────┐     ┌─────────┐     ┌──────────────┐     ┌─────┐
│ Client  │────▶│ Cloudflare │────▶│ Tunnel  │────▶│ user-backend │────▶│ RDS │
└─────────┘     └────────────┘     └─────────┘     └──────────────┘     └─────┘
                     │                                    │
                     │  • DDoS filtering                  │
                     │  • WAF rules                       ▼
                     │  • Rate limiting            ┌───────────┐
                     │  • Bot detection            │   Redis   │
                     │                             │ (session) │
                     ▼                             └───────────┘
              Block if malicious
```

### Game Session

```
┌─────────┐     ┌────────────┐     ┌──────────────┐     ┌──────────────┐
│ Client  │────▶│ Cloudflare │────▶│ user-backend │────▶│ game-runtime │
└─────────┘     └────────────┘     └──────────────┘     └──────────────┘
     │                                    │                     │
     │  WebSocket                         │  Route to           │  Execute in
     │  connection                        │  correct runtime    │  V8 isolate
     │                                    │                     │
     │                                    ▼                     │
     │                              ┌───────────┐               │
     │                              │   Redis   │               │
     │                              │ (routing) │◀──────────────┘
     │                              └───────────┘   Return game state
     │                                    │
     │◀───────────────────────────────────┘
          Game state update via WebSocket
```

### Image Upload (with Content Moderation)

```
┌─────────┐     ┌────────────┐     ┌──────────────┐     ┌────────────┐
│ Client  │────▶│ Cloudflare │────▶│ user-backend │────▶│ Moderation │
└─────────┘     └────────────┘     └──────────────┘     └────────────┘
     │                                    │                    │
     │  Upload                            │                    │  Scan image
     │  image                             │                    │  Check against
     │                                    │                    │  known violations
     │                                    │                    │
     │                                    │              ┌─────┴─────┐
     │                                    │              │           │
     │                                    │              ▼           ▼
     │                                    │         Clean?      Violation?
     │                                    │              │           │
     │                                    │◀─────────────┘           │
     │                                    │                    Report to
     │                              ┌─────┴─────┐              authorities
     │                              │           │
     │                              ▼           ▼
     │                        Save to S3   Block user
     │◀─────────────────────────────┘
          Response
```

---

## Cost Estimates

### Cost Optimization

The architecture is designed to be cost-efficient through:

- **Auto-scaling** - Services scale based on demand, paying only for what's used
- **Serverless containers** - ECS Fargate eliminates idle EC2 capacity
- **Managed services** - RDS and ElastiCache reduce operational overhead
- **CDN caching** - CloudFront reduces origin traffic and latency
- **Reserved capacity** - Critical services use reserved instances for cost savings

Cost scales approximately linearly with user growth, with economies of scale at higher tiers.

---

## Static Assets CDN

Games, store items, and shared libraries are served via CloudFront CDN for low-latency global delivery.

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│     Client      │────▶│   CloudFront    │────▶│       S3        │
│                 │     │   (CDN Edge)    │     │  (Origin)       │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                              │
                              │  Cache Strategy:
                              │  • HTML: no-cache (always fresh)
                              │  • JS/Assets: immutable (1 year)
                              │  • Config JSON: no-cache
                              │
```

### Asset Organization

```
s3://<bucket-name>/
├── games/           # Built game bundles (HTML, JS, assets)
│   └── <publisher>/  # Publisher namespace
│       └── <game>/   # Individual game assets
├── store/           # Store item assets (images, configs)
│   ├── backgrounds/
│   ├── pfp-frames/
│   └── message-borders/
└── libs/            # Shared game libraries
    └── v1/          # Versioned for cache busting
        └── *.min.js # Bundled libraries
```

### Cache Invalidation

On deploy, CloudFront cache is invalidated for updated paths. HTML files use `no-cache` headers so browsers always check for updates, while hashed JS/CSS assets are cached indefinitely.

---

## Deployment Pipeline

### CI/CD Flow

The deployment pipeline runs in parallel where possible for speed:

```
┌─────────────────────────────────────────────────────────────────────┐
│                           GITHUB ACTIONS                             │
│                                                                      │
│   on push to main                                                    │
│        │                                                             │
│        ├────────────────────┬────────────────────┐                   │
│        │                    │                    │                   │
│        ▼                    ▼                    ▼                   │
│   ┌──────────┐       ┌────────────┐      ┌─────────────┐            │
│   │  STATIC  │       │   BUILD    │      │   BUILD     │            │
│   │  ASSETS  │       │  BACKEND   │      │   GAMES     │            │
│   │          │       │  IMAGES    │      │  & LIBS     │            │
│   │ • Sync   │       │            │      │             │            │
│   │   to S3  │       │ • Docker   │      │ • npm build │            │
│   │ • CDN    │       │   build    │      │ • Bundle    │            │
│   │   purge  │       │ • Push ECR │      │             │            │
│   └────┬─────┘       └─────┬──────┘      └──────┬──────┘            │
│        │                   │                    │                    │
│        │                   ▼                    │                    │
│        │            ┌────────────┐              │                    │
│        │            │ MIGRATIONS │              │                    │
│        │            │            │              │                    │
│        │            │ • Alembic  │              │                    │
│        │            │   upgrade  │              │                    │
│        │            └─────┬──────┘              │                    │
│        │                  │                     │                    │
│        │                  ▼                     │                    │
│        │    ┌─────────────┴─────────────┐       │                    │
│        │    │                           │       │                    │
│        │    ▼                           ▼       │                    │
│        │ ┌──────────┐           ┌──────────┐    │                    │
│        │ │  DEPLOY  │           │  DEPLOY  │    │                    │
│        │ │  USER    │           │  ADMIN   │    │                    │
│        │ │ BACKEND  │           │ BACKEND  │    │                    │
│        │ └──────────┘           └──────────┘    │                    │
│        │                                        │                    │
│        │    ┌───────────────────────────────────┘                    │
│        │    │                                                        │
│        │    ▼                                                        │
│        │ ┌──────────┐    ┌──────────┐                                │
│        │ │  DEPLOY  │    │  DEPLOY  │                                │
│        │ │   GAME   │    │ CONTENT  │                                │
│        │ │ RUNTIME  │    │MODERATION│                                │
│        │ └──────────┘    └──────────┘                                │
│        │                                                             │
│        └───────────────────┐                                         │
│                            ▼                                         │
│                    ┌─────────────┐                                   │
│                    │  REGISTER   │                                   │
│                    │  CONTENT    │                                   │
│                    │             │                                   │
│                    │ • Sync game │                                   │
│                    │   configs   │                                   │
│                    │ • Sync store│                                   │
│                    │   items     │                                   │
│                    └─────────────┘                                   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Deployment Strategy

- **Zero-downtime rolling deployments**
- New tasks launch alongside old tasks
- Traffic shifts after health checks pass
- Automatic rollback on failure (circuit breaker)

### Frontend Deployment

Frontend apps deploy directly to Cloudflare Pages on push to main, separate from backend pipeline.

---

## Key Design Principles

### 1. Defense in Depth

Multiple security layers so no single point of failure:
- Cloudflare DDoS/WAF (Layer 1)
- Cloudflare Tunnel - no public IPs (Layer 2)
- mTLS authentication (Layer 3)
- IP allowlisting (Layer 4)
- Network isolation via security groups (Layer 5)

### 2. Principle of Least Privilege

Each service can only access what it needs:
- Game runtime: Only user-backend API
- Content moderation: Only external moderation APIs
- Backends: Only their required databases

### 3. Fail Secure

When components fail, they fail safely:
- Runtime dies → Game sessions end cleanly
- Moderation unavailable → Uploads blocked (not allowed)
- Database unavailable → Service returns errors (not stale data)

### 4. Compliance by Design

Built-in compliance rather than bolted on:
- Log retention configured at infrastructure level
- Content moderation in the upload path (not async)
- Encryption at rest and in transit by default

---

## Technologies Used

| Category | Technology |
|----------|------------|
| **Frontend** | React, Vite, TypeScript |
| **Mobile** | Flutter, Dart |
| **Backend** | Python, FastAPI, asyncio |
| **Game Runtime** | Node.js, TypeScript, V8 Isolates |
| **Database** | PostgreSQL 15 |
| **Cache** | Redis 7 |
| **Video/Audio** | LiveKit (WebRTC SFU) |
| **Container Orchestration** | AWS ECS Fargate |
| **CDN/Security** | Cloudflare (Pages, Tunnel, WAF), CloudFront (static assets) |
| **CI/CD** | GitHub Actions |
| **Infrastructure** | AWS (VPC, RDS, ElastiCache, S3, ECR, CloudFront) |

---

*This document describes the production architecture for WeLynk. For security reasons, specific configuration values, IP addresses, and operational procedures are not included.*
