# MAESTRO Threat Analyzer — Data Flow & Architecture

High-level architecture showing how data flows through the MAESTRO Threat Analyzer system, from user input through AI processing to visual output.

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              BROWSER / CLIENT                               │
│                                                                             │
│  ┌───────────────────────┐     ┌───────────────────────────┐                │
│  │   SidebarInputForm    │     │         page.tsx           │                │
│  │  (react-hook-form)    │     │   (State orchestration)    │                │
│  │  • Preset selector    │────▶│   • layers[] state         │                │
│  │  • Textarea (50-5K    │     │   • logs[] state           │                │
│  │    chars)             │     │   • mermaidCode state      │                │
│  │  • Zod validation     │     │   • executiveSummary state │                │
│  └───────────────────────┘     │   • refs & flags           │                │
│                                └─────┬───────────────┬──────┘                │
│                                      │               │                       │
│                    ┌─────────────────▼┐  ┌───────────▼──────────────┐        │
│                    │    LayerCard x7  │  │   Progress Log Panel     │        │
│                    │  (Accordion UI)  │  │   (ScrollArea + logs)    │        │
│                    │  • Collapse/     │  │                        │        │
│                    │   expand        │  │   ┌──────────────────┐  │        │
│                    │  • Threat view  │  │   │Architect Diagram │  │        │
│                    │  • Mitigation   │  │   │ Card             │  │        │
│                    │   view          │  │   │(MermaidDiagram)  │  │        │
│                    └─────────────────┘  │   └──────────────────┘  │        │
│                                         └─────────────────────────┘        │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────┐      │
│  │                     PDF Export (jsPDF)                            │      │
│  │  • Header + disclaimer                                           │      │
│  │  • Architecture description                                      │      │
│  │  • SVG diagram (canvas rasterization)                            │      │
│  │  • Executive summary                                             │      │
│  │  • Per-layer: threat + mitigation (recommendation/reasoning/     │      │
│  │    caveats)                                                      │      │
│  └───────────────────────────────────────────────────────────────────┘      │
│                                                                             │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │  (Server Action calls)
┌─────────────────────────────────────────────────────────────────────────────┐
│                         NEXT.JS SERVER                                     │
│                                                                             │
│  ┌───────────────────────────────────────────┐                             │
│  │          Server Actions (actions.ts)      │                             │
│  │                                           │                             │
│  │  suggestThreat(arch, name, desc) ──────┐  │                             │
│  │  recommendMitigation(threat, name) ───┐ │ │                             │
│  │  getExecutiveSummary(arch, layers) ──┐ │ │ │                             │
│  │  getArchitectureDiagram(arch) ──────┐ │ │ │ │                             │
│  │                                     │ │ │ │ │                             │
│  │  ┌───────────────────────────────┐ │ │ │ │ │                             │
│  │  │    withRetry(errorPredicate)  │ │ │ │ │ │                             │
│  │  │  • retries on transient errors│ │ │ │ │ │                             │
│  │  │  • respects MaestroError      │ │ │ │ │ │                             │
│  │  └───────────────────────────────┘ │ │ │ │ │                             │
│  └─────────┬───────┬───────┬─────────┘ │ │ │ │ │                             │
│            │       │       │           ▼ │ │ │ │ │                             │
│            │       │       │      ┌──────┘ │ │ │ │ │                             │
│            │       │       │      │        │ │ │ │ │ │                             │
│            ▼       ▼       ▼      ▼        ▼ │ │ │ │ │                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                   Genkit AI Flows (ai/flows/)                           │  │
│  │                                                                         │  │
│  │  suggestThreatsForLayer()          • Gemini/OpenAI/Ollama              │  │
│  │  recommendMitigations()            • Structured output (zod)           │  │
│  │  generateExecutiveSummary()        • Context window management         │  │
│  │  generateArchitectureDiagram()     • Mermaid syntax output             │  │
│  └─────────────────────────┬─────────────────────────────────────────────┘  │
└────────────────────────────┼────────────────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  LLM PROVIDER   │
                    │  (Gemini 2.5    │
                    │   Flash / GPT   │
                    │   / Ollama)     │
                    └─────────────────┘
```

## Data Flow: Analysis Pipeline

### 1. Input Collection

```
User Input
  │
  ├─→ preset: PresetKey | null
  │     ├── "multi-agent-research-platform"
  │     ├── "autonomous-testing-agent"
  │     ├── "multi-modal-content-creation-pipeline"
  │     ├── "ai-powered-coding-assistant"
  │     └── "customer-support-agent-network"
  │
  └─→ description: string (50–5000 chars)
      (if no preset, textarea required)
```

The `SidebarInputForm` component manages form state with `react-hook-form` + Zod validation. On submit, it calls the `onAnalyze` callback passed from `page.tsx`.

### 2. Analysis Loop (Client-Side Orchestration)

The main application component (`page.tsx`) orchestrates the sequential analysis:

```
handleAnalyze(description)
  │
  ├─→ Reset state
  │   • layers = INITIAL_LAYERS (all "pending")
  │   • logs = []
  │   • executiveSummary = null
  │   • analysisCancelledRef = false
  │
  ├─→ FOR EACH (7 MAESTRO layers):
  │   │
  │   ├─→ updateLayerStatus(layer.id, "analyzing")
  │   │
  │   ├─→ suggestThreat(description, layer.name, layer.description)
  │   │   └─→ Server Action → Genkit Flow → LLM
  │   │       └─→ Returns: { threatAnalysis: Markdown }
  │   │
  │   ├─→ Check cancellation flag
  │   │
  │   ├─→ recommendMitigation(threat, layer.name)
  │   │   └─→ Server Action → Genkit Flow → LLM
  │   │       └─→ Returns: {
  │   │             recommendation: string,
  │   │             reasoning: string,
  │   │             caveats: string
  │   │           }
  │   │
  │   └─→ updateLayerStatus(layer.id, "complete")
  │
  ├─→ getExecutiveSummary(description, completedLayers)
  │   └─→ Server Action → Genkit Flow → LLM
  │       └─→ Returns: { summary: Markdown }
  │
  └─→ Set UI state & update logs
```

**Key pattern:** Server actions are called directly from the client-side component. Each server action wraps the Genkit flow call with `withRetry` for resilience.

### 3. Architecture Diagram (Independent Flow)

```
handleGenerateDiagram()
  │
  ├─→ getArchitectureDiagram(currentArchitecture)
  │   └─→ Server Action → Genkit Flow → LLM
  │       └─→ Returns: { mermaidCode: string }
  │
  └─→ setMermaidCode(mermaidCode)
      └─→ MermaidDiagram component renders SVG
          └─→ mermaid.render(id, code) → <svg>
```

The diagram is generated **independently** from the main analysis pipeline and persists across multiple analyses.

### 4. PDF Export (Client-Side)

```
handleDownloadPdf()
  │
  ├─→ new jsPDF()
  │
  ├─→ Add: Title, disclaimer, developer attribution
  │
  ├─→ Add: Architecture description
  │
  ├─→ Add: Architecture diagram
  │   └─→ querySelector('svg') → XMLSerializer → canvas.toDataURL()
  │
  ├─→ Add: Executive summary (Markdown → plain text)
  │   └─→ strip ###, ##, #, ** characters
  │
  ├─→ FOR EACH layer:
  │   ├─→ Layer name (h2)
  │   ├─→ Threat description (body)
  │   └─→ Mitigation: recommendation, reasoning, caveats
  │
  └─→ doc.save("MAESTRO_Threat_Analysis.pdf")
```

## Data Types

### Core Type: `LayerData`

```typescript
interface LayerData {
  id: string;           // e.g., "foundation-models"
  name: string;         // e.g., "Foundation Models"
  description: string;  // Layer context for LLM prompt
  icon: string;         // SVG path string
  threat: string | null;
  mitigation: {
    recommendation: string;
    reasoning: string;
    caveats: string;
  } | null;
  status: "pending" | "analyzing" | "complete" | "error";
}
```

### State Shape in `page.tsx`

```typescript
const [layers, setLayers] = useState<LayerData[]>(INITIAL_LAYERS);
const [isAnalyzing, setIsAnalyzing] = useState(false);
const [isDownloading, setIsDownloading] = useState(false);
const [buttonText, setButtonText] = useState("Generate Analysis");
const [logs, setLogs] = useState<string[]>([]);
const [currentArchitecture, setCurrentArchitecture] = useState("");
const [executiveSummary, setExecutiveSummary] = useState<string | null>(null);
const [mermaidCode, setMermaidCode] = useState<string>("");
const [isGeneratingDiagram, setIsGeneratingDiagram] = useState(false);

// Ref for cancellation signal
const analysisCancelledRef = useRef(false);
// Refs for DOM access
const logsContainerRef = useRef<HTMLDivElement>(null);
const diagramContainerRef = useRef<HTMLDivElement>(null);
```

## Provider Routing

The Genkit SDK routes to the configured LLM provider:

```
LLM_PROVIDER env var
  │
  ├─→ "google"   → GeminiModel ("gemini-2.5-flash")
  ├─→ "openai"   → ChatOpenAI ("gpt-4o")
  │                 (requires OPENAI_API_KEY + OPENAI_ORG_ID)
  ├─→ "ollama"   → Ollama (model: "llama3.3")
  │                 (requires OLLAMA_SERVER_ADDRESS)
  └─→ (default)  → GeminiModel
```

Each AI flow defines a `model` property and uses `genkit.core.use` for context tracking. The provider is abstracted so the same prompt works across any configured backend.
