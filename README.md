# Restrictive Meal Planner — Agentic RAG (Langflow)

A retrieval-augmented meal planning assistant built in [Langflow](https://www.langflow.org/). Given a simple request like *"Prepare a meal plan"*, it generates a full 7-day plan that (a) mirrors the style and structure of previously approved meal plans and (b) respects a separate, authoritative list of dietary restrictions.

The defining design choice is **two independent vector indexes**, queried in parallel and fused at prompt-assembly time:

| Index | Collection | Source | Role |
|-------|-----------|--------|------|
| **Meal Plans** | `MealPlan` | Sample approved meal plan PDFs | Few-shot grounding — *what a good plan looks like* |
| **Restrictions** | `Restrictions` | `Restrictions.pdf` | Hard constraints — *what is never allowed* |

Keeping restrictions in their own index means the constraint signal never gets diluted by the (much larger) corpus of sample plans during retrieval. The model receives them as a distinct, dedicated block.

---

## Architecture

The project is two Langflow flows: an **offline ingestion flow** that builds the indexes, and an **online query flow** that answers requests.

```
INGESTION (run once / on data change)
┌─────────────────────────────────────────────────────────────┐
│  Sample Meal Plans (PDFs) ─► Split Text ─► Embed ─► Chroma    │
│                              (1000/200)            [MealPlan] │
│                                                              │
│  Restrictions.pdf ────────► Split Text ─► Embed ─► Chroma    │
│                              (1000/200)         [Restrictions]│
└─────────────────────────────────────────────────────────────┘

QUERY (per request)
┌─────────────────────────────────────────────────────────────┐
│  Chat Input ──┬─► Chroma [MealPlan]     ─► Parser ─► context  │
│               │                                              │
│               ├─► Chroma [Restrictions] ─► Parser ─► restr.   │
│               │                                              │
│               └────────────────────────────────► question    │
│                                                              │
│  context + question + restrictions ─► Prompt ─► LLM ─► Output │
│                                                 (Claude)      │
└─────────────────────────────────────────────────────────────┘
```

---

## Ingestion Flow

Two parallel pipelines write into two Chroma collections that share a persist directory but stay logically separate.

### Pipeline A — Meal Plans index
1. **File** (`Local` storage) — loads the sample meal plan PDFs.
2. **Split Text** — chunk size `1000`, chunk overlap `200`.
3. **Embedding Model** — `text-embedding-3-small`.
4. **Chroma DB** — collection `MealPlan`, ingests chunks + embeddings.

### Pipeline B — Restrictions index
1. **Read File** — loads `Restrictions.pdf`.
2. **Split Text** — chunk size `1000`, chunk overlap `200`.
3. **Embedding Model** — `text-embedding-3-small`.
4. **Chroma DB** — collection `Restrictions`, ingests chunks + embeddings.

> Both collections persist to a local directory (configured per node as `/Users/<you>/Desktop/Rec...`). Update this path for your machine before running.

---

## Query Flow

1. **Chat Input** — the user request (e.g. *"Prepare a meal plan"*). This single message is fanned out to three downstream destinations.
2. **Embedding Model** (`text-embedding-3-small`) — embeds the query so it can be matched against both stores. Use the **same** embedding model here as in ingestion, or retrieval quality collapses.
3. **Chroma DB [MealPlan]** — semantic search over approved plans → **Parser** (`Table → Parsed Text`) → feeds the prompt's `context` variable.
4. **Chroma DB [Restrictions]** — semantic search over restricted items → **Parser** (`Table → Parsed Text`) → feeds the prompt's `restrictions` variable.
5. **Prompt** — a template with three dynamic variables (`context`, `question`, `restrictions`) that assembles the final instruction. The template asks for a complete weekly plan covering Monday through Sunday.
6. **Language Model** — `claude-opus-4-6`, generates the plan.
7. **Chat Output** — displays the result in the Playground.

### Prompt template (variables)
| Variable | Filled from | Purpose |
|----------|-------------|---------|
| `context` | MealPlan retrieval | Examples of well-formed, approved plans to imitate |
| `question` | Chat Input | The user's actual request |
| `restrictions` | Restrictions retrieval | Hard dietary constraints the plan must obey |

---

## Why two indexes (the "agentic RAG" part)

A naive RAG setup dumps every document into one collection and hopes the top-k results happen to include the relevant restrictions. That's unreliable: when the corpus is dominated by meal plans, a query like *"prepare a meal plan"* retrieves mostly plan chunks and may miss the constraint document entirely.

Splitting the knowledge into **two purpose-built retrievers** and routing each into its own prompt slot gives you:

- **Guaranteed constraint visibility** — restrictions are always retrieved and always present in the prompt, never crowded out.
- **Separation of concerns** — "style/structure" knowledge and "hard rules" knowledge are tuned, updated, and re-indexed independently.
- **Cleaner prompting** — the model sees labeled blocks (examples vs. rules) rather than an undifferentiated context blob, which improves adherence.

---

## Prerequisites

- **Langflow** installed and running
- **Python** compatible with your Langflow version
- **OpenAI API key** — for `text-embedding-3-small`
- **Anthropic API key** — for the `claude-opus-4-6` Language Model node
- **Chroma** (bundled with the Langflow Chroma DB component; persists locally)

---

## Setup

1. **Clone / import the flows** into Langflow (the ingestion flow and the query flow).
2. **Set API keys** in the Embedding Model and Language Model nodes (or as environment variables).
3. **Fix the persist directory** on every Chroma DB node to a valid local path on your machine.
4. **Add your source documents:**
   - Approved meal plan PDFs → the `File` node (MealPlan pipeline)
   - Your restrictions document → the `Read File` node (`Restrictions.pdf`)
5. **Run the ingestion flow once** to populate both `MealPlan` and `Restrictions` collections.
6. **Open the query flow** in the Playground and ask for a plan.

---

## Usage

In the Playground, send a request such as:

```
Prepare a meal plan
```

The assistant will retrieve approved-plan examples and active restrictions, then return a 7-day plan (Monday–Sunday) that follows the examples' structure while honoring every restriction.

---

## Configuration reference

| Setting | Value | Where |
|---------|-------|-------|
| Chunk size | `1000` | Split Text (both pipelines) |
| Chunk overlap | `200` | Split Text (both pipelines) |
| Embedding model | `text-embedding-3-small` | Embedding Model (ingestion + query) |
| LLM | `claude-opus-4-6` | Language Model |
| Vector store | Chroma (local, persisted) | Chroma DB |
| Collections | `MealPlan`, `Restrictions` | Chroma DB |

---

## Tips & gotchas

- **Embedding model must match.** Query-side and ingestion-side embeddings have to be identical; mixing models produces meaningless similarity scores.
- **Re-index after editing sources.** Changing the restrictions PDF or adding new sample plans requires re-running the relevant ingestion pipeline.
- **Tune chunking for restrictions.** If your restriction items are short, one-line entries, a smaller chunk size for the Restrictions index can sharpen retrieval.
- **Validate constraint adherence.** RAG reduces but doesn't eliminate the chance of the model slipping in a restricted item. Consider a downstream check or a stricter system message if this is high-stakes.

---

## Project structure (suggested)

```
restrictive-meal-planner/
├── README.md
├── flows/
│   ├── ingestion.json      # File/Read File → Split → Embed → Chroma (x2)
│   └── query.json          # Chat Input → dual Chroma → Prompt → LLM → Output
├── data/
│   ├── meal_plans/         # approved sample plan PDFs
│   └── Restrictions.pdf
└── chroma/                 # persisted vector store (gitignore this)
```
