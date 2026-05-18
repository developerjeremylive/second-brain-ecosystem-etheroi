# Arquitectura del Ecosistema — Second Brain

## La Historia

Necesitaba resolver un problema concreto: **el manejo de la información dentro de mi PC**.

Comencé creando una biblioteca virtual para todo lo que me llamaba la atención — conocimiento técnico, libros, podcasts, resúmenes de videos de YouTube. Quería armar mi propia biblioteca de conocimiento, curada y organizada.

Todo empezó con el [gist de Karpathy sobre LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f). Se me ocurrió hacer un segundo cerebro. Al principio fueron dos carpetas con información en archivos `.md` escritos a mano. Funcionaba, pero eventualmente manejarlos se volvía complicado: links rotos, notas duplicadas, contenido desactualizado, ideas huérfanas.

Así que decidí **automatizar mi Second Brain**.

Hoy el ecosistema tiene estos componentes:

- **Obsidian** — El vault. Donde vive todo el conocimiento. Es la interfaz humana.
- **Librarian** — Un pipeline review-driven de mantenimiento de conocimiento para el vault de Obsidian. Lee fuentes ubicadas explícitamente en `raw/`, clasifica, detecta duplicados, genera propuestas y aplica cambios aprobados al wiki. Todas las mutaciones requieren aprobación humana.
- **Content Toolkit** — Herramientas de ingesta. Transcripción de videos/audio con Whisper, pipeline completo de YouTube → audio → transcripción → resumen inteligente. Produce artefactos que puedo guardar explícitamente en `raw/` cuando quiero que Librarian los procese.
- **Researcher** — Un agente de búsqueda web (patrón Search-o1: piensa → busca → lee → sintetiza). Se encarga de encontrar información que le falta a la biblioteca y complementarla. No depende de Librarian — se invoca directamente, y su output puede guardarse en `raw/` cuando sea útil.

Está en alpha. Puede tener bugs. Pero ya es útil.

---

## Diagrama de Arquitectura

```mermaid
graph TB
    subgraph USER["👤 Usuario"]
        U1["Captura ideas, notas, links"]
        U2["Chat con Librarian<br/>via TUI / CLI"]
        U3["Aprueba / Rechaza<br/>propuestas"]
        U5["Consentimiento explícito<br/>mover/copiar a raw/"]
        U4["Lee y navega<br/>el wiki en Obsidian"]
    end

    subgraph SOURCES["📥 Fuentes Externas"]
        YT["📺 YouTube"]
        POD["🎙️ Podcasts"]
        WEB["🌐 Web / Papers"]
        PDF["📄 PDFs / Artículos"]
        MANUAL["📝 Notas manuales"]
    end

    subgraph TOOLKIT["🔧 Content Toolkit"]
        INGEST["ingest-youtube<br/>yt-dlp → ffmpeg → Whisper"]
        TRANSCRIBE["transcribe<br/>Whisper standalone"]
        ARTIFACT["transcripciones / resúmenes<br/>en staging para revisar"]
    end

    subgraph VAULT["🗄️ Obsidian Vault"]
        INBOX["inbox/<br/>Captura humana<br/>(Librarian no lo lee)"]
        RAW["raw/<br/>Fuentes inmutables<br/>aprobadas para IA"]
        WIKI["wiki/<br/>Conocimiento curado<br/>conceptos · entidades<br/>sources · synthesis"]
        REVIEWS["reviews/<br/>Superficie humana<br/>de revisión"]
        PROPOSALS[".librarian/proposals/<br/>Fuente interna<br/>de verdad"]
        LEDGER[".librarian/processed.json<br/>Ledger de fuentes procesadas"]
    end

    subgraph LIBRARIAN_AGENT["📚 Librarian (Agente Curador)"]
        L1["Inspect raw/"]
        L2["Classify & Dedup"]
        L3["Generate Proposals"]
        L4["Apply<br/>(con aprobación humana)"]
        L1 --> L2 --> L3
    end

    subgraph RESEARCHER_AGENT["🔍 Researcher (Agente de Búsqueda)"]
        R1["THINK"]
        R2["SEARCH<br/>Brave / SearXNG / Tavily"]
        R3["READ & Synthesize"]
        R1 --> R2 --> R3
    end

    %% Fuentes → Content Toolkit → artefactos en staging
    YT --> INGEST
    POD --> TRANSCRIBE
    INGEST -->|"transcripción + resumen"| ARTIFACT
    TRANSCRIBE -->|"transcripción"| ARTIFACT
    ARTIFACT -->|"el usuario revisa output"| U5

    %% Fuentes manuales → inbox/ o raw/ con consentimiento explícito
    WEB -->|"captura manual"| INBOX
    PDF -->|"captura manual"| INBOX
    MANUAL --> INBOX
    U1 --> INBOX
    U1 -->|"fuente aprobada directa"| U5
    INBOX -->|"promover intencionalmente"| U5
    U5 --> RAW

    %% raw/ → Librarian → propuestas → aprobación → apply → wiki/
    RAW --> L1
    L3 -->|"guarda propuesta"| PROPOSALS
    L3 -->|"exporta revisión legible"| REVIEWS
    REVIEWS -.->|"propuestas pendientes"| U3
    PROPOSALS -->|"propuesta aprobada"| L4
    L4 -->|"escribe notas curadas"| WIKI
    L4 -->|"marca fuente procesada<br/>después de apply exitoso"| LEDGER

    %% Usuario interactúa con Librarian
    U2 --> L1
    U3 -->|"aprobar / rechazar vía CLI"| PROPOSALS
    U3 -->|"aplicar propuesta aprobada"| L4

    %% Researcher → puede alimentar raw/ solo con consentimiento del usuario
    R3 -->|"respuesta + sources"| U5

    %% Usuario lee wiki
    WIKI --> U4

    %% Researcher se invoca directamente
    U2 -.->|"o"| R1

    %% Styling
    classDef user fill:#f9e79f,stroke:#f4d03f,color:#000
    classDef source fill:#aed6f1,stroke:#5dade2,color:#000
    classDef toolkit fill:#d5f5e3,stroke:#58d68d,color:#000
    classDef vault fill:#fadbd8,stroke:#ec7063,color:#000
    classDef agent fill:#e8daef,stroke:#af7ac5,color:#000
    classDef research fill:#fdebd0,stroke:#f0b27a,color:#000

    class U1,U2,U3,U4,U5 user
    class YT,POD,WEB,PDF,MANUAL source
    class INGEST,TRANSCRIBE,ARTIFACT toolkit
    class RAW,WIKI,INBOX,REVIEWS,PROPOSALS,LEDGER vault
    class L1,L2,L3,L4 agent
    class R1,R2,R3 research
```

---

## Flujo de Comunicación

### 1. Usuario ↔ Obsidian

El usuario interactúa directamente con Obsidian como interfaz principal. Escribe notas, captura ideas, revisa páginas generadas del wiki y navega el grafo de conocimiento. Todo es Markdown plano.

```
Usuario → Obsidian (vault/)
Usuario ← Obsidian (leer wiki/, buscar, navegar)
```

### 2. Usuario ↔ Librarian

El usuario se comunica con Librarian a través de una TUI (terminal) o CLI. Le puede pedir que ingiera fuentes aprobadas, busque en el wiki, o haga mantenimiento. Librarian genera propuestas antes de cualquier mutación.

`reviews/` es una superficie humana de revisión y export. `.librarian/proposals/` es la fuente de verdad interna de propuestas.

```
Usuario → Librarian TUI/CLI → "ingerí fuentes aprobadas", "buscá X en wiki"
Librarian → .librarian/proposals/ → guarda propuesta
Librarian → reviews/ → exporta propuesta legible
Usuario → Librarian → "aprobá propuesta #42"
Usuario → Librarian → "aplicá propuesta #42"
Librarian → wiki/ → escribe nota curada
Librarian → .librarian/processed.json → marca fuente procesada
```

### 3. Content Toolkit → Usuario → Vault (raw/)

Content Toolkit es un pre-procesador. Transforma medios (video, audio) en texto, pero no decide qué puede procesar Librarian. El usuario revisa el output y lo guarda o mueve explícitamente a `raw/` cuando debe entrar a la capa procesable por IA.

```
YouTube URL → ingest-youtube → transcripción + resumen → revisión humana → raw/
Video/Audio → transcribe → transcripción → revisión humana → raw/
```

### 4. Researcher (independiente)

Researcher es un agente de búsqueda web. No depende de Librarian — se invoca directamente. Busca en la web, lee páginas y sintetiza respuestas. Su output puede copiarse a `raw/` por el usuario para que Librarian lo procese después.

```
Usuario → researcher "qué es agentic RAG?" → respuesta + sources
Usuario revisa resultado → raw/ → (opcional) Librarian lo procesa
```

---

## Componentes del Ecosistema

| Componente | Rol | Repo |
|------------|-----|------|
| **Obsidian** | Interfaz humana, vault de conocimiento | Vault local |
| **Librarian** | Pipeline review-driven y proposal-based de mantenimiento de conocimiento | [`librarian`](../librarian/) |
| **Content Toolkit** | Ingesta de medios (YouTube → texto, transcripción) con entrada al vault aprobada por el usuario | [`content-toolkit`](../content-toolkit/) |
| **Researcher** | Búsqueda web agéntica (Search-o1) | [`researcher`](../researcher/) |

---

## Decisiones Clave

- **Todo lo que Librarian procesa entra por `raw/` primero** — `raw/` es la frontera explícita de consentimiento para procesamiento con IA.
- **Librarian no busca en la web** — Su scope es gestor de Obsidian, no Google.
- **Researcher es repo separado** — Responsabilidad única, reutilizable.
- **Content Toolkit es pre-procesador** — Transforma medios antes de que el usuario decida si el resultado pertenece en `raw/`.
- **Propuestas antes de apply** — Nunca se escribe directo a `wiki/` sin aprobación humana y un paso explícito de apply.
- **Reviews y proposals son superficies separadas** — `reviews/` es para revisión humana; `.librarian/proposals/` es la fuente interna de verdad.
- **Wikilinks > tags** — El grafo de conexiones es más valioso que categorías.
