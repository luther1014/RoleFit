# JobFit Analyzer

A full-stack learning project for practicing **Python backend APIs, React frontend development, SQL databases, Redis queues, worker processes, Docker Compose, authentication, testing, and cloud-ready architecture**.

This README is the **source of truth** for the project. Use it with Codex or any coding assistant as the main specification. The goal is not to build a huge product. The goal is to learn how a realistic full-stack, data-processing application is structured.

---

## 1. Project Summary

**JobFit Analyzer** helps users compare their CV/resume against a job description.

A user can:

1. Register and log in.
2. Upload or paste their CV.
3. Paste a job description.
4. Create an analysis job.
5. Wait while the system analyzes the match in the background.
6. View results such as:
   - match score,
   - matched skills,
   - missing skills,
   - suggested improvements,
   - possible interview questions.

The project should use a background worker and Redis queue so that the API does not process the analysis directly inside the request.

---

## 2. Main Learning Goals

By building this project, I want to learn:

- How to build REST APIs with **FastAPI**.
- How to design a small backend with clean structure.
- How to use **PostgreSQL** with SQLAlchemy.
- How to use **Redis as a queue**.
- How a separate **worker process** handles background jobs.
- How to build a frontend using **React + TypeScript**.
- How frontend and backend communicate through API calls.
- How authentication with **JWT** works.
- How to run a multi-service app using **Docker Compose**.
- How to write basic automated tests with **pytest**.
- How to prepare a project that looks cloud-ready and production-aware.
- How to explain this architecture in a job interview.

---

## 3. High-Level Architecture

```text
User Browser
   |
   v
React Frontend
   |
   v
FastAPI Backend
   |
   +------------------> PostgreSQL Database
   |
   +------------------> Redis Queue
                            |
                            v
                      Worker Process
                            |
                            v
                    PostgreSQL Database
```

### Simple explanation

- The **frontend** is what the user sees.
- The **backend API** receives requests from the frontend.
- The **database** stores users, analysis jobs, and results.
- **Redis** stores background tasks temporarily.
- The **worker** takes tasks from Redis and processes them.
- The worker saves final results back into PostgreSQL.

---

## 4. Core Concept: Queue and Worker

The analysis should not happen directly inside the API request.

Bad approach:

```text
User submits CV + JD
→ API receives request
→ API analyzes everything immediately
→ User waits
→ API returns result
```

Better approach:

```text
User submits CV + JD
→ API creates analysis job with status "pending"
→ API puts job into Redis queue
→ API immediately returns job_id
→ Worker picks up job in the background
→ Worker updates status to "processing"
→ Worker analyzes CV and JD
→ Worker saves result
→ Worker updates status to "completed"
→ Frontend shows the result
```

This is important because real applications often process slow tasks asynchronously.

---

## 5. Tech Stack

### Backend

- Python
- FastAPI
- SQLAlchemy
- Alembic
- PostgreSQL
- Redis
- RQ or Celery
- Pydantic
- Pytest

Recommended for this project: **RQ** because it is simpler than Celery for a weekend/learning project.

### Frontend

- React
- TypeScript
- Vite
- React Router
- Axios or Fetch API
- Basic CSS / Tailwind optional

### DevOps / Tooling

- Docker
- Docker Compose
- Git
- GitHub Actions for basic CI
- `.env` files for configuration

---

## 6. Services in Docker Compose

Docker Compose should run multiple services:

```text
frontend   → React application
backend    → FastAPI API server
worker     → background worker process
postgres   → PostgreSQL database
redis      → Redis queue
```

Yes, Docker Compose creates separate containers for these services.

Important detail:

The backend and worker can use the same backend codebase/image but run different commands.

Example:

```yaml
backend:
  build: ./backend
  command: uvicorn app.main:app --host 0.0.0.0 --port 8000

worker:
  build: ./backend
  command: rq worker analysis_jobs --url redis://redis:6379
```

The backend serves HTTP requests.

The worker waits for background jobs.

---

## 7. MVP Scope

The first version should be small but complete.

### MVP Features

- User registration
- User login
- JWT authentication
- Create analysis job
- Store CV text and job description text
- Enqueue job in Redis
- Worker processes the job
- Save analysis result
- Dashboard with list of analyses
- Detail page with result
- Docker Compose setup
- Basic backend tests
- Clear README

### Not required for MVP

- Real AI/LLM integration
- PDF parsing
- Beautiful UI
- Payment system
- Email notifications
- Advanced role-based permissions
- Complex deployment

---

## 8. Suggested Folder Structure

```text
jobfit-analyzer/
│
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── database.py
│   │   │
│   │   ├── auth/
│   │   │   ├── router.py
│   │   │   ├── schemas.py
│   │   │   ├── service.py
│   │   │   └── security.py
│   │   │
│   │   ├── analyses/
│   │   │   ├── router.py
│   │   │   ├── schemas.py
│   │   │   ├── service.py
│   │   │   └── analyzer.py
│   │   │
│   │   ├── models/
│   │   │   ├── user.py
│   │   │   ├── analysis_job.py
│   │   │   └── analysis_result.py
│   │   │
│   │   ├── worker/
│   │   │   ├── queue.py
│   │   │   └── tasks.py
│   │   │
│   │   └── tests/
│   │       ├── test_auth.py
│   │       └── test_analyses.py
│   │
│   ├── alembic/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── alembic.ini
│
├── frontend/
│   ├── src/
│   │   ├── api/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── routes/
│   │   └── main.tsx
│   │
│   ├── Dockerfile
│   ├── package.json
│   └── vite.config.ts
│
├── docker-compose.yml
├── .env.example
├── README.md
└── .github/
    └── workflows/
        └── backend-tests.yml
```

---

## 9. Database Design

### users

Stores registered users.

```text
id
email
hashed_password
created_at
```

### analysis_jobs

Stores each analysis request.

```text
id
user_id
title
cv_text
job_description_text
status
error_message
created_at
updated_at
```

Possible status values:

```text
pending
processing
completed
failed
```

### analysis_results

Stores the final result of an analysis.

```text
id
job_id
match_score
matched_skills_json
missing_skills_json
suggestions_json
interview_questions_json
created_at
```

---

## 10. API Endpoints

### Auth

```http
POST /auth/register
POST /auth/login
GET /auth/me
```

### Analyses

```http
POST /analyses
GET /analyses
GET /analyses/{analysis_id}
GET /analyses/{analysis_id}/result
DELETE /analyses/{analysis_id}
```

---

## 11. Example API Flow

### 1. Register

```http
POST /auth/register
```

Request:

```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

### 2. Login

```http
POST /auth/login
```

Response:

```json
{
  "access_token": "jwt_token_here",
  "token_type": "bearer"
}
```

### 3. Create analysis

```http
POST /analyses
Authorization: Bearer jwt_token_here
```

Request:

```json
{
  "title": "Full Stack Engineer - Python Cloud APIs",
  "cv_text": "My CV text here...",
  "job_description_text": "The job description text here..."
}
```

Response:

```json
{
  "id": 1,
  "title": "Full Stack Engineer - Python Cloud APIs",
  "status": "pending"
}
```

### 4. Worker processes the job

The worker changes the job status:

```text
pending → processing → completed
```

or:

```text
pending → processing → failed
```

### 5. Get result

```http
GET /analyses/1/result
Authorization: Bearer jwt_token_here
```

Response:

```json
{
  "match_score": 72,
  "matched_skills": ["Python", "React", "Docker", "SQL"],
  "missing_skills": ["FastAPI", "AWS", "message queues"],
  "suggestions": [
    "Add a backend project using FastAPI.",
    "Mention Docker Compose experience more clearly.",
    "Add a project showing Redis queue and worker architecture."
  ],
  "interview_questions": [
    "How have you used Docker in your projects?",
    "Can you explain how a REST API works?",
    "What is your experience with SQL databases?"
  ]
}
```

---

## 12. Analysis Logic for MVP

Do not use an LLM in the first version.

Use simple keyword matching.

Create a predefined list of skills:

```python
SKILLS = [
    "python",
    "fastapi",
    "django",
    "flask",
    "react",
    "typescript",
    "javascript",
    "postgresql",
    "sql",
    "redis",
    "docker",
    "git",
    "ci/cd",
    "aws",
    "azure",
    "gcp",
    "rest api",
    "authentication",
    "jwt",
    "message queue",
    "event-driven architecture",
    "security",
    "owasp"
]
```

### Algorithm

1. Convert CV text to lowercase.
2. Convert job description text to lowercase.
3. Find skills mentioned in the job description.
4. Find skills mentioned in the CV.
5. Calculate overlap.
6. Match score:

```text
matched required skills / total required skills * 100
```

7. Missing skills:

```text
skills in job description but not in CV
```

8. Generate simple suggestions based on missing skills.

Example:

```text
If "fastapi" is missing:
"Build and mention a small FastAPI REST API project."
```

---

## 13. Frontend Pages

### Login Page

- Email input
- Password input
- Login button
- Link to register

### Register Page

- Email input
- Password input
- Create account button

### Dashboard Page

Shows all analysis jobs.

Columns/cards:

```text
Title
Status
Created date
Match score if completed
Open details button
```

### Create Analysis Page

Form fields:

```text
Title
CV text
Job description text
Submit button
```

### Analysis Detail Page

Shows:

```text
Title
Status
Match score
Matched skills
Missing skills
Suggestions
Interview questions
```

If status is `pending` or `processing`, the frontend should poll the backend every few seconds.

---

## 14. Frontend Polling Concept

After creating an analysis job, the frontend can call:

```http
GET /analyses/{analysis_id}
```

every 2–5 seconds.

When status becomes `completed`, call:

```http
GET /analyses/{analysis_id}/result
```

This teaches how frontend apps handle background jobs.

---

## 15. Environment Variables

Create `.env.example`:

```env
DATABASE_URL=postgresql+psycopg2://postgres:postgres@postgres:5432/jobfit
REDIS_URL=redis://redis:6379
JWT_SECRET_KEY=change_me
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=60
```

Never commit real secrets.

---

## 16. Docker Compose Example

```yaml
services:
  backend:
    build: ./backend
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql+psycopg2://postgres:postgres@postgres:5432/jobfit
      REDIS_URL: redis://redis:6379
      JWT_SECRET_KEY: dev_secret
      JWT_ALGORITHM: HS256
      ACCESS_TOKEN_EXPIRE_MINUTES: 60
    volumes:
      - ./backend:/app
    depends_on:
      - postgres
      - redis

  worker:
    build: ./backend
    command: rq worker analysis_jobs --url redis://redis:6379
    environment:
      DATABASE_URL: postgresql+psycopg2://postgres:postgres@postgres:5432/jobfit
      REDIS_URL: redis://redis:6379
      JWT_SECRET_KEY: dev_secret
      JWT_ALGORITHM: HS256
      ACCESS_TOKEN_EXPIRE_MINUTES: 60
    volumes:
      - ./backend:/app
    depends_on:
      - postgres
      - redis

  frontend:
    build: ./frontend
    command: npm run dev -- --host 0.0.0.0
    ports:
      - "5173:5173"
    volumes:
      - ./frontend:/app
      - /app/node_modules
    depends_on:
      - backend

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: jobfit
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

---

## 17. How to Run Locally

From the project root:

```bash
docker compose up --build
```

Backend:

```text
http://localhost:8000
```

FastAPI docs:

```text
http://localhost:8000/docs
```

Frontend:

```text
http://localhost:5173
```

PostgreSQL runs inside Docker.

Redis runs inside Docker.

---

## 18. Implementation Plan

### Phase 1: Project setup

- Create backend folder.
- Create FastAPI app.
- Add health check endpoint.
- Create frontend with Vite + React + TypeScript.
- Add Docker Compose with backend, frontend, postgres, redis.

Goal:

```text
docker compose up --build works
```

### Phase 2: Database setup

- Add SQLAlchemy.
- Add PostgreSQL connection.
- Create User, AnalysisJob, and AnalysisResult models.
- Add Alembic migrations.

Goal:

```text
Database tables are created correctly.
```

### Phase 3: Authentication

- Register user.
- Hash password.
- Login user.
- Return JWT token.
- Protect analysis endpoints.

Goal:

```text
Only logged-in users can create and view their analyses.
```

### Phase 4: Analysis job creation

- Create POST /analyses endpoint.
- Save analysis job with status `pending`.
- Return job ID.

Goal:

```text
A logged-in user can create an analysis job.
```

### Phase 5: Redis queue and worker

- Add Redis connection.
- Add RQ queue.
- Enqueue analysis job.
- Create worker task.
- Worker updates status to `processing`.
- Worker runs analysis.
- Worker saves result.
- Worker updates status to `completed`.

Goal:

```text
Analysis happens in the background.
```

### Phase 6: Frontend

- Login/register pages.
- Dashboard page.
- Create analysis page.
- Analysis detail page.
- Poll job status.

Goal:

```text
User can use the app from the browser.
```

### Phase 7: Testing

Add basic tests for:

- Register
- Login
- Create analysis
- Get analyses
- Analysis algorithm

Goal:

```text
pytest passes.
```

### Phase 8: Polish

- Add README screenshots if possible.
- Add GitHub Actions.
- Improve error handling.
- Add simple styling.
- Add architecture diagram in README.

---

## 19. Testing Plan

### Backend tests

Use pytest.

Test examples:

```text
test_register_user
test_login_user
test_create_analysis_requires_auth
test_create_analysis_success
test_analysis_algorithm_finds_missing_skills
```

### Manual test scenario

1. Start app with Docker Compose.
2. Register user.
3. Login.
4. Create analysis with a sample CV and JD.
5. Confirm job appears as pending.
6. Confirm worker picks it up.
7. Confirm status changes to completed.
8. Confirm result is visible in frontend.

---

## 20. Security Basics to Practice

This project should include basic security concepts:

- Passwords must be hashed, not stored as plain text.
- JWT secret should come from environment variables.
- Protected endpoints should require authentication.
- Users should only access their own analyses.
- Input length should be limited.
- Do not commit secrets.
- CORS should be configured intentionally.
- API errors should not expose sensitive internal details.

---

## 21. Cloud-Ready Notes

This project does not need to be deployed immediately, but it should be designed as if it could be deployed.

Good practices:

- Use environment variables.
- Use Dockerfiles.
- Use Docker Compose for local development.
- Keep backend stateless.
- Store data in PostgreSQL.
- Store background jobs in Redis.
- Separate API process from worker process.
- Add health check endpoint.
- Add basic tests.
- Add GitHub Actions.

---

## 22. Possible Future Improvements

Only add these after the MVP works.

### Better CV input

- PDF upload
- DOCX upload
- Text extraction

### Better analysis

- Weighted skill scoring
- Experience level detection
- Project relevance scoring
- Keyword frequency
- LLM-based suggestions

### Better product features

- Save multiple CV versions
- Compare one CV against many jobs
- Track applications
- Generate cover letters
- Generate interview prep plans

### Better architecture

- Celery instead of RQ
- Object storage for uploaded files
- API rate limiting
- Admin dashboard
- Monitoring and logs
- Deployment to Azure, AWS, or GCP

---

## 23. Interview Explanation

A good short explanation:

```text
JobFit Analyzer is a full-stack application that compares a user's CV with a job description. I built it to practice production-style backend architecture. The frontend is built with React and TypeScript. The backend uses FastAPI and PostgreSQL. When a user creates an analysis, the API saves a job with pending status and pushes it into a Redis queue. A separate worker process consumes the job, analyzes the CV and job description, saves the result in PostgreSQL, and updates the job status. The whole system runs with Docker Compose using separate containers for frontend, backend, worker, PostgreSQL, and Redis.
```

---

## 24. What This Project Demonstrates

This project demonstrates:

- REST API design
- Authentication
- SQL database modeling
- Background processing
- Redis queue usage
- Worker process architecture
- Docker Compose multi-service setup
- React frontend development
- Async job status polling
- Basic testing
- Cloud-ready thinking

---

## 25. Definition of Done

The project is considered complete when:

- `docker compose up --build` starts all services.
- User can register and log in.
- User can create a CV/JD analysis.
- Backend creates job with status `pending`.
- Redis receives the job.
- Worker processes the job.
- PostgreSQL stores the result.
- Frontend shows the completed result.
- User can view previous analyses.
- Basic tests pass.
- README explains the architecture clearly.

---

## 26. Learning Rule

Do not blindly copy code.

For every major feature, understand:

1. What problem it solves.
2. Which file contains the logic.
3. How data flows through the system.
4. How frontend and backend communicate.
5. How Docker Compose connects the services.
6. How the worker receives and processes jobs.
7. How the database stores the final result.

This project is not only for building something. It is for learning how real full-stack systems are structured.
