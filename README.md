# AI Interview Portal

AI Interview Portal is a Spring Boot + Thymeleaf interview practice application. It combines an authenticated dashboard, static practice questions, AI-generated interview rounds, live code execution, explainable scoring, and MySQL-backed progress tracking in one app.

## What The App Does

- lets users register, sign in, and practice in a protected workspace
- serves a static question bank for `java`, `python`, `c`, `sql`, and `dsa`
- generates dynamic interview questions with Groq through an OpenAI-compatible API
- uses resume text, job description text, weak skills, difficulty, and interview track to personalize prompts
- accepts text or code answers depending on the round type
- runs supported code through JDoodle for `java`, `python3`, and `c`
- stores generated prompts, evaluated attempts, feedback, and user performance snapshots in MySQL

## Main Features

### 1. AI Interview Lab

- technical and behavioral interview tracks
- text and code response modes
- Groq-backed LLM prompt generation with fallback to internal prompt catalogs
- adaptive follow-up questions tied to earlier attempts
- richer prompt rendering for generated questions, including fenced code blocks

### 2. Evaluation And Feedback

- keyword coverage scoring
- relevance, depth, communication, and behavioral structure scoring
- personalized strengths and improvement areas
- study-plan suggestions and follow-up recommendations
- server-side prompt lookup before evaluation, so scoring metadata is not trusted from the browser

### 3. Persistence And Analytics

- prompt log in `generated_interview_question`
- evaluated answers in `interview_attempt`
- latest user snapshot in `user_performance_profile`
- dashboard recommendations, readiness labels, recent-attempt trends, and open follow-up counts

### 4. Static Practice And Code Execution

- category-based question bank
- guided practice vs code-lab routing based on question type
- structured execution results with runtime and memory data
- clear provider-status handling for compile errors, runtime errors, timeouts, and missing JDoodle credentials

## Tech Stack

- Java 17
- Spring Boot 3.4.4
- Spring MVC
- Thymeleaf
- Spring Security
- Spring Data JPA / Hibernate
- MySQL
- H2 for tests
- Groq chat completions API
- JDoodle execution API

## Project Structure

```text
src/main/java/com/Portal/AI_Interview_Portal
├── config
├── controller
├── dto
├── entity
├── repository
└── service

src/main/resources
├── static
└── templates
```

## Package Guide

- `config`: security, LLM config, and code-execution config
- `controller`: page routes and form flows
- `dto`: form payloads and UI view models
- `entity`: JPA entities mapped to MySQL tables
- `repository`: Spring Data repositories
- `service`: generation, evaluation, analytics, execution, and auth logic

## Important Runtime Flow

1. A signed-in user opens `/interview-lab`.
2. The app collects category, difficulty, track, response mode, focus area, resume text, and JD text.
3. `QuestionGenerationService` asks Groq for a structured prompt. If that fails, it falls back to internal prompt catalogs.
4. The generated prompt is stored in `generated_interview_question`.
5. The user submits an answer.
6. `InterviewEvaluationService` reloads the stored prompt from MySQL by `generatedQuestionRecordId`, evaluates the answer, and saves the attempt transactionally.
7. `PerformanceInsightsService` refreshes the user snapshot and recommendations.

## Security And Reliability Notes

- registration uses a dedicated `RegistrationForm` and forces `ROLE_USER`
- duplicate email registration is handled cleanly in the UI
- logout uses Spring Security’s normal protected flow instead of a custom `GET /logout`
- evaluation no longer trusts hidden browser fields for the scoring rubric
- follow-up question generation reuses an existing stored follow-up instead of creating duplicates on refresh
- zero-score attempts are included in dashboard averages

## Database Tables

- `users`: registered users
- `question`: static practice questions
- `submission`: static code-practice submissions
- `generated_interview_question`: stored dynamic prompts
- `interview_attempt`: evaluated interview answers and scoring details
- `user_performance_profile`: latest per-user analytics snapshot

## Configuration

Main runtime properties live in `src/main/resources/application.properties`.

The app supports a local `.env.properties` file through:

```properties
spring.config.import=optional:file:.env.properties
```

Recommended local secrets/config:

```properties
APP_DB_URL=jdbc:mysql://localhost:3306/interview?createDatabaseIfNotExist=true&serverTimezone=UTC
APP_DB_USERNAME=root
APP_DB_PASSWORD=

APP_LLM_ENABLED=true
APP_LLM_BASE_URL=https://api.groq.com/openai/v1
APP_LLM_API_KEY=YOUR_GROQ_KEY
APP_LLM_MODEL=openai/gpt-oss-20b

APP_ADMIN_BOOTSTRAP_ENABLED=true
APP_ADMIN_NAME=Platform Admin
APP_ADMIN_EMAIL=admin@example.com
APP_ADMIN_PASSWORD=ChangeMe123!

APP_JDOODLE_CLIENT_ID=YOUR_JDOODLE_CLIENT_ID
APP_JDOODLE_CLIENT_SECRET=YOUR_JDOODLE_CLIENT_SECRET
```

Notes:

- if JDoodle credentials are missing, the UI still works but live code execution returns a clear configuration error
- if no `ROLE_ADMIN` exists and `APP_ADMIN_EMAIL` plus `APP_ADMIN_PASSWORD` are set, the app creates the first admin automatically on startup
- if the configured admin email already belongs to a normal user and no admin exists yet, startup promotes that account to `ROLE_ADMIN`
- `.env.properties` is local-only and should not be committed

## Run Locally

### Prerequisites

- Java 17
- Maven
- MySQL on `localhost:3306`

### Start

```bash
mvn spring-boot:run
```

Open:

```text
http://localhost:8096
```

## Run Tests

```bash
mvn -B -q test
```

## Current State

- full-width responsive UI with light/dark theme support
- prompt renderer for cleaner generated-question presentation
- Groq-backed dynamic question generation with fallback behavior
- server-side validated evaluation flow
- MySQL persistence for prompts, attempts, and performance profiles
- documented environment-based configuration for database and code execution

For the detailed implementation record, see `updates.md`.
