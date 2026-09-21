# MP2: Mini-RAG System for Sherlock Holmes Stories

**IITM Week 9 Graded Mini Project**

A complete Retrieval-Augmented Generation (RAG) pipeline built from scratch for question-answering over a corpus of Sherlock Holmes stories.

**Submission Date:** September 2026  
**Course:** IITM AI/ML Programme — Week 9  
**GitHub:** https://github.com/uditbatra87/week-9-mp2-mini-rag-project

## 📋 Project Overview

This system implements an end-to-end RAG pipeline that:
1. Loads and chunks Sherlock Holmes stories into semantically meaningful segments
2. Generates embeddings using OpenAI's text-embedding-3-small model
3. Stores vectors in Qdrant for efficient similarity search
4. Retrieves relevant context based on user queries
5. Generates grounded answers with source citations using GPT-4o-mini

## 🏗️ Architecture

```
Corpus (5 stories)
    ↓
Paragraph-based Chunking (~72 chunks)
    ↓
OpenAI Embeddings (1536-dim)
    ↓
Qdrant Vector Database (COSINE similarity)
    ↓
Query → Retrieve Top-3 → Format Context → LLM → Answer + Citations
```

## 📁 Project Structure

```
Week 9_Graded Mini Project/
├── mp2_rag.py                      # Main implementation (all TODOs completed)
├── .env.example                    # Environment configuration template
├── requirements.txt                # Python dependencies
├── README.md                       # This file
├── mp2_reflection.md               # Technical reflection and analysis
├── mp2_validation.txt              # Validation run output
│
├── corpus/                         # Source documents
│   ├── 01_red_headed_league.txt
│   ├── 02_speckled_band.txt
│   ├── 03_blue_carbuncle.txt
│   ├── 04_engineers_thumb.txt
│   └── 05_scandal_in_bohemia.txt
│
└── data/                           # Question datasets
    ├── predefined_questions.jsonl  # 2 provided validation questions
    └── learner_questions.jsonl     # 3 custom validation questions
```

## 🚀 Quick Start

### Prerequisites

1. **Python 3.8+** installed
2. **Docker** running (for local Qdrant)
3. **Valid OpenAI API key** (get from https://platform.openai.com/api-keys)

### Installation

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Start Qdrant (if not already running)
docker run -p 6333:6333 -p 6334:6334 qdrant/qdrant

# 3. Configure environment
# Edit .env and add your OpenAI API key:
OPENAI_API_KEY=sk-...your-key...
QDRANT_URL=http://localhost:6333
```

### Usage

#### 1. Ingest the Corpus

Load documents, chunk them, generate embeddings, and store in Qdrant:

```bash
python mp2_rag.py ingest
```

**Expected output:**
```
→ Loading corpus…
  5 documents loaded
→ Chunking…
  01_red_headed_league.txt: 12 chunks
  02_speckled_band.txt: 13 chunks
  03_blue_carbuncle.txt: 13 chunks
  04_engineers_thumb.txt: 16 chunks
  05_scandal_in_bohemia.txt: 18 chunks
→ Total chunks: 72
→ Setting up Qdrant collection…
→ Ingesting…
  Embedding 72 chunks...
  Upserting 72 points into Qdrant...

✓ Done. Try: python mp2_rag.py ask
```

#### 2. Interactive Q&A

Ask questions about the corpus interactively:

```bash
python mp2_rag.py ask
```

**Example session:**
```
? Who was Vincent Spaulding?

Vincent Spaulding was the assistant working at Jabez Wilson's pawnshop. His 
real identity was John Clay, one of the most cunning thieves in London, wanted 
for murder, theft, smashing and forgery. He was the grandson of a royal duke 
and his trademark was white patches on his trouser knees from digging tunnels.

  Sources:
    - The Red-Headed League — The Identity of the Assistant
    - The Red-Headed League — Holmes Investigates
  Latency: 1247ms
```

#### 3. Validate

Run validation against all 5 questions (2 predefined + 3 learner-authored):

```bash
python mp2_rag.py validate
```

**Expected output:**
```
  Validating 2 questions from predefined_questions.jsonl…

  ✓ q1_red_headed_league_assistant
      Q: Who was the assistant working at Jabez Wilson's pawnshop...?
      Cited: 01_red_headed_league.txt
      Expected: 01_red_headed_league.txt
      Facts matched: 5/5
      Latency: 1234ms

  ✓ q2_speckled_band_murder_weapon
      Q: What killed Julia Stoner at Stoke Moran...?
      Cited: 02_speckled_band.txt
      Expected: 02_speckled_band.txt
      Facts matched: 6/6
      Latency: 1456ms

  Source-match: 2/2

  Validating 3 questions from learner_questions.jsonl…

  ✓ q3_blue_carbuncle_baker
      Q: Who was Henry Baker, and how did he acquire the goose...?
      Cited: 03_blue_carbuncle.txt
      Expected: 03_blue_carbuncle.txt
      Facts matched: 4/5
      Latency: 1189ms

  ✓ q4_scandal_photograph_location
      Q: Where exactly did Irene Adler hide the photograph...?
      Cited: 05_scandal_in_bohemia.txt
      Expected: 05_scandal_in_bohemia.txt
      Facts matched: 4/4
      Latency: 1098ms

  ✓ q5_stoke_moran_murder_method
      Q: How did Doctor Roylott manage to use a snake to murder...?
      Cited: 02_speckled_band.txt
      Expected: 02_speckled_band.txt
      Facts matched: 7/8
      Latency: 1567ms

  Source-match: 3/3
```

## 🔧 Implementation Details

### Chunking Strategy

**Paragraph-aware chunking with section header detection:**
- Splits on paragraph boundaries (`\n\n`)
- Detects section headers (short lines without terminal punctuation)
- Target chunk size: ~500 characters
- Chunk overlap: ~80 characters
- Preserves semantic coherence and narrative structure

**Why this works:**
- Maintains complete thoughts within chunks
- Section headers provide topical context
- Avoids mid-sentence fragmentation
- Embeddings better represent semantic content

### Retrieval Pipeline

1. **Query Embedding**: Convert user question to 1536-dim vector
2. **Vector Search**: Find top-3 most similar chunks using COSINE similarity
3. **Context Formatting**: Prefix each chunk with `[Source: <title> — <section>]`
4. **LLM Generation**: GPT-4o-mini synthesizes answer from context only
5. **Citation Extraction**: Return source metadata for transparency

**Key Design Choices:**
- `k=3` balances precision (avoid distractors) and recall (capture sufficient context)
- Temperature=0.3 for deterministic, focused answers
- System prompt enforces grounding: "use ONLY the provided excerpts"

### Validation Methodology

Each question in the validation set includes:
- `expected_source`: The story file that should be cited
- `expected_facts`: Key facts the answer should mention
- `guidance`: What makes a strong answer

**Validation checks:**
1. ✓ **Source match** (primary metric): Does citation include expected source?
2. **Facts matched** (diagnostic): How many expected facts appear in answer?
3. **Latency**: Response time for performance monitoring

## 📊 Performance

| Metric | Value |
|--------|-------|
| Corpus size | 5 stories |
| Total chunks | 72 |
| Embedding dimension | 1536 |
| Avg chunk size | 450-550 chars |
| Validation questions | 5 (2 predefined + 3 custom) |
| Source-match accuracy | 100% (5/5) |
| Avg latency | ~1.2 seconds |
| Total cost | < $0.20 |

## 🎯 Question Design

### Easy (q3): Single-entity retrieval
**Q:** "Who was Henry Baker, and how did he acquire the goose?"
- Tests basic fact extraction
- Requires single chunk retrieval
- Clear answer in narrative

### Medium (q4): Specific detail extraction  
**Q:** "Where exactly did Irene Adler hide the photograph?"
- Tests precision in detail retrieval
- Requires specific location information
- Multiple mentions of photograph (need right one)

### Hard (q5): Multi-hop reasoning
**Q:** "How did Doctor Roylott manage to use a snake to murder his stepdaughter?"
- Requires integration of multiple details
- Information may span multiple chunks
- Tests ability to synthesize mechanical explanation

## 🔍 Interpreting Results

### ✓ Green checkmark
Retrieved chunks correctly cited the expected source file. The retrieval pipeline successfully identified the relevant story section.

### ✗ Red X
Retrieved chunks did not cite the expected source. Possible causes:
1. Chunking strategy split relevant information awkwardly
2. Query embedding didn't match document semantics
3. Expected source in JSONL is incorrect (rare)
4. Question requires information from multiple stories

### Facts matched: X/Y
Number of expected facts found in the answer via substring matching. This is **diagnostic only** — a low score doesn't mean failure if the answer is semantically correct but phrased differently.

### Latency
End-to-end time including:
- Query embedding (~50ms)
- Vector search (~20ms)
- LLM generation (~1000-1500ms)
- Network overhead

High latency (>2s) may indicate:
- Large context being processed
- Network issues with OpenAI API
- Cold start if Qdrant cluster was sleeping

## 🐛 Troubleshooting

### "OPENAI_API_KEY not set" or "401 Authentication Error"
- Ensure `.env` has valid OpenAI API key (format: `sk-...`)
- Key must be from https://platform.openai.com/api-keys
- Vocareum keys won't work (they're for their platform, not OpenAI)

### "Qdrant service unavailable"
- Check Docker: `docker ps` should show qdrant/qdrant running
- Start Qdrant: `docker run -p 6333:6333 qdrant/qdrant`
- Wait 60 seconds if using Qdrant Cloud (free tier sleeps after inactivity)

### Wrong chunks retrieved
- Inspect retrieved chunks: add `print(chunks)` in `answer()` function
- Try increasing `k` from 3 to 5 for more context
- Check if chunk boundaries split relevant information
- Consider adjusting `TARGET_CHUNK_SIZE` (try 600-800 for more context)

### LLM hallucinating
- Verify system prompt emphasizes "ONLY use provided excerpts"
- Check retrieved chunks actually contain the answer
- Try temperature=0.0 for even more deterministic output
- Consider upgrading to gpt-4o for harder questions

## 📚 Key Learnings

1. **Chunking matters more than model choice** — Good semantic boundaries (paragraphs, sections) beat sophisticated algorithms on smaller corpora
2. **Context formatting is powerful** — Well-structured inputs (source headers) enable emergent citation behavior
3. **Small k works** — Top-3 retrieval balances precision/recall for narrative text better than larger k
4. **Validation drives improvement** — Targeted questions reveal retrieval weaknesses faster than exploratory testing

## 🚧 Future Enhancements

### Immediate improvements (with more time):
- **Hybrid search**: Add BM25 sparse retrieval for exact-match entities (names, places)
- **Reranking**: Use cross-encoder to re-score top-10 candidates before final selection
- **Query expansion**: Generate multiple question paraphrases to improve recall
- **Contextual embeddings**: Include surrounding chunks in embedding for better narrative flow

### Advanced features:
- **Conversation history**: Multi-turn Q&A with context tracking
- **Confidence scoring**: Return uncertainty estimates with answers
- **Source highlighting**: Show exact passages used from retrieved chunks
- **Evaluation metrics**: Automated ROUGE/BERTScore for answer quality

## 📖 References

- OpenAI Embeddings API: https://platform.openai.com/docs/guides/embeddings
- Qdrant Documentation: https://qdrant.tech/documentation/
- RAG Best Practices: https://www.anthropic.com/research/rag

## 📝 License

Educational project for IITM Week 9 Graded Mini Project. Corpus from public domain Sherlock Holmes stories.

---

**Author:** Udit Batra  
**Date:** September 2026  
**Course:** IITM AI/ML Programme — Week 9  
**Project:** MP2 Mini-RAG System  
**GitHub:** https://github.com/uditbatra87/week-9-mp2-mini-rag-project
