# COMP3011

I forgot to intially push things into my repository, but please check TITAN history, which demonstrates that I have been adjusting the code gradually rather than submitting everything all at once. 

A Spring Boot web application that records audio from the browser
microphone, transcribes it via OpenAI's transcription API, and exposes
administrative/statistics endpoints per the assignment's YAML spec.

## Architecture

- **`HomeController` / `static/index.html`** — serves the single-page
  frontend. Uses the browser `MediaRecorder` API to capture audio and
  `fetch()` to upload it asynchronously.
- **`TranscriptionController` / `TranscriptionService`** — receives the
  uploaded audio, forwards it to OpenAI's `/v1/audio/transcriptions`
  endpoint using the `gpt-4o-transcribe` model, and returns the
  transcript. `gpt-4o-transcribe` was chosen over `whisper-1` because
  it returns real token usage data (required for `/api/v1/global/stats`)
  and produced more accurate transcriptions in testing.
- **`AdminController`** — implements `/api/v1/admin/uptime` and
  `/api/v1/admin/shutdown` per the YAML spec.
- **`ShutdownState`** — isolates the shutdown race-condition logic
  (an `AtomicBoolean.compareAndSet`) from the actual `System.exit()`
  side effect, so it can be tested independently without killing the
  test JVM.
- **`StatsController` / `TokenStats`** — tracks cumulative input/output
  token usage using `AtomicLong`, safe under concurrent access.
- **`ErrorResponse`** — standard error shape (timestamp, status, error,
  message, path) returned consistently across all endpoints per the
  YAML spec.

## Configuration

The OpenAI API key is read exclusively from the `OPENAI_API_KEY`
environment variable at runtime (`System.getenv(...)`). It is never
hardcoded, logged, or persisted anywhere in the codebase.

For local development, set it via your shell (`export
OPENAI_API_KEY=...`) or in your IDE's run configuration. On TITAN,
this is provided automatically by the grading environment.

## Running locally

JAVA_HOME=$(/usr/libexec/java_home -v 17) ./mvnw spring-boot:run

Then visit `http://localhost:8080/`.

## Building the executable JAR

JAVA_HOME=$(/usr/libexec/java_home -v 17) ./mvnw clean package
java -jar target/Assignment1-0.0.1-SNAPSHOT.jar

## Testing

Regression tests live in `src/test/java/comp3011/assignment1/`:

- **`TranscriptionControllerTest`** — verifies the transcription
  endpoint using a stubbed `RestTemplate` (no real calls made to
  OpenAI), covering both a successful transcription and a missing-file
  400 response. This proves controller correctness independent of
  external API availability.
- **`ConcurrencyTest`** — fires 250 concurrent requests at
  `/api/v1/admin/uptime` using a 50-thread pool, asserting all succeed
  within a bounded time window. This demonstrates the application
  handles the assignment's required load (200+ concurrent blocking
  requests) without crashing or significant delay.
- **`ShutdownStateTest`** — fires 100 threads simultaneously at
  `ShutdownState.tryBeginShutdown()`, asserting exactly one wins.
  This proves the shutdown endpoint's race-condition protection
  (`AtomicBoolean.compareAndSet`) is correct under genuine concurrent
  pressure, not just under casual manual testing.

## Known limitations / future improvements

- Transcription accuracy depends on OpenAI's model output and may
  differ slightly in punctuation/capitalization from expected text.
- No persistence layer — all stats reset on server restart, per spec.
