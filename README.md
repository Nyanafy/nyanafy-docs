# nyanafy-docs
Public architecture and documentation for Nyanafy —  a gamified Kannada learning platform for kids.


🎮 Nyanafy — ಕಲಿಸೋಣ (Let's Teach Kannada)

A gamified Kannada language learning platform for kids, built to make regional language education engaging, accessible, and joyful. Launching with Kannada — designed from the ground up to support multiple Indian regional languages in the future.


🌟 The Problem

Kannada is one of India's oldest classical languages, spoken by over 44 million people. Yet for children growing up outside Karnataka, or in urban households where English dominates daily life, learning Kannada is either boring, inaccessible, or both.

Existing tools are either:

Too academic — textbook-style, no engagement
Too generic — not culturally grounded in Kannada
Non-existent — very few apps target regional Indian languages for young children

Nyanafy exists to change that.


💡 The Solution

Nyanafy turns Kannada learning into an adventure. Kids explore worlds, complete activities, earn XP, and unlock new levels — all while genuinely learning Kannada through their immediate, familiar world.

Every interaction is designed for a child aged 4-10, with:

A friendly mascot that guides and encourages with audio
Visual and audio-rich content grounded in everyday Kannada life
Short, focused activity sessions with 15 different activity types
Immediate positive reinforcement through XP and celebrations
A curriculum that introduces language naturally through context — not rote memorisation

Parents get peace of mind knowing their child is engaging with their heritage language in a safe, structured, and joyful environment.


✨ Core Features
For Kids
🗺️ Learning Worlds — explore themed worlds, each with multiple paths and levels
🎯 15 Activity Types — matching, listening, picture quizzes, word building, and more
🤖 Mascot Guide — a friendly character that speaks, guides, and encourages throughout
⭐ XP & Levels — earn experience points and level up through the curriculum
🎉 Celebrations — rewarding moments when levels and paths are completed
🔊 Audio Rich — native Kannada pronunciations for every word and letter

For Parents
👨‍👩‍👧 Multiple Child Profiles — manage up to 3 child profiles under one parent account
📊 Progress Tracking — see which levels and paths each child has completed
🔒 Safe Environment — no ads, no in-app purchases, no external links

For the Platform
🌐 Content-Driven Architecture — new lessons and worlds added without code changes
🌍 Multi-Language Ready — localized content map supports future expansion to other Indian regional languages
📱 Web First — works on any device with a browser


🛠️ Technology Stack
| Layer	            | Technology               |	Purpose |
| ----------------- | ------------------------ | ---------------------------------------------- |
| Frontend	        | React.js	               | Web UI, cross-device experience                |
| State Management	| TanStack Query	         | Intelligent caching, minimal API calls         |
| Backend	          | Spring Boot 4 (Java 21)  |	REST API, business logic, data orchestration  |
| Primary Database	| MongoDB Atlas	           | Flexible content schema, progress tracking     |
| Authentication    |	Supabase Auth            |	Secure parent login, session management       |
| File Storage	    | Supabase Storage         |	CDN delivery for images and audio assets      |
| API Documentation |	SpringDoc OpenAPI	       | Swagger UI for API reference                   |
| Frontend Hosting	| Vercel	                 | Fast, global React deployment                  |
| Backend Hosting	  | Railway	                 | Spring Boot deployment                         |


🏗️ Key Architectural Decisions
MongoDB for Flexible Content Schema

Lessons, activities, and assets are stored as documents. New content — letters, words, game configurations — can be added or modified without code changes or redeployment. The game engine adapts to whatever data it receives, making Nyanafy a truly content-driven platform.

Supabase Auth — Identity Separated from Application Data

Authentication is handled entirely by Supabase, returning a signed JWT (ES256) on login. Spring Boot validates this token independently without storing passwords or managing sessions. Application data (profiles, progress, content) lives in MongoDB, cleanly separated from identity concerns.

Supabase Storage — Binary Assets Outside the Database

Images (SVG illustrations) and audio files (Kannada pronunciations) are stored in Supabase Storage and delivered via CDN. Only storage keys are stored in MongoDB — the API resolves these to full CDN URLs before returning to the frontend. This keeps documents small and asset delivery fast globally.

Aggregation APIs — One Call Per Screen

Rather than having React make multiple sequential API calls per page, Spring Boot aggregates data from multiple MongoDB collections server-side and returns a single, complete response per screen. This is particularly important for kids on mobile devices with slower connections.

Localized Content Map — Multi-Language by Design

Every learning asset stores language-specific content in a map keyed by language code:

"kn" → Kannada content (text, transliteration, audio)
"hi" → Hindi content (future)
"te" → Telugu content (future)

Adding a new language requires only new content entries — zero schema changes, zero code changes.

Separate Progress Collections — Concurrent Learning Paths

Progress is tracked independently across three collections:

learner_progress — overall XP and summary stats
learner_level_progress — completion status per level
learner_path_progress — completion status per path

This allows a child to progress simultaneously across multiple paths and worlds without data conflicts or messy queries.

TanStack Query Caching — Performance by Default

The React frontend caches all API responses intelligently. Data is only re-fetched when something meaningful changes (activity completed, level finished) — not on every navigation event. This prevents unnecessary API calls as kids switch between screens repeatedly during a session.


👤 User Flows
Parent Account
Register → Choose "For my child(ren)" → Add up to 3 child profiles

Login → Profile Selector → Choose child → Dashboard

Self Learner (12+)
Register → Choose "For myself" → Dashboard (direct, no profile selector)

Learning Flow
Dashboard → World Map → Select Path → Select Level
→ Complete Activities (15 per level) → Celebration Screen
→ Level marked complete → Next level unlocked → Continue
