# Agent Architecture

This document describes how the AI agents in Portfolio Pilot collaborate to turn a user's raw portfolio data into a full financial analysis.

## Agent Collaboration Overview

```mermaid
graph TB
    User[User Request] -->|Portfolio Analysis| Planner[Financial Planner<br/>Orchestrator Agent]

    Planner -->|Check Instruments| Tagger[InstrumentTagger<br/>Agent]
    Tagger -->|Classify Assets| DB[(Database)]

    Planner -->|Generate Analysis| Reporter[Report Writer<br/>Agent]
    Reporter -->|Markdown Reports| DB

    Planner -->|Create Visualizations| Charter[Chart Maker<br/>Agent]
    Charter -->|JSON Chart Data| DB

    Planner -->|Project Future| Retirement[Retirement Specialist<br/>Agent]
    Retirement -->|Income Projections| DB

    DB -->|Results| Response[Complete Analysis<br/>Report]

    Planner -->|Retrieve Context| Vectors[(S3 Vectors<br/>Knowledge Base)]

    Schedule[EventBridge<br/>Every 2 Hours] -->|Trigger| Researcher[Researcher<br/>Agent]
    Researcher -->|Store Insights| Vectors

    style Planner fill:#FFD700,stroke:#333,stroke-width:3px
    style Researcher fill:#87CEEB
    style Schedule fill:#9333EA
    style Tagger fill:#98FB98
    style Reporter fill:#DDA0DD
    style Charter fill:#F0E68C
    style Retirement fill:#FFB6C1
```

## Agent Responsibilities

### Financial Planner (Orchestrator)

**Role**: coordinates the entire analysis workflow
- Receives analysis requests over SQS
- Identifies missing instrument data and delegates to InstrumentTagger
- Fans out work to the specialist agents in parallel
- Retrieves relevant context from the S3 Vectors knowledge base
- Compiles the final result and updates job status throughout

### InstrumentTagger

**Role**: automatically classifies financial instruments
- Determines asset class (equity, fixed income, etc.), region, and sector exposure
- Uses structured outputs for a consistent, parseable format
- Runs only when the database is missing data for a given symbol

### Researcher (independent agent)

**Role**: autonomously gathers market intelligence
- Runs on an EventBridge schedule (every 2 hours), independent of user requests
- Browses financial sources for market trends, earnings, and economic indicators
- Writes findings into the S3 Vectors knowledge base
- That knowledge is later retrieved by the Financial Planner for context

### Report Writer

**Role**: generates the narrative portfolio analysis
- Analyzes composition, diversification, and risk exposure
- Writes an executive summary and detailed markdown report
- Pulls in supporting context from the knowledge base via a tool call

### Chart Maker

**Role**: turns portfolio data into visualizations
- Produces allocation breakdowns by asset class, region, and sector
- Outputs Recharts-compatible JSON (pie, bar, donut charts)
- Returns structured output directly — no tool calls

### Retirement Specialist

**Role**: projects long-term financial outcomes
- Runs Monte Carlo simulations against the user's goals
- Factors in years to retirement and target income
- Produces a success-probability projection plus recommendations

## Agent Communication Flow

```mermaid
sequenceDiagram
    participant S as EventBridge Schedule
    participant Re as Researcher
    participant V as S3 Vectors
    participant U as User
    participant P as Financial Planner
    participant T as InstrumentTagger
    participant Rw as Report Writer
    participant C as Chart Maker
    participant Rt as Retirement Specialist
    participant DB as Database

    Note over S,Re: Independent Research Flow (Every 2 Hours)
    S->>Re: Trigger scheduled research
    Re->>V: Store market insights

    Note over U,DB: User-Triggered Analysis Flow
    U->>P: Request Portfolio Analysis
    P->>DB: Check for missing instrument data

    alt Missing Instrument Data
        P->>T: Tag unknown instruments
        T->>DB: Store classifications
    end

    P->>V: Retrieve relevant research
    Note right of P: Uses research previously<br/>stored by Researcher

    par Parallel Analysis
        P->>Rw: Generate portfolio report
        Rw->>DB: Save analysis
    and
        P->>C: Create visualizations
        C->>DB: Save chart data
    and
        P->>Rt: Calculate projections
        Rt->>DB: Save retirement analysis
    end

    P->>DB: Compile all results
    P->>U: Return complete analysis
```

## Agent Capabilities Matrix

| Agent | Model | Primary Function | Output Format |
|-------|-------|-------------------|----------------|
| Financial Planner | Amazon Nova Pro | Orchestration & coordination | Job status + compiled result |
| InstrumentTagger | Amazon Nova Pro | Asset classification | Structured JSON |
| Researcher | Bedrock open-weight model | Market intelligence | Markdown articles |
| Report Writer | Amazon Nova Pro | Portfolio narrative | Markdown report |
| Chart Maker | Amazon Nova Pro | Data visualization | Recharts JSON |
| Retirement Specialist | Amazon Nova Pro | Future projections | Analysis + chart data |

## Knowledge Integration

Agents draw on two sources of context:

**S3 Vectors knowledge base** — historical research, earnings analysis, economic indicators, and investment commentary, continuously refreshed by the Researcher.

**Database reference data** — instrument classifications, user portfolios, and prior reports, all validated through Pydantic schemas before being written.

## Agent Collaboration Patterns

1. **Data enrichment**: `Unknown Instrument → InstrumentTagger → Enriched Data → Other Agents`
2. **Independent research**: `EventBridge (every 2 hrs) → Researcher → S3 Vectors → Knowledge Base Growth`
3. **Knowledge integration**: `Financial Planner → Retrieve from S3 Vectors → Contextual Analysis`
4. **Parallel processing**: `Financial Planner → [Report Writer, Chart Maker, Retirement] → Compiled Results`

## Key Design Principles

1. **Specialization** — each agent has one focused responsibility
2. **Orchestration, not micromanagement** — the planner coordinates but leaves execution to the specialists
3. **Parallel execution** — independent agents run simultaneously for speed
4. **Shared knowledge** — the S3 Vectors store gives every agent access to accumulated research
5. **Graceful degradation** — the system keeps working even if one agent's output is missing
6. **Tools XOR structured outputs** — a current LiteLLM/Bedrock limitation means each agent uses either tool calling or structured outputs, never both; see the implementation note in each agent's `agent.py`

## Possible Future Enhancements

- Additional specialist agents (tax optimization, rebalancing, deeper risk analysis)
- Real-time market data integration
- International market coverage and ESG scoring
