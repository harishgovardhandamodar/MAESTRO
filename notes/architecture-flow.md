# Architecture Flow - Detailed System Architecture

## High-Level Architecture

MAESTRO follows a **three-tier architecture**: Client UI → Server Actions → AI Orchestration.

```mermaid
graph TB
    subgraph "TIER 1: Client (Next.js Client Components)"
        SF[SidebarInputForm<br/>Text Input + Presets]
        LC[LayerCards x7<br/>Per-layer results]
        MD[MermaidDiagram<br/>Visual architecture]
        PL[ProgressLog<br/>Console-style output]
        PDF[PDF Export<br/>jsPDF client-side]
    end

    subgraph "TIER 2: Server Actions (Next.js Server)"
        SA1[suggestThreat]
        SA2[recommendMitigation]
        SA3[getExecutiveSummary]
        SA4[getArchitectureDiagram]
        WR[withRetry Wrapper<br/>Exponential Backoff]
        EH[AIErrorHandler<br/>Error Classification]
    end

    subgraph "TIER 3: AI Orchestration (Genkit)"
        GEN[Genkit Core]
        T1[suggestThreatsForLayer<br/>Flow]
        T2[recommendMitigations<br/>Flow]
        T3[generateExecutiveSummary<br/>Flow]
        T4[generateArchitectureDiagram<br/>Flow]
    end
```

## Data Flow: Threat Analysis

```mermaid
sequenceDiagram
    participant U as User
    participant P as page.tsx
    participant SA as Server Actions
    participant GE as Genkit Flows
    participant LLM as LLM Provider

    U->>P: Click "Generate Analysis"
    P->>P: Reset state, start loop
    
    loop For each of 7 MAESTRO layers
        P->>P: Set status "analyzing"
        P->>SA: suggestThreat(arch, layer)
        SA->>SA: withRetry wrapper
        SA->>GE: suggestThreatsForLayer()
        GE->>LLM: Prompt + schema
        LLM-->>GE: Threat markdown
        GE-->>SA: Structured output
        SA-->>P: Threat result
        P->>P: Update layer state
        
        P->>SA: recommendMitigation(threat)
        SA->>SA: withRetry wrapper
        SA->>GE: recommendMitigations()
        GE->>LLM: Prompt + schema
        LLM-->>GE: Mitigation object
        GE-->>SA: Structured output
        SA-->>P: Mitigation result
        P->>P: Set status "complete"
    end
    
    P->>SA: getExecutiveSummary(arch, layers)
    SA->>GE: generateExecutiveSummary()
    GE->>LLM: Prompt + all results
    LLM-->>GE: Summary markdown
    GE-->>SA: Summary
    SA-->>P: Final summary
    P->>U: Analysis complete
```

## State Management

The main `page.tsx` component uses a set of React hooks to manage all application state:

```typescript
interface AppState {
    // Analysis state
    layers: LayerData[];           // 7 layer objects with status
    isAnalyzing: boolean;          // Global analyzing lock
    buttonText: string;           // Dynamic button label
    
    // Content state
    currentArchitecture: string;   // User's architecture text
    executiveSummary: string|null; // Final summary markdown
    mermaidCode: string;          // Generated diagram code
    
    // UI state
    logs: string[];              // Progress log entries
    isDownloading: boolean;       // PDF download lock
    isGeneratingDiagram: boolean; // Diagram generation lock
    
    // Refs (persistent across renders)
    analysisCancelledRef: MutableRefObject<boolean>;
    logsContainerRef: RefObject<HTMLDivElement>;
    diagramContainerRef: RefObject<HTMLDivElement>;
}
```

### State Update Flow

```mermaid
graph LR
    A[User Input] --> B(handleAnalyze)
    B --> C[Reset State]
    C --> D[Layer Loop]
    D --> E[Update Status]
    E --> F[Update Threat]
    F --> G[Update Mitigation]
    G --> H[Next Layer]
    H --> D
    D -->|done| I[Executive Summary]
    I --> J[Complete]
    
    K[Stop Button] --> L[Set Cancel Flag]
    L -->|next check| M[Break Loop]
```

## Server Action Architecture

Each server action in `actions.ts` follows this pattern:

```typescript
// Example: suggestThreat
export async function suggestThreat(
    architecturedescription: string,
    layerName: string,
    layerDescription: string
): Promise<SuggestThreatsForLayerOutput> {
    
    return withRetry(async () => {
        try {
            const result = await suggestThreatsForLayer({
                architecturedescription,
                layerName,
                layerDescription,
            });
            return result;
        } catch (error) {
            // Transform raw error → MaestroError
            const maestroError = AIErrorHandler.handleAIFlowError(
                error, 
                'suggestThreatsForLayer',
                { context: data }
            );
            throw maestroError;
        }
    }, undefined, (error) => {
        // Retry predicate
        if (error instanceof Error && error.message.includes('MaestroError')) {
            const maestroError = JSON.parse(...);
            return AIErrorHandler.shouldRetry(maestroError);
        }
        return true;
    });
}
```

## Configuration Environment Variables

```bash
# .env.example
LLM_PROVIDER=google    # or openai, ollama
LLM_MODEL=gemini-2.5-flash  # default model
OPENAI_API_KEY=        # if using OpenAI
OLLAMA_SERVER_ADDRESS=http://localhost:11434  # if using Ollama
```

Provider selection happens in `src/ai/genkit.ts`:

```typescript
switch (provider) {
    case 'openai':
        config.plugins = [openAI({apiKey: process.env.OPENAI_API_KEY})];
        config.model = `openai/${process.env.LLM_MODEL || 'gpt-4o-mini'}`;
        break;
    case 'ollama':
        config.plugins = [ollama({
            serverAddress: process.env.OLLAMA_SERVER_ADDRESS,
            models: [{name: process.env.LLM_MODEL, type: 'generate'}]
        })];
        config.model = `ollama/${process.env.LLM_MODEL}`;
        break;
    default: // google
        config.plugins = [googleAI()];
        config.model = `googleai/${process.env.LLM_MODEL || 'gemini-2.5-flash'}`;
}
```

## PDF Generation Pipeline

PDF generation happens entirely client-side using jsPDF:

```mermaid
graph TD
    A[Download Button] --> B[handleDownloadPdf]
    B --> C[Create jsPDF Document]
    C --> D[Add Header & Disclaimer]
    D --> E[Add Architecture Text]
    E --> F[Add Mermaid Diagram Image]
    F --> G[Add Executive Summary]
    G --> H[Loop: Add Each Layer]
    H --> I[Save PDF to Browser]
    
    F -->|convert SVG| J[Canvas Rendering]
    J -->|toDataURL| K[PNG Embedding]
    K --> F
```