# MAESTRO Threat Analyzer

AI-powered threat modeling tool based on the [CAI SA MAESTRO framework](https://cloudsecurityalliance.org/blog/2025/02/06/agentic-ai-threat-modeling-framework-maestro), designed to analyze multi-agent AI systems across 7 security layers and generate actionable threat assessments with mitigation recommendations.

## Why MAESTRO

Traditional threat modeling frameworks struggle with agentic AI systems that exhibit:
- **Autonomy** — agents make independent decisions
- **Non-determinism** — LLM outputs vary between invocations
- **Complex interactions** — agent-to-agent communication creates emergent attack surfaces
- **Dynamic tool use** — runtime function calling and external API integrations

The MAESTRO methodology provides a structured, seven-layer approach to systematically address these novel threats.

## MAESTRO Framework Layers

| # | Layer | What It Covers |
|---|-------|---------------|
| 1 | **Foundation Models** | LLM security, prompt injection, model stealing, backdoors |
| 2 | **Data Operations** | Training data poisoning, PII leakage, RAG integrity |
| 3 | **Agent Frameworks** | Tool-use abuse, memory corruption, orchestration hijacking |
| 4 | **Deployment & Infrastructure** | Container escapes, supply chain attacks, configuration drift |
| 5 | **Evaluation & Observability** | Model drift, evaluation gaming, monitoring blind spots |
| 6 | **Security & Compliance** | Access control, audit trails, regulatory alignment |
| 7 | **Agent Ecosystem** | Inter-agent attacks, marketplace poisoning, reputation manipulation |

## Key Features

- **Layer-by-layer AI analysis** — each MAESTRO layer analyzed sequentially with dedicated LLM prompts
- **Mitigation recommendations** — actionable remediation with reasoning and caveats
- **PDF report generation** — complete analysis exported as a shareable document
- **Architecture diagram** — AI-generated Mermaid visualization of the system under analysis
- **Executive summary** — synthesized cross-cutting findings across all layers
- **Use-case presets** — pre-built architecture descriptions for common multi-agent patterns
- **Real-time progress logging** — terminal-style output showing analysis progression

## Quick Start

### Prerequisites

- Node.js 18+ with npm
- API key for LLM provider (Google Gemini, OpenAI, or Ollama)

### Setup

```bash
git clone <repo> && cd MAESTRO
npm install
```

Copy `.env.example` to `.env` and configure:

| Variable | Required For | Example |
|----------|-------------|---------|
| `LLM_PROVIDER` | All | `google`, `openai`, `ollama` |
| `GEMINI_API_KEY` | Google Gemini | `AIzaSy...` |
| `OPENAI_API_KEY` | OpenAI | `sk-...` |
| `OLLAMA_SERVER_ADDRESS` | Ollama | `http://localhost:11434` |
| `LLM_MODEL` | Optional override | `gemini-2.5-flash` |

### Running

Start two processes in separate terminals:

```bash
# Terminal 1: Next.js frontend
npm run dev

# Terminal 2: Genkit AI flows backend
npm run genkit:dev
```

Visit **http://localhost:9002** to use the application.

### Development Workflow

```bash
npm run build          # Production build
npm run start          # Production server
npm run lint           # ESLint checks
npm run typecheck      # TypeScript type checking
npm run test:run       # Run all tests once
npm run test:coverage  # Run tests with coverage report
```

## Technology Stack

| Category | Technology |
|----------|-----------|
| Framework | Next.js 15 (App Router, Turbopack) |
| Language | TypeScript (strict mode) |
| AI | Genkit (multi-provider LLM orchestration) |
| UI | shadcn/ui + Tailwind CSS 4 |
| Forms | react-hook-form + Zod validation |
| PDF | jsPDF (client-side generation) |
| Diagrams | Mermaid.js |
| Testing | Vitest + React Testing Library |
| LLM Providers | Google Gemini, OpenAI, Ollama |

## Architecture At A Glance

```
┌─────────────────────────────────────────────────────────────────┐
│                        BROWSER                                  │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │ SidebarInput │  │ LayerCard x7 │  │ Analysis Log +        │  │
│  │ Form         │  │ (threat +    │  │ Architecture Diagram  │  │
│  │              │  │  mitigation) │  │ (Mermaid.js)          │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬────────────┘  │
│         │                 │                      │              │
│  ┌──────▼─────────────────▼──────────────────────▼────────────┐  │
│  │                  Home Component (page.tsx)                   │  │
│  │              State orchestration + PDF export                │  │
│  └───────────────────────────┬─────────────────────────────────┘  │
└──────────────────────────────┼───────────────────────────────────┘
                               │
                    ┌──────────▼────────────┐
                    │   Server Actions      │
                    │   (actions.ts)        │
                    │   withRetry wrapper   │
                    └──────────┬────────────┘
                               │
                    ┌──────────▼────────────┐
                    │   Genkit AI Flows     │
                    │   (flows/)            │
                    │   4 independent flows │
                    └──────────┬────────────┘
                               │
                    ┌──────────▼────────────┐
                    │    LLM Provider       │
                    │  (Gemini/OpenAI/      │
                    │   Ollama)             │
                    └───────────────────────┘
```

## Project Structure

```
src/
├── ai/                      # Genkit AI configuration + flows
│   ├── genkit.ts            # Provider setup
│   └── flows/               # AI workflow definitions
├── app/                     # Next.js App Router pages
│   ├── actions.ts           # Server actions (AI gateways)
│   └── page.tsx             # Main application page
├── components/              # React components
│   ├── ui/                  # shadcn/ui components
│   ├── layer-card.tsx       # Per-layer visualization
│   ├── sidebar-input-form.tsx  # Architecture input form
│   └── mermaid-diagram.tsx  # Diagram renderer
├── data/                    # Static data
│   └── maestro.ts           # MAESTRO layer definitions
├── lib/                     # Utilities
│   ├── types.ts             # Shared TypeScript types
│   ├── ai-error-handler.ts  # Custom error handling
│   ├── retry-utils.ts       # Retry logic
│   └── errors.ts            # Error classes + enums
└── test/                    # Test configuration
    └── setup.ts             # Vitest global mocks
```

## License

Proprietary — DistributedApps.ai
