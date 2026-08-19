# Nyanafy — Technology Stack

## Overview

Nyanafy is built on a modern, cloud-native stack chosen specifically for a solo-built, content-driven kids learning platform — prioritising developer productivity, scalability without upfront cost, and clean separation of concerns.

Every technology choice was made with three constraints in mind:
- **Solo developer** — minimal operational overhead, managed services where possible
- **Startup budget** — free tiers that scale as the product grows
- **Kids app requirements** — fast content delivery, reliable auth, safe data handling

---

## Frontend

### React.js
**Why:** React's component model maps naturally to a game engine UI — reusable activity components, consistent state patterns, and a large ecosystem of animation libraries. Critically, React skills transfer directly to React Native, enabling a future mobile app without rebuilding the frontend from scratch.

**Alternatives considered:** Vue.js, plain HTML/JS
**Why React won:** Ecosystem maturity, future React Native path, and stronger community support for the animation and interaction patterns a kids app requires.

---

### TanStack Query (React Query)
**Why:** A kids app involves a lot of repeated screen navigation — children go back and forth between the world map, paths, and levels constantly. TanStack Query's intelligent caching means these screens load instantly on revisit without redundant API calls. Cache invalidation is triggered only on meaningful events (activity completed, level finished), not on every navigation.

**Problem it solved:** Without it, every screen change triggered a fresh API call — causing unnecessary load on the backend and visible loading delays for children.

---

### Framer Motion + Lottie
**Why:** Children need visual feedback — celebrations, transitions, and mascot animations need to feel alive. Framer Motion handles page and component transitions cleanly within React. Lottie renders lightweight JSON-based animations for reward moments without heavy video files.

---

## Backend

### Spring Boot 4 (Java 21)
**Why:** The platform was built by a Java developer with production experience in payment systems and risk management. Using Spring Boot meant applying deep existing knowledge rather than learning a new backend framework — reducing risk on a solo project. Spring Boot 4 with Java 21 brings virtual threads, modern records, and production-grade security out of the box.

**Java 21 specifically:** Virtual threads (enabled via `spring.threads.virtual.enabled=true`) handle concurrent learner sessions efficiently without the overhead of traditional thread-per-request models — important as the user base grows.

**Alternatives considered:** Node.js/Express, FastAPI
**Why Spring Boot won:** Existing deep expertise, production-grade Spring Security for JWT validation, robust MongoDB integration via Spring Data.

---

### Spring Data MongoDB
**Why:** Provides clean repository abstractions over MongoDB without sacrificing the flexibility of the document model. Custom aggregation pipelines are used for page-level data fetching, keeping complex joins server-side rather than in React.

---

### Spring Security + OAuth2 Resource Server
**Why:** Rather than building JWT validation from scratch, Spring Security's OAuth2 Resource Server validates Supabase-issued JWTs against Supabase's public JWKS endpoint automatically. Role-based access control (admin vs learner) is handled via custom JWT claim mapping.

---

### SpringDoc OpenAPI (Swagger UI)
**Why:** API documentation generated automatically from controller annotations. During development, Swagger UI enabled rapid endpoint testing with Bearer token authorization. In production, Swagger is disabled via environment variable.

---

## Database

### MongoDB Atlas (Free Tier M0)
**Why:** The core challenge of a content-driven learning platform is that different activity types have fundamentally different data shapes — a matching game has different fields than a listening activity or a number tracing exercise. MongoDB's flexible document schema accommodates this without the complexity of a polymorphic relational schema.

**Key benefit:** New activity types, levels, and worlds can be added by inserting new documents — no schema migrations, no code changes, no redeployment required.

**Alternatives considered:** PostgreSQL (Supabase), Firebase Firestore
**Why MongoDB won:** Document flexibility for varied activity configs, familiarity with Spring Data MongoDB, and the ability to embed related data (activities within levels) that is always fetched together.

**Free tier capacity:** 512MB storage — sufficient for all Kannada content at MVP and well beyond, since images and audio live in Supabase Storage rather than MongoDB.

---

## Authentication & Storage

### Supabase Auth
**Why:** Building authentication from scratch — password hashing, JWT issuance, email verification, password reset flows — is complex, security-sensitive, and time-consuming. Supabase Auth handles all of this, issuing ES256-signed JWTs that Spring Boot validates independently. The separation means auth can be swapped or upgraded without touching application data.

**Specific features used:**
- Email/password authentication
- Password reset via email
- JWT (ES256) with custom app_metadata claims (admin role)
- Email confirmation (disabled for MVP, enabled pre-production)

**Alternatives considered:** Firebase Auth, Auth0, building custom JWT
**Why Supabase won:** Free tier generosity, ES256 JWT compatibility with Spring Security, and co-location with Supabase Storage under one project.

---

### Supabase Storage
**Why:** Learning content for a kids app is image and audio heavy — SVG illustrations for every vocabulary word, audio pronunciations for every letter and word. Storing binary files in MongoDB would hit document size limits quickly and slow down every query. Supabase Storage provides CDN-backed delivery with public bucket support, keeping assets fast and globally distributed.

**Design decision:** MongoDB stores only the storage key (file path). The Spring Boot API resolves keys to full CDN URLs before returning responses — the frontend never knows or cares where files are stored.

**Bucket structure:**
```
learning_assets/   → SVG illustrations (public)
audio/             → Kannada pronunciation files (public)
avatars/           → Child profile avatars (public)
mascots/           → Mascot character assets (public)
```

**Alternatives considered:** AWS S3, Cloudflare R2
**Why Supabase Storage won:** Already using Supabase Auth — one less service to manage, one dashboard, one billing account.

---

## Infrastructure & Hosting

### Vercel (React Frontend)
**Why:** Zero-configuration deployment for React applications with automatic preview deployments on every push, global CDN, and a generous free tier. Environment variables managed per deployment environment (development, production).

---

### Railway (Spring Boot Backend)
**Why:** Railway handles Java/Spring Boot deployments better than most free-tier alternatives (Render has slow cold starts for JVM apps). Automatic deployment from GitHub, environment variable management, and sufficient free tier for MVP traffic.

---

## Development Tools

### Gradle (Build Tool)
**Why:** Modern, flexible build system for the Spring Boot project. Gradle's incremental builds are faster than Maven for iterative development.

### Lombok
**Why:** Eliminates boilerplate Java code (getters, setters, builders, constructors) from model and DTO classes — critical for maintaining velocity as a solo developer adding new collections and response types frequently.

### ngrok (Local Testing)
**Why:** Enables testing the full stack on a real mobile device without deployment — exposes the local React and Spring Boot servers via public URLs for phone-based testing during development.

---

## Cost at MVP Launch

| Service | Plan | Cost |
|---|---|---|
| MongoDB Atlas | M0 Free | $0 |
| Supabase | Free tier | $0 |
| Vercel | Hobby (free) | $0 |
| Railway | Free tier | $0 |
| Domain | Custom domain | ~$10-15/year |
| **Total** | | **~$10-15/year** |

Built and launched for the cost of a domain name. Infrastructure scales with the product — upgrade only when traffic demands it.

---

*For system design and how these technologies connect see [High Level Architecture](high-level-architecture.md)*
