# Pathwright — AI Learning Path Generator

A full-stack app that generates a personalized, milestone-based learning roadmap
using an LLM, and tracks your progress as you complete each step.

**Stack:** Java 17 + Spring Boot 3 + Spring Data JPA + MySQL + vanilla HTML/CSS/JS frontend.

---

## Architecture (30-second version)

```
Browser (static/index.html)
     │  fetch()
     ▼
RoadmapController / MilestoneController / UserController   (REST layer)
     │
     ▼
RoadmapService  ──calls──▶  LlmRoadmapService  ──HTTP──▶  OpenAI Chat Completions API
     │                            │
     │                     parses structured JSON (LlmRoadmapResponse DTO)
     │                     retries once if the model returns malformed JSON
     ▼
RoadmapRepository / MilestoneRepository / UserRepository   (Spring Data JPA)
     │
     ▼
MySQL  (users → roadmaps → milestones, 1:many:many)
```

The key design decision: the LLM only ever produces the *initial* roadmap.
Once milestones are persisted as rows in the `milestones` table, all progress
tracking (checking things off) is normal CRUD against the database — the LLM
is never called again for that. This keeps costs low and the app usable
offline once a roadmap exists.

---

## Prerequisites

- Java 17+ (`java -version` to check)
- Maven 3.8+ (`mvn -version` to check)
- MySQL 8+ running locally (or update `application.properties` to point elsewhere)
- An OpenAI API key (or swap the base URL/model in `application.properties`
  for any OpenAI-compatible provider, e.g. Groq, together.ai, or a local
  Ollama server with an OpenAI-compatible endpoint)

## Setup

**1. Create the database** (or let it auto-create — see below)

```sql
CREATE DATABASE learning_path_db;
```

Actually, `application.properties` already has `createDatabaseIfNotExist=true`,
so this step is optional as long as your MySQL user has permission to create databases.

**2. Set your environment variables**

```bash
export DB_USERNAME=root
export DB_PASSWORD=your_mysql_password
export OPENAI_API_KEY=sk-...your-key...
```

(On Windows: use `set VAR=value` in cmd, or `$env:VAR="value"` in PowerShell.)

**3. Run it**

```bash
mvn spring-boot:run
```

**4. Open it**

Go to [http://localhost:8080](http://localhost:8080) — the frontend is served
directly from Spring Boot's static resources, no separate frontend server needed.

---

## API Reference

| Method | Endpoint                    | Purpose                                  |
|--------|------------------------------|-------------------------------------------|
| POST   | `/api/users`                 | Create or fetch a user by email          |
| POST   | `/api/roadmaps`               | Generate a new roadmap via LLM           |
| GET    | `/api/roadmaps/{id}`          | Fetch a roadmap with all milestones      |
| GET    | `/api/roadmaps/user/{userId}` | List all roadmaps for a user             |
| PATCH  | `/api/milestones/{id}`        | Toggle a milestone's completion status   |

Example: generate a roadmap

```bash
curl -X POST http://localhost:8080/api/roadmaps \
  -H "Content-Type: application/json" \
  -d '{
    "userId": 1,
    "currentSkills": "Core Java, basic HTML/CSS, some SQL",
    "targetRole": "Backend Developer"
  }'
```

---

## Things worth understanding before an interview

**Why is the LLM response parsed into a separate DTO (`LlmRoadmapResponse`)
instead of straight into the JPA entities?**
Because the LLM's output shape and the database schema are different
concerns. If OpenAI changes response formatting, or you swap providers, only
the DTO + parsing logic changes — the persistence layer is untouched. This
is a real "why did you structure it this way" interview question, and now
you have a real answer.

**What happens if the LLM returns invalid JSON?**
`LlmRoadmapService` tries to parse the response; if it fails, it retries once
with a stricter prompt reminder. If it still fails, it throws a clear
exception that the controller turns into a `502 Bad Gateway` with a friendly
error message — instead of crashing the whole app. This is deliberately the
first thing to point at when asked "how do you handle failures."

**Why is `resourceLinks` stored as a JSON string instead of its own table?**
A deliberate tradeoff: resources are small, always fetched with their parent
milestone, and never queried independently — so a normalized table would add
join overhead for no real benefit. This is worth being able to defend, not
just do by default.

**Ideas to extend it (good "what would you add next" answers):**
- Regenerate a single milestone if the user wants it swapped out
- Streaming the LLM response so the UI shows the roadmap building live
- Auth (Spring Security) instead of email-only user lookup
- Rate-limiting the `/api/roadmaps` endpoint (LLM calls cost money per hit)
- A `/api/roadmaps/{id}/regenerate` endpoint using the stored raw response
  as context to avoid a full re-prompt

---

## Project structure

```
src/main/java/com/learningpath/app/
├── AppApplication.java
├── model/          → JPA entities: User, Roadmap, Milestone
├── repository/      → Spring Data JPA repositories
├── service/          → LlmRoadmapService (LLM call + parsing), RoadmapService (orchestration), UserService
├── controller/        → REST endpoints
├── dto/                → Request/response shapes, including the LLM's structured JSON contract
└── config/              → Global exception handling
src/main/resources/
├── application.properties
└── static/index.html   → Frontend (no build step needed)
```
