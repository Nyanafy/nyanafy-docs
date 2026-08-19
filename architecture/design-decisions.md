# Nyanafy — Design Decisions

This document captures the significant architectural and product design decisions made during the development of Nyanafy — the reasoning behind each choice, the alternatives considered, and the trade-offs accepted.

These decisions are documented not just for transparency, but as a living reference for future contributors and as a record of intentional thinking over accidental outcomes.

---

## 1. Content-Driven Architecture Over Hardcoded UI

### Decision
All learning content — worlds, paths, levels, activities, and assets — is stored in MongoDB and fetched dynamically. The game engine renders whatever data it receives. No content is hardcoded in the frontend.

### Why
A Kannada learning platform needs to grow continuously — new letters, new words, new levels, new worlds. If content were hardcoded in React components, every new lesson would require a code change, a build, and a redeployment. At scale, this would make content management impossible for a solo team.

### Result
Adding a new level requires only inserting a MongoDB document. Zero code changes. Zero redeployment. The same game engine that renders Level 1 renders Level 50.

### Trade-off
More complex initial architecture. A simpler hardcoded approach would have been faster to build initially but would have created an unmaintainable wall of content in the codebase within weeks.

---

## 2. Separating Identity (Supabase) from Application Data (MongoDB)

### Decision
Supabase Auth owns identity — email, password, JWT issuance. MongoDB owns everything else — profiles, progress, content. The link between them is a single `supabaseUserId` field stored in MongoDB.

### Why
Mixing authentication state with application data creates tight coupling that is difficult to change. If we ever need to switch auth providers, change JWT algorithms, or add OAuth providers, the application data layer is completely unaffected.

### Result
Spring Boot never sees or stores passwords. It only validates JWTs. MongoDB documents reference users by `supabaseUserId` — a UUID that never changes regardless of what happens on the auth side.

### Trade-off
Two services to manage instead of one. Accepted because the separation of concerns is worth the operational simplicity of using managed services for both.

---

## 3. Aggregation APIs — One Response Per Page

### Decision
Spring Boot aggregates data from multiple MongoDB collections server-side and returns a single complete response per page. React never makes multiple sequential API calls to render one screen.

### Why
Kids navigate back and forth between screens constantly — world map, path page, level page, back to path, back to world map. Multiple sequential API calls per screen would cause visible loading delays and unnecessary backend load. On slow mobile connections, the user experience would feel broken.

### Result
Every page in Nyanafy loads with a single API call. The backend does the joining, filtering, and shaping — the frontend just renders.

### Trade-off
More complex Spring Boot service layer. Aggregation queries are harder to write and test than simple CRUD. Accepted because the frontend performance benefit is non-negotiable for a kids app.

---

## 4. Localized Content Map for Multi-Language from Day One

### Decision
Every learning asset stores language-specific content in a map keyed by language code (`"kn"` for Kannada, `"hi"` for Hindi, etc.) from the very first document inserted.

### Why
Adding multi-language support after the fact almost always requires breaking schema changes — renaming fields, migrating documents, updating every API that touches content. Doing it from day one costs almost nothing extra but completely eliminates future migration risk.

### Result
Kannada content works exactly as before. Adding Hindi support in the future means inserting new entries in the same map — zero schema changes, zero API changes, zero disruption to existing Kannada content.

### Trade-off
Slightly more verbose document structure from day one. Accepted immediately — the future cost of NOT doing this would have been far higher.

---

## 5. Separate Progress Collections for Independent Path Tracking

### Decision
Learner progress is tracked across three independent collections:
- `learner_progress` — overall XP and summary stats
- `learner_level_progress` — per-level completion status
- `learner_path_progress` — per-path completion status

### Why
The initial design used a single `learnerProgress` document with embedded arrays for completed levels and paths. This worked for one path but became immediately messy when a child could progress simultaneously across multiple paths and worlds — arrays from different paths mixed together, queries became complex, and the UI showed incorrect data.

### Result
Each collection has a single, clear responsibility. A child can be on Path 1 Level 3 and Path 2 Level 1 simultaneously with no data conflicts. Path pages query `learner_path_progress` independently. The dashboard queries only `learner_progress` for lightweight summary stats.

### Trade-off
Three collections instead of one means three updates on level completion instead of one. This is handled in a single API call (`POST /api/progress/complete-level`) that updates all three atomically — the frontend never knows or cares.

---

## 6. XP Tracked Locally During Activity, Saved Only on Level Completion

### Decision
XP earned during activities is accumulated in React state. A single API call saves everything to MongoDB when the child taps "Finish Level." There is no API call after each individual activity completion.

### Why
A level contains 15 activities. Making an API call after each one would mean 15 database writes per level — unnecessary load, and potential for race conditions or partial saves if the network drops mid-level.

### Result
The backend receives one clean, complete update per level: total XP earned, all activities completed, timestamp. Simple, reliable, minimal network usage.

### Trade-off
If a child closes the app mid-level, progress within that level is lost. This is an acceptable trade-off for MVP — the level simply starts again. Checkpoint saving within a level is a Phase 2 consideration, prioritized only after real user feedback confirms it's a genuine pain point.

---

## 7. Multi-Step Activity Sequences — Checkpoint Saving

### Decision
For activity sequences with multiple indexed steps (e.g. a multi-part learning flow), progress is saved after each step completes — not just at the end of the entire sequence.

### Why
A child completing step 3 of 5 in a sequence and accidentally closing the tab should not lose steps 1 and 2. The emotional cost of repeating already-completed steps is high for young children and damages trust in the app.

### Result
`currentStepIndex` is persisted after each step. On return, the sequence resumes exactly where the child left off.

### Trade-off
More API calls than pure end-of-sequence saving. Accepted because the user experience cost of losing multi-step progress outweighs the backend load of a few extra writes.

---

## 8. Parent Account with Child Profiles, Not Separate Child Accounts

### Decision
Children do not have their own login credentials. A parent account manages up to 3 child profiles. Children select their profile from a profile selector screen after the parent logs in.

### Why
Children aged 4-10 cannot be expected to manage their own email and password. Giving children independent accounts creates safety, COPPA compliance, and support complexity far beyond MVP scope. The Netflix-style profile selector within a parent account is a familiar, intuitive pattern.

### Result
One login per family. Clean parental oversight. Simple auth flow. Child data is always linked to a responsible adult account.

### Trade-off
A teenager wanting to use the app independently cannot do so without a parent creating an account. This is addressed by the "Self Learner" account type for users 12+ who register independently.

---

## 9. Storage Keys in MongoDB, URLs Resolved in API

### Decision
MongoDB stores only the storage key (file path) for images and audio. The Spring Boot API resolves these to full Supabase CDN URLs before returning responses to React.

### Why
Storing full URLs in MongoDB couples the database to the storage provider. If Supabase Storage URLs change (region migration, provider switch, CDN change), every document in MongoDB would need updating — thousands of records.

### Result
Changing the storage provider requires updating one environment variable (`SUPABASE_STORAGE_BASE_URL`). All documents remain unchanged. React always receives ready-to-use URLs without knowing anything about how or where files are stored.

### Trade-off
An extra string concatenation on every API response. Negligible performance cost for a significant architectural flexibility gain.

---

## 10. Admin Role via Supabase Custom Claims

### Decision
Content management endpoints (POST/PUT/DELETE on worlds, levels, assets) require an `admin` role. This role is stored in `app_metadata` in Supabase and included in the JWT. Spring Boot reads it from the JWT claim — no separate admin database table required.

### Why
A separate admin user table would require additional queries on every protected request. JWT claims are already validated on every request — adding a role check is zero additional overhead.

### Result
The platform admin (currently the solo developer) sets `role: admin` in their Supabase account once. Every subsequent API call carries this role in the JWT automatically. Parent accounts have no admin claim — they cannot modify content regardless of how they call the API.

### Trade-off
Admin role management requires direct Supabase dashboard access or SQL. An admin management UI is a Phase 2 consideration when the team grows beyond one person.

---

## 11. Game Engine Designed as a Reusable Content Renderer

### Decision
The React game engine is built as a generic content renderer, not as a collection of hardcoded activity screens. Activity type, content, sequence, and configuration all come from the API. The engine routes to the correct activity component based on `activityType` in the data.

### Why
With 15 activity types and potentially hundreds of levels across multiple paths and worlds, hardcoding each screen would create an unmaintainable mess. A data-driven engine means the same codebase renders every level in the app.

### Result
Adding a new level requires zero frontend changes. Adding a new activity type requires only one new React component and one new `activityType` value in MongoDB. All routing, sequencing, XP tracking, and progress saving are handled by the engine automatically.

### Trade-off
More complex initial engine design. Worth it from day one — the alternative would have become unmanageable within the first few levels of content.

---

*For technology choices see [Tech Stack](tech-stack.md)*
*For system structure see [High Level Architecture](high-level-architecture.md)*
