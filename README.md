# Steam Game Similarity & Recommender System

## Overview

A system that leverages Steam game metadata, user-generated tags, and player reviews to build semantic embeddings and a knowledge graph. The goal is to enable:

1. **Similarity/difference analysis** between games based on player perception (not just genre labels)
2. **A recommender system** where a user describes what they enjoy (a game, a mechanic, a vibe) and receives the top-5 matching games with explanations

---

## Data Sources

All data is accessible via Steam's **public JSON APIs** (no authentication required for store endpoints):

- **Game details**: `https://store.steampowered.com/api/appdetails?appids={appid}`
  - Description, genres, categories, tags, price, metacritic, screenshots, developer/publisher
- **Reviews**: `https://store.steampowered.com/appreviews/{appid}?json=1`
  - Review text, vote (up/down), playtime at review, helpful votes, language filter, cursor-based pagination
- **Game list**: `https://api.steampowered.com/ISteamApps/GetAppList/v2/`
  - Full catalogue of app IDs to filter and rank

### Sample Size

- **Games**: Start with the **top 1,000 most-reviewed games** (covers the vast majority of the active player base and genre diversity)
- **Reviews per game**: **100–300 reviews** per game is a reasonable range
  - 100 reviews is the minimum to capture recurring themes beyond noise
  - 300 reviews hits diminishing returns for theme extraction — most dominant sentiments surface well before this
  - Filter for English, minimum 1 helpful vote, and minimum 1hr playtime to improve signal quality
- **Tags per game**: Steam typically surfaces **15–20 user-generated tags** per game; collect all of them — they are compact and highly informative

---

## Pipeline

### 1. Data Collection

Pull game metadata and reviews from the Steam API. Handle pagination, rate limiting, and caching.

| Tool | Notes |
|------|-------|
| **`httpx` + `asyncio`** | Async HTTP client — fast, modern, good for batch API calls with rate limiting control |
| **`steamspypi`** | Wrapper around SteamSpy API — provides pre-ranked lists by review count, owners, playtime. Useful for identifying the top-1000 target list without scraping |

Storage: save raw responses as JSON (one file per game) for reproducibility before any processing.

### 2. Text Processing & Cleaning

Clean review text, remove spam/low-effort reviews, and create aggregated text representations per game.

| Tool | Notes |
|------|-------|
| **`spaCy`** (with `en_core_web_sm` or `_md`) | Fast tokenization, lemmatization, stopword removal. Good for cleaning noisy user-generated text |
| **`textacy`** (built on spaCy) | Adds corpus-level operations, text statistics, and preprocessing pipelines on top of spaCy — less boilerplate for batch cleaning |

Considerations:
- Deduplicate near-identical reviews (copy-paste spam is common on Steam)
- Consider separating positive and negative review text per game — the contrast itself is informative
- Aggregate reviews into a single "review corpus" per game, or sample a representative subset

### 3. Embedding Generation

Create dense vector representations of each game from its combined textual data (description + review themes + tags).

| Tool | Notes |
|------|-------|
| **`all-MiniLM-L6-v2`** (via `sentence-transformers`) | Fast, lightweight general-purpose sentence embeddings. Good baseline for semantic similarity on informal text like reviews |
| **`all-mpnet-base-v2`** (via `sentence-transformers`) | Higher quality embeddings, slower. Better at capturing nuance — worth comparing whether the quality gain matters for this domain |

Considerations:
- Embed **three text fields separately** and experiment with concatenation vs. weighted averaging:
  1. Store description (marketing perspective)
  2. Aggregated positive reviews (player perspective — what works)
  3. Aggregated negative reviews (player perspective — what doesn't)
- Tags can be embedded as a joined string ("roguelike, cozy, base building") or used as discrete features alongside the embeddings
- Embedding dimension: 384 (MiniLM) vs 768 (MPNet) — relevant for storage and similarity computation at scale

### 4. Knowledge Graph Construction

Model relationships between games, genres, tags, developers, mechanics, and extracted themes.

| Tool | Notes |
|------|-------|
| **`NetworkX`** | Pure Python, easy to get started, good for prototyping and analysis (centrality, community detection). Keeps everything in-memory |
| **`Neo4j` (with `py2neo` or `neo4j` driver)** | Full graph database — better for querying complex traversals ("games by this developer that share 3+ tags with Game X"). Overkill for prototype, valuable if the project scales |

Node types: `Game`, `Tag`, `Genre`, `Developer`, `Publisher`, `Theme` (extracted)
Edge types: `HAS_TAG`, `IN_GENRE`, `DEVELOPED_BY`, `SIMILAR_TO` (embedding-based), `SHARES_THEME_WITH`

### 5. Theme / Topic Extraction

Extract latent topics from review corpora to enrich the graph and provide explainable dimensions.

| Tool | Notes |
|------|-------|
| **`BERTopic`** | You already know this one well. Works great for discovering topics across the review corpus. Can use the same sentence-transformer backbone as the embedding step |
| **`Top2Vec`** | Alternative that jointly learns topic vectors and document vectors — no need to predefine number of topics. Worth comparing whether it finds different/better clusters for game reviews |

The extracted topics become additional node attributes or edges in the knowledge graph.

### 6. Similarity Search & Recommendation

Given a query (game name, free-text description of preferences, or a combination), find the top-5 most similar games.

| Tool | Notes |
|------|-------|
| **`FAISS`** (Facebook AI Similarity Search) | Industry standard for fast approximate nearest neighbor search on embeddings. Efficient even at 10k+ vectors |
| **`ChromaDB`** | Lightweight vector database with built-in embedding support and metadata filtering. Easier API, good for prototyping. Allows filtering by tags/genre during similarity search |

Query modes to implement:
- **"Games like X"** — find nearest neighbors to Game X's embedding
- **"Games like X but more Y"** — interpolate between Game X's embedding and the embedding of concept Y
- **"I like A, B, C — what else?"** — average or weighted combination of multiple game embeddings
- **Free text** — "I want a cozy roguelike with base building and no combat" → embed the query and search

### 7. Interface (stretch goal)

A simple front-end for interactive exploration.

| Tool | Notes |
|------|-------|
| **Streamlit** | Fastest path to a working demo. Text input → recommendations with explanations |
| **Gradio** | Similar to Streamlit, slightly better for ML demo interfaces with input/output paradigms |

---

## Project Structure (initial)

```
steam-recommender/
├── README.md
├── config.yaml              # API settings, sample sizes, model choices
├── data/
│   ├── raw/                 # Raw API responses (JSON per game)
│   ├── processed/           # Cleaned text, merged datasets
│   └── embeddings/          # Stored vectors
├── src/
│   ├── collect.py           # Data collection from Steam API
│   ├── clean.py             # Text preprocessing
│   ├── embed.py             # Embedding generation
│   ├── graph.py             # Knowledge graph construction
│   ├── topics.py            # Topic extraction
│   └── recommend.py         # Similarity search & recommendation logic
├── notebooks/
│   ├── 01_exploration.ipynb
│   ├── 02_embedding_comparison.ipynb
│   └── 03_graph_analysis.ipynb
├── requirements.txt
└── .gitignore
```

---

## Open Questions

- **Review language**: Start English-only, or include multilingual reviews with translation (you have experience with `opus-mt` from the thesis project)?
- **Temporal dimension**: Should recent reviews weigh more? Games evolve significantly post-launch (e.g., No Man's Sky, Cyberpunk 2077)
- **Negative signal**: How to handle "overwhelmingly negative" games — are they similar to each other, or should sentiment be a separate axis?
- **Cold start**: Games with few reviews — rely more on store description + tags, or exclude from the initial dataset?
- **Evaluation**: How to measure recommendation quality without user studies? Possibly use tag overlap or genre co-occurrence as a proxy metric

---

## Tech Stack Summary

| Stage | Option A | Option B |
|-------|----------|----------|
| Data collection | `httpx` + `asyncio` | `steamspypi` |
| Text cleaning | `spaCy` | `textacy` |
| Embeddings | `all-MiniLM-L6-v2` | `all-mpnet-base-v2` |
| Knowledge graph | `NetworkX` | `Neo4j` |
| Topic extraction | `BERTopic` | `Top2Vec` |
| Similarity search | `FAISS` | `ChromaDB` |
| Interface | Streamlit | Gradio |
