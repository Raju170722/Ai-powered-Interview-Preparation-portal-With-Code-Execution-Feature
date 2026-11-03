# AI Interview Portal - Project Notes And Updates

This file is the internal implementation record. Update it whenever code, setup, UI, tests, or behavior changes so it stays aligned with the real project.

## 1. Project Summary

AI Interview Portal is a Spring Boot interview-practice product with:

- authentication
- static question-bank practice
- dynamic AI interview rounds
- live code execution
- explainable answer scoring
- MySQL persistence for prompts, attempts, and user performance

In simple terms, the app helps a user practice interview questions, get feedback, and return for more targeted follow-up rounds based on earlier performance.

## 2. Current Architecture

### 2.1 Main stack

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

### 2.2 Package structure

| Package | Purpose |
| --- | --- |
| `config` | Security, LLM config, and code-execution config |
| `controller` | HTTP routes and page flow |
| `dto` | Form models, UI payloads, and view DTOs |
| `entity` | JPA database entities |
| `repository` | Spring Data JPA repositories |
| `service` | Business logic for generation, evaluation, analytics, execution, and auth |

## 3. End-To-End App Flow

### 3.1 User entry flow

Public pages:

- `/`
- `/login`
- `/register`

Protected pages:

- `/dashboard`
- `/questions`
- `/code-editor`
- `/interview-lab`
- `/submissions`

### 3.2 Dynamic interview flow

1. User opens `/interview-lab`.
2. User selects category, difficulty, response mode, and interview track.
3. User can optionally provide focus area, resume text, and JD text.
4. `QuestionGenerationService` builds a request context using:
   - category
   - difficulty
   - interview track
   - weak skills from earlier attempts
   - resume text
   - job description text
   - prior attempt feedback
5. The app calls `LlmQuestionGenerationClient`.
6. If the LLM succeeds, the response is converted into `InterviewQuestionPayload`.
7. If the LLM fails or is disabled, the app falls back to internal prompt catalogs.
8. The generated prompt is stored in `generated_interview_question`.
9. The user submits a text or code answer.
10. `InterviewEvaluationService` reloads the stored prompt from MySQL by `generatedQuestionRecordId`, evaluates the answer, and saves the result transactionally.
11. `PerformanceInsightsService` refreshes the user’s stored performance profile.

### 3.3 Static practice flow

The static question bank serves `java`, `python`, `c`, `sql`, and `dsa`.

The app decides whether a question should open:

- the code lab
- or a guided explanation flow

Behavioral practice is handled in the AI Interview Lab rather than the static code bank.

## 4. AI Features Explained Simply

### 4.1 Dynamic prompt generation

The app can generate a complete interview question from:

- category
- difficulty
- interview track
- weak skills
- resume text
- JD text
- prior attempt feedback

Primary path:

- Groq via OpenAI-compatible `/chat/completions`

Fallback path:

- internal prompt blueprints stored in Java service logic

### 4.2 Adaptive follow-up generation

After an attempt is scored, the app can generate a follow-up question using:

- the earlier question
- the user’s weak areas
- recurrence tags
- focus skill
- earlier feedback

The current implementation prevents duplicate follow-up rows on refresh by reusing an existing stored follow-up for the same parent attempt.

### 4.3 Explainable evaluation

The app mostly uses local scoring logic, not a second LLM evaluator.

The evaluator checks:

- expected keyword coverage
- response relevance
- response depth
- communication clarity
- behavioral structure for behavioral rounds

This makes the scoring easier to understand and less unpredictable than a fully LLM-judged evaluation loop.

### 4.4 Prompt rendering in the UI

Generated prompts can contain fenced code blocks.

The UI now renders them cleanly by:

- keeping the raw prompt text in a hidden field
- using `prompt-render.js` to parse fenced sections
- presenting prose as readable paragraphs
- presenting code snippets in a styled code shell

This fixes the earlier issue where raw triple-backticks were shown directly in the browser.

### 4.5 Live code execution

For supported coding flows, the app sends code to JDoodle and stores:

- execution output
- status label
- CPU time
- memory
- compilation status

Supported languages:

- `java`
- `python3`
- `c`

If JDoodle credentials are missing, the app returns a clear configuration message instead of failing silently.

## 5. Dependencies From `pom.xml`

| Dependency | Why it is used |
| --- | --- |
| `spring-boot-starter-web` | MVC controllers, routing, HTTP handling |
| `spring-boot-starter-thymeleaf` | Server-rendered HTML pages |
| `spring-boot-starter-data-jpa` | ORM and repository access |
| `spring-boot-starter-security` | Authentication and route protection |
| `spring-boot-devtools` | Local development support |
| `mysql-connector-j` | MySQL connectivity |
| `spring-boot-starter-test` | Test support |
| `spring-security-test` | Security test helpers |
| `h2` | In-memory test database |

## 6. Main Files And What They Do

### 6.1 Config files

- `src/main/java/com/Portal/AI_Interview_Portal/config/SecurityConfig.java`
  - protects the authenticated app
  - permits public assets
  - keeps login/logout inside Spring Security
- `src/main/java/com/Portal/AI_Interview_Portal/config/LlmQuestionGenerationProperties.java`
  - binds LLM settings such as base URL, API key, model, and temperature
- `src/main/java/com/Portal/AI_Interview_Portal/config/CodeExecutionProperties.java`
  - binds JDoodle execution settings from environment/config

### 6.2 Controllers

- `src/main/java/com/Portal/AI_Interview_Portal/controller/AuthController.java`
  - home, login, register, and dashboard entry
  - uses `RegistrationForm` instead of binding the user entity directly
- `src/main/java/com/Portal/AI_Interview_Portal/controller/InterviewController.java`
  - AI Interview Lab
  - prompt generation
  - answer evaluation using stored prompt IDs
- `src/main/java/com/Portal/AI_Interview_Portal/controller/QuestionController.java`
  - static question-bank browsing
  - code-lab vs guided-practice routing
- `src/main/java/com/Portal/AI_Interview_Portal/controller/CodeController.java`
  - code execution submission/result flow
- `src/main/java/com/Portal/AI_Interview_Portal/controller/SubmissionsController.java`
  - history pages for attempts, stored prompts, and performance profile

### 6.3 DTOs

- `src/main/java/com/Portal/AI_Interview_Portal/dto/InterviewQuestionPayload.java`
  - carries the generated prompt and related metadata
- `src/main/java/com/Portal/AI_Interview_Portal/dto/CodeExecutionResult.java`
  - structured result returned by live code execution
- `src/main/java/com/Portal/AI_Interview_Portal/dto/RegistrationForm.java`
  - safe registration payload with name, email, and password
- `src/main/java/com/Portal/AI_Interview_Portal/dto/QuestionCardView.java`
  - question-bank card model including `supportsCodeLab`
- `src/main/java/com/Portal/AI_Interview_Portal/dto/PerformanceSnapshot.java`
  - dashboard summary model

### 6.4 Services

- `src/main/java/com/Portal/AI_Interview_Portal/service/QuestionGenerationService.java`
  - central prompt-generation logic
  - Groq first, fallback second
  - follow-up deduplication
  - prompt storage
- `src/main/java/com/Portal/AI_Interview_Portal/service/LlmQuestionGenerationClient.java`
  - low-level Groq/OpenAI-compatible API client
  - structured response parsing
- `src/main/java/com/Portal/AI_Interview_Portal/service/InterviewEvaluationService.java`
  - reloads the stored prompt by ID
  - evaluates answers
  - saves the attempt transactionally
  - links prompt and attempt
- `src/main/java/com/Portal/AI_Interview_Portal/service/NlpAnalysisService.java`
  - local heuristic scoring and feedback logic
- `src/main/java/com/Portal/AI_Interview_Portal/service/PerformanceInsightsService.java`
  - dashboard analytics and stored performance profiles
  - includes zero-score rounds in averages
- `src/main/java/com/Portal/AI_Interview_Portal/service/CodeExecutionService.java`
  - JDoodle integration
  - language normalization
  - provider error classification
- `src/main/java/com/Portal/AI_Interview_Portal/service/CustomUserDetailsService.java`
  - Spring Security user lookup by email
- `src/main/java/com/Portal/AI_Interview_Portal/service/userservice.java`
  - registration logic
  - duplicate email validation
  - forced `ROLE_USER`

### 6.5 Entities

- `src/main/java/com/Portal/AI_Interview_Portal/entity/user.java`
  - registered user account
- `src/main/java/com/Portal/AI_Interview_Portal/entity/Question.java`
  - static question-bank row
- `src/main/java/com/Portal/AI_Interview_Portal/entity/Submission.java`
  - static code practice submission
- `src/main/java/com/Portal/AI_Interview_Portal/entity/GeneratedInterviewQuestion.java`
  - stored generated prompt
- `src/main/java/com/Portal/AI_Interview_Portal/entity/InterviewAttempt.java`
  - evaluated response with score, analysis, feedback, and execution data
- `src/main/java/com/Portal/AI_Interview_Portal/entity/UserPerformanceProfile.java`
  - latest persisted performance summary per user

### 6.6 Repositories

- `src/main/java/com/Portal/AI_Interview_Portal/repository/UserRepository.java`
  - case-insensitive user lookup and duplicate checks
- `src/main/java/com/Portal/AI_Interview_Portal/repository/QuestionRepository.java`
  - static question-bank access
- `src/main/java/com/Portal/AI_Interview_Portal/repository/SubmissionRepository.java`
  - static coding submission history
- `src/main/java/com/Portal/AI_Interview_Portal/repository/GeneratedInterviewQuestionRepository.java`
  - generated prompt storage and follow-up lookup
- `src/main/java/com/Portal/AI_Interview_Portal/repository/InterviewAttemptRepository.java`
  - evaluated attempt history
- `src/main/java/com/Portal/AI_Interview_Portal/repository/UserPerformanceProfileRepository.java`
  - stored performance profile access

### 6.7 Templates

- `src/main/resources/templates/index.html`
  - public landing page
- `src/main/resources/templates/login.html`
  - sign-in page
- `src/main/resources/templates/register.html`
  - sign-up page with duplicate-email feedback
- `src/main/resources/templates/dashboard.html`
  - dashboard, recommendations, and logout form
- `src/main/resources/templates/questions.html`
  - static question-bank page
- `src/main/resources/templates/code-editor.html`
  - code lab UI
- `src/main/resources/templates/code-result.html`
  - static execution result page
- `src/main/resources/templates/interview-lab.html`
  - dynamic prompt generation and response studio
  - rich prompt renderer target
- `src/main/resources/templates/interview-feedback.html`
  - scored result and next steps
- `src/main/resources/templates/submissions.html`
  - attempt history, prompt log, and saved profile details

### 6.8 Static assets

- `src/main/resources/static/app-ui.css`
  - authenticated-app styling, response studio, prompt rendering styles
- `src/main/resources/static/public-ui.css`
  - public-page styling
- `src/main/resources/static/theme.js`
  - light/dark theme toggler and persistence
- `src/main/resources/static/prompt-render.js`
  - converts fenced prompt content into readable UI blocks

### 6.9 Runtime config files

- `src/main/resources/application.properties`
  - main app configuration
  - environment-backed DB and JDoodle settings
  - Groq defaults for LLM generation
- `src/test/resources/application-test.properties`
  - H2-based test configuration
- `.gitignore`
  - keeps `.env.properties` local

## 7. Database Design

Main tables:

- `users`
- `question`
- `submission`
- `generated_interview_question`
- `interview_attempt`
- `user_performance_profile`

### 7.1 What each table stores

- `users`
  - account data and role
- `question`
  - static question-bank content
- `submission`
  - static code-practice history
- `generated_interview_question`
  - generated prompt, source label, focus skill, context, response mode, round number, follow-up linkage
- `interview_attempt`
  - stored answer, analysis, feedback, scoring breakdown, follow-up data, and execution output
- `user_performance_profile`
  - latest snapshot of score trends, weaknesses, recommendations, and milestone text

### 7.2 Important stored AI data

The project stores:

- AI-generated prompts
- user answers
- analysis summaries
- personalized feedback
- prompt-to-attempt linkage
- user-specific performance snapshots

## 8. Configuration And Secrets

Runtime config is environment-driven where it matters.

### 8.1 Database

`application.properties` uses:

- `APP_DB_URL`
- `APP_DB_USERNAME`
- `APP_DB_PASSWORD`

Default local DB target:

- MySQL database `interview`
- port `3306`

### 8.2 Groq

LLM prompt generation uses:

- `APP_LLM_ENABLED`
- `APP_LLM_BASE_URL`
- `APP_LLM_API_KEY`
- `APP_LLM_MODEL`
- `APP_LLM_TEMPERATURE`

Default provider target:

- `https://api.groq.com/openai/v1`

### 8.3 JDoodle

Live code execution uses:

- `APP_CODE_EXECUTION_BASE_URL`
- `APP_JDOODLE_CLIENT_ID`
- `APP_JDOODLE_CLIENT_SECRET`

### 8.4 Local secret file

The app supports local secrets through:

- `.env.properties`

This file is not meant to be committed.

## 9. Major Recent Updates

### 9.1 UI and UX updates

Implemented:

- full-width responsive layout
- smoother scrolling
- light/dark theme toggler
- improved response studio
- richer prompt rendering for generated code snippets
- guided-practice vs code-lab distinction in the static bank

Main files:

- `app-ui.css`
- `public-ui.css`
- `theme.js`
- `prompt-render.js`
- `index.html`
- `dashboard.html`
- `questions.html`
- `interview-lab.html`

### 9.2 Dynamic prompt generation updates

Implemented:

- Groq integration
- OpenAI-compatible request format
- resume + JD grounding
- weak-skill and prior-feedback context
- category-tight personalization for technical prompts
- follow-up prompt deduplication

Main files:

- `QuestionGenerationService.java`
- `LlmQuestionGenerationClient.java`
- `LlmQuestionGenerationProperties.java`
- `GeneratedInterviewQuestion.java`
- `GeneratedInterviewQuestionRepository.java`

### 9.3 Evaluation and persistence hardening

Implemented:

- evaluation based on stored prompt IDs instead of browser-hidden fields
- transactional attempt save flow
- prompt-to-attempt linking
- answered-state updates on prompt rows
- persisted user performance snapshots

Main files:

- `InterviewController.java`
- `InterviewEvaluationService.java`
- `InterviewAttempt.java`
- `UserPerformanceProfile.java`
- `PerformanceInsightsService.java`

### 9.4 Auth and configuration hardening

Implemented:

- dedicated `RegistrationForm`
- forced `ROLE_USER`
- duplicate-email validation
- environment-based DB config
- environment-based JDoodle config
- reduced SQL/bind logging
- POST-based logout flow

Main files:

- `AuthController.java`
- `userservice.java`
- `UserRepository.java`
- `SecurityConfig.java`
- `CodeExecutionProperties.java`
- `CodeExecutionService.java`
- `application.properties`

### 9.5 Analytics fix

Implemented:

- zero-score rounds now count in averages and trend snapshots

Main file:

- `PerformanceInsightsService.java`

## 10. Tests

Current tests:

- `AiInterviewPortalApplicationTests.java`
- `InterviewFeaturePagesTests.java`
- `CodeExecutionServiceTests.java`
- `LlmQuestionGenerationClientTests.java`
- `AiPersistenceStorageTests.java`
- `PerformanceInsightsServiceTests.java`

Coverage includes:

- application startup
- public/protected page rendering
- static asset access
- registration hardening
- prompt generation and evaluation flow
- persistence of prompts, attempts, and performance profiles
- code execution result handling
- LLM response parsing
- zero-score analytics handling

## 11. Operational Notes

- runtime port is `8096`
- default database name is `interview`
- Hibernate uses `ddl-auto=update`
- tests use H2
- `.env.properties` is supported for local-only secrets
- live code execution requires JDoodle credentials
- Groq-backed prompt generation is optional and falls back cleanly if disabled
- Mockito emits self-attach warnings under the current JDK during tests, but the suite passes

## 12. Simple Project Explanation

In simple words, this project is:

- an interview practice portal
- where users can do static practice or AI-generated interview rounds
- get feedback on their answers
- run code for supported languages
- and store their progress in MySQL so the next round can be more personalized

Recent important fixes:

- the evaluation flow now trusts the database, not hidden browser fields
- config secrets are externalized
- registration is safer
- prompt rendering is cleaner
- dashboard averages include failed rounds
