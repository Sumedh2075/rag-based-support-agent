# Aster & Row AI Support Agent

An intelligent, secure, and robust customer support AI agent built for **Aster & Row**. The system combines Retrieval-Augmented Generation (RAG) over structured markdown policy documents, deterministic order-tracking tool execution, multi-turn conversation memory, and strict security/privacy guardrails.

---

## Setup Instructions

Follow these steps to set up and run the agent from a clean clone on Python 3.10+.

1. Clone the repository:
```bash
git clone [https://github.com/Sumedh2075/rag-based-support-agent.git](https://github.com/Sumedh2075/rag-based-support-agent.git)
cd rag-based-support-agent 
```
2. Create and activate a virtual environment:
```bash
python -m venv venv

# For Windows:
venv\Scripts\activate

# For Mac / Linux:
source venv/bin/activate
```
3. Install Required Dependencies:
```bash
pip install openai python-dotenv pytest
# openai       - Provides access to the OpenAI API for the support agent core logic
# python-dotenv - Loads environment variables (like your OPENAI_API_KEY) from a .env file
# pytest       - Runs the 20 deterministic test cases in test_agent.py
```
4. Set Environment Variables:
Create a .env file in the root directory and add the following variables:
```
OPENAI_API_KEY=your_openai_api_key_here
LOG_LEVEL=INFO
```
5. Run the CLI interface:
```
python cli.py
```

## Tech Stack & Architecture

* **Model & Core Logic:**\
  Python 3.13 standard library with regex-based intent classification and stateful multi-turn history propagation.
* **Retrieval Approach (RAG):**\
  Zero-dependency lexical chunker (`rag.py`) that indexes markdown files by headings, filters stop-words, and applies domain-specific business routing weights.
* **Storage Approach:**\
  Local flat-file architecture using JSON (`data/orders.json`) for mock order databases and markdown files (`knowledge-base/`) for policy documentation.
* **Observability:**\
  Plain structured JSONL logger (`traces.jsonl`) recording queries, retrieved passages, tool calls, latencies, and handoff triggers without leaking secrets.

---

## Architecture Overview

* **Input Processing:**\
  User queries are ingested via the CLI and evaluated against deterministic regex routing patterns to classify the intent (e.g., policy question vs. order lookup).
* **Context Retrieval & Tool Execution:**\
  * *RAG Flow:* If policy information is needed, `rag.py` parses the local markdown documents, executes lexical matching against user tokens, and retrieves the most relevant chunks.
  * *Tool Flow:* If an order ID is detected, the agent triggers the `order_lookup` tool to fetch live data from the local JSON database.
* **Generation & Guardrails:**\
  The retrieved context, tool outputs, and conversational history are passed to the LLM. Strict system instructions enforce boundary limits, triggering graceful handoffs for out-of-scope requests.
* **State Management & Telemetry:**\
  Output is delivered to the user while the interaction state is preserved in-memory for multi-turn continuity. Metrics are quietly logged to `traces.jsonl`.
## Architecture Overview

```text
Customer Message
       │
       ▼
[Is it safe & on-topic?] ──(No)──> Safe Boundary Reply
       │ (Yes)
       ▼
[Does it need an Order ID?] 
       ├─► Yes ──> Query Secure Orders Database (Masks PII) ──► Order Status
       └─► No  ──> Search Company Policy Files (RAG + Citations) ──► Policy Answer
```
## Evaluation Results
The evaluation suite runs via pytest, executing 20 deterministic test cases covering retrieval, tool execution, security guardrails, and handoffs.
```bash
pytest -v test_agent.py
```
| Capability Category | Passed Cases | Total Cases | Accuracy |
| :--- | :---: | :---: | :---: |
| Retrieval & Groundedness | 5 | 5 | 100.0% |
| Tool Use & Reliability | 5 | 5 | 100.0% |
| Security & Guardrails | 3 | 3 | 100.0% |
| Privacy & Handoffs | 4 | 4 | 100.0% |
| Multi-Turn Context | 3 | 3 | 100.0% |
| **OVERALL SCORE** | **20** | **20** | **100.0%** |

## Bug Diary

### 1. Tool Naming Mismatch
* **How I reproduced it:** Ran the evaluation suite; order lookup assertions threw `AssertionError: Expected tool 'order_lookup', got '['lookup_order']'`.
* **Root Cause:** The metadata logger in `agent.py` was tracking `lookup_order`, while test assertions and function signatures explicitly expected `order_lookup`.
* **The Fix:** Standardized all tool references across `agent.py` and test modules to use `order_lookup`.
* **Regression Test:** Verified via `test_visible_cases[valid-order-lookup]`.

### 2. Missing Boolean Flag in Tool Response
* **How I reproduced it:** Tested unknown order IDs (`ORD-9999`), which resulted in a `KeyError: 'found'` crash inside `agent.py`.
* **Root Cause:** The order lookup function returned raw error strings or dictionary data directly without wrapping them in a standard schema containing a `found` boolean flag.
* **The Fix:** Updated `order_lookup.py` to consistently return a dictionary wrapper containing `"found": True/False` alongside payload or error details.
* **Regression Test:** Verified via `test_visible_cases[unknown-order]`.

### 3. Strict String Matching on Punctuation
* **How I reproduced it:** The `trailplus-return-window` test case failed despite retrieving the correct policy content.
* **Root Cause:** The test suite asserted strict matching against "45 calendar days", but the knowledge base chunk was hyphenated as "45-calendar-day".
* **The Fix:** Refined token normalization and text parsing rules in `rag.py` to handle hyphenated interval expressions smoothly.
* **Regression Test:** Verified via `test_visible_cases[trailplus-return-window]`.

## Production Improvements

### 1. Advanced Vector & Hybrid Retrieval
* **Current State:** Zero-dependency lexical chunker matching keyword intersections.
* **Production Goal:** Integrate a vector database (e.g., PGVector or Chroma) paired with hybrid sparse-dense retrieval to effectively handle heavily paraphrased customer queries and deep semantic context.

### 2. Persistent Stateful Management
* **Current State:** Conversation history is held entirely in-memory during CLI runtime and resets between sessions.
* **Production Goal:** Implement robust user session management backed by Redis or an external database to persist context across continuous multi-turn customer interactions.

### 3. Real-Time ERP/CRM Webhook Integration
* **Current State:** Order lookups rely on a static local JSON file (`data/orders.json`).
* **Production Goal:** Connect the order lookup tool to a live, authenticated enterprise resource planning (ERP) or customer relationship management (CRM) webhook for real-time order tracking and modifications.

## AI Coding Tools & Reflection
**Tools Used:** Gemini 1.5 Pro / Flash.

**Purpose:** Assisted with modular code scaffolding, designing deterministic regex routing patterns, formulating regex matchers for order IDs, and debugging pytest parameters.

**AI Mistake Example:** Early in development, the AI suggested implementing a heavy vector database (ChromaDB + OpenAI embeddings) for the static markdown knowledge base. This was overly complex and bloated project dependencies unnecessarily, when a clean, zero-dependency lexical indexer (rag.py) was faster, deterministic, and fully sufficient for the task requirements.

## Demonstration Video / GIF

A demonstration showing knowledge-base retrieval with citations, order lookups, multi-turn tracking, security refusals, and the pytest execution suite.

👉 [Watch the Agent Demonstration Video](https://drive.google.com/file/d/1k7H-WlY6jNmAvDIPE_2BtxDL2MRIDmjg/view?usp=sharing)



