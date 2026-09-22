# AI Nutrition Planner

A sample project demonstrating how to build AI agents with Spring AI.

**This is a copy of the Spring AI implementation from [SandraAhlgrimm/ai-nutrition-planner](https://github.com/SandraAhlgrimm/ai-nutrition-planner)**, a repository I created together with [Sandra Ahlgrimm](https://github.com/SandraAhlgrimm) for our talk comparing agentic Java frameworks. That repository implements the exact same nutrition planning use case three times — with Embabel, LangChain4j, and Spring AI — so the frameworks can be compared side by side. This repository extracts the Spring AI variant on its own; head over to the original if you want the comparison.

[Slides: Building AI Agents with Spring AI](slides.pdf)

## Use Case

The agent creates a personalized weekly meal plan:

1. Fetches the user profile and seasonal ingredients in parallel
2. Generates recipes for the requested days and meals using seasonal produce
3. Validates the plan against the user profile (allergens, calorie limits, dietary restrictions)
4. Revises recipes based on feedback and re-validates — looping until the plan passes or a maximum number of iterations is reached

## Agentic AI Patterns Implemented

| Pattern | Implementation |
|---------|----------------|
| **Parallel Execution** — independent steps run concurrently | `Workflow.parallel()` |
| **Validation / Reflection Loop** — iterate until output passes quality checks | `ValidationRetryAdvisor` |
| **Tool Use** — agent calls external functions | `.tools()` on `ChatClient` |
| **Tool Search** — dynamic tool discovery at runtime | `ToolSearchToolCallAdvisor` — separate search step discovers tools via metadata before each call |
| **Human-in-the-Loop** — pause workflow for user input | `AskUserQuestionTool` |
| **Agent Skills** — invoke pre-packaged executable skills | `SkillsTool` |
| **Multi-Agent Orchestration** — specialized sub-agents collaborate | custom orchestration in code — no direct framework support |
| **Persona** — role-based system prompts per agent | custom implementation via `.system()` on `ChatClient` |
| **MCP Server** — expose agent as a Model Context Protocol tool | `@McpTool` |

## Prerequisites

- **Java 25**
- **Maven** (Maven wrapper included)
- **Docker Desktop** (for Grafana observability stack and Ollama)
- An LLM provider: **Azure OpenAI**, **OpenAI**, or **Ollama** (local)

## Setup
### LLM Providers

**Ollama** (local, no API key required):

```bash
docker compose --profile ollama up -d
```

This starts Ollama and automatically pulls `qwen2.5`. 

**Azure OpenAI**:

```bash
export AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
export AZURE_OPENAI_API_KEY=your-api-key
export AZURE_OPENAI_DEPLOYMENT_NAME=gpt-4o
```

**OpenAI**:

```bash
export OPENAI_API_KEY=sk-...
```

## Run

Change into the implementation directory, set the Spring profile related to an LLM provider and start the app:

```bash
cd langchain4j   # or embabel, spring-ai
export SPRING_PROFILES_ACTIVE=ollama # or openai, azure
./mvnw spring-boot:run
```

The app starts on port `8080`. Basic auth: `alice` / `123456`. UI at [http://localhost:8080](http://localhost:8080), REST API at `http://localhost:8080/api/nutrition-plan`.

## Observability

A Grafana + OTLP stack (Loki, Tempo, Mimir) is included via Docker Compose:
```bash
docker compose --profile observability up -d
```

- **Grafana**: [http://localhost:3000](http://localhost:3000) (admin/admin)
- **OTLP collector**: `localhost:4318` (HTTP) / `localhost:4317` (gRPC)

To enable tracing and metrics export from any module, activate the `observability` profile 
in addition to the profile for the LLM provider of choice:

```bash
SPRING_PROFILES_ACTIVE=ollama,observability mvn spring-boot:run
```
The dashboard shows agent invocation rates, execution durations (p95), active agents, HTTP endpoint latency, JVM metrics, and distributed traces.

## Example Request

```bash
curl -s -X POST http://localhost:8080/api/nutrition-plan \
  -u alice:123456 \
  -H "Content-Type: application/json" \
  -d '{
    "days": [
      { "day": "MONDAY",    "meals": ["BREAKFAST", "LUNCH", "DINNER"] },
      { "day": "TUESDAY",   "meals": ["BREAKFAST", "LUNCH", "DINNER"] },
      { "day": "WEDNESDAY", "meals": ["LUNCH", "DINNER"] }
    ],
    "countryCode": "DE",
    "additionalInstructions": "Prefer quick recipes with less than 30 minutes prep time."
  }' | jq .
```
