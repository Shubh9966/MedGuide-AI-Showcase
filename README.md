<div align="center">

# 🩺 MedGuide AI — Multimodal Medical RAG & Triage Assistant

<p align="center">
  <a href="https://www.python.org/"><img src="https://img.shields.io/badge/Python-3.10%2B-blue?style=for-the-badge&logo=python&logoColor=white" alt="Python"></a>
  <a href="https://fastapi.tiangolo.com/"><img src="https://img.shields.io/badge/FastAPI-0.100%2B-009688?style=for-the-badge&logo=fastapi&logoColor=white" alt="FastAPI"></a>
  <a href="https://langchain-ai.github.io/langgraph/"><img src="https://img.shields.io/badge/LangGraph-Orchestration-1C3D5A?style=for-the-badge&logo=langchain&logoColor=white" alt="LangGraph"></a>
  <a href="https://groq.com/"><img src="https://img.shields.io/badge/Groq-AI_Inference-F55036?style=for-the-badge&logo=groq&logoColor=white" alt="Groq"></a>
  <a href="https://www.pinecone.io/"><img src="https://img.shields.io/badge/Pinecone-Vector_DB-000000?style=for-the-badge&logo=pinecone&logoColor=white" alt="Pinecone"></a>
  <a href="https://tavily.com/"><img src="https://img.shields.io/badge/Tavily-Clinical_Search-FF6F61?style=for-the-badge&logo=search&logoColor=white" alt="Tavily"></a>
  <a href="https://www.sqlite.org/"><img src="https://img.shields.io/badge/SQLite-Memory-003B57?style=for-the-badge&logo=sqlite&logoColor=white" alt="SQLite"></a>
  <a href="mailto:shubham.shivare9112@gmail.com"><img src="https://img.shields.io/badge/Source_Code-Private-critical?style=for-the-badge&logo=github" alt="Private Repository"></a>
</p>

**Transform text, voice, and medical image inputs into source-cited educational health guidance powered by Pinecone RAG, Tavily clinical search fallback, and LangGraph.**

</div>

---

## 🔒 Source Code Access Notice

> **Note:** The core proprietary production implementation, vector ingestion pipelines, and agent orchestration layers are hosted in an authenticated private repository.  
> If you are a recruiter, collaborator, or enterprise seeking to review the full codebase or evaluate live integration, please contact: **[shubham.shivare9112@gmail.com](mailto:shubham.shivare9112@gmail.com)**.

---
## 🏗️ Architecture & LangGraph Workflow

```mermaid
flowchart TD
    classDef entry fill:#1E293B,stroke:#0F172A,stroke-width:2px,color:#F8FAFC,font-weight:bold
    classDef multimodal fill:#7C3AED,stroke:#6D28D9,stroke-width:2px,color:#FFFFFF
    classDef retrieve fill:#0D9488,stroke:#0F766E,stroke-width:2px,color:#FFFFFF
    classDef route fill:#D97706,stroke:#B45309,stroke-width:2px,color:#FFFFFF
    classDef web fill:#0284C7,stroke:#0369A1,stroke-width:2px,color:#FFFFFF
    classDef generate fill:#2563EB,stroke:#1D4ED8,stroke-width:2px,color:#FFFFFF
    classDef memory fill:#059669,stroke:#047857,stroke-width:2px,color:#FFFFFF
    classDef limit fill:#DC2626,stroke:#B91C1C,stroke-width:2px,color:#FFFFFF
    classDef guard fill:#DB2777,stroke:#BE185D,stroke-width:2px,color:#FFFFFF

    START((USER INPUT)):::entry --> MOD{"Input Modality<br/>Text, Voice, or Image?"}:::route

    MOD -->|"Voice Audio - WebM"| STT["Groq STT API<br/>whisper-large-v3-turbo"]:::multimodal
    MOD -->|"Image / Camera - <= 3MB"| RL
    MOD -->|"Text Message"| RL

    STT -->|"Transcript sent as text"| RL{"IP Rate Limit - /api/chat<br/>Max 2 requests per IP per hour?"}:::limit

    RL -->|"No - 3rd request"| BLK["HTTP 429 Response<br/>Rate-limit message with contact email<br/>No LLM call made"]:::limit
    RL -->|"Yes - Image attached"| VIS["Groq Vision API<br/>qwen/qwen3.8-27b Observation"]:::multimodal
    RL -->|"Yes - Text only"| GRD

    VIS --> GRD{"1. guard Node<br/>Is this a medical question?<br/>Groq classifier llama-3.1-8b-instant"}:::guard

    GRD -->|"No - Non-medical"| REF["refuse Node<br/>Fixed reply: This is a medical chatbot<br/>No retrieval, no web, no answer LLM"]:::guard
    GRD -->|"Yes - Medical"| RET["2. retrieve Node<br/>Pinecone Top-5 Search - Score >= 0.70"]:::retrieve

    RET --> RT{"3. route Node<br/>Score < 0.80 or Latest Query?"}:::route

    RT -->|"No - Strong KB Match"| ANS["5. answer Node<br/>Groq LLM - openai/gpt-oss-120b"]:::generate
    RT -->|"Yes - Weak KB or Current Guideline"| WEB["4. web Node<br/>Tavily Clinical Guidance Search"]:::web

    WEB --> ANS
    ANS --> MEM["SQLite Checkpointer<br/>Save Thread State in medical_memory.db"]:::memory
    REF --> MEM
    MEM --> END((CITED RESPONSE + TTS)):::entry
```

**Graph Color Legend:**
🟪 **Purple (`Multimodal Vision & Whisper STT`)** · 🟩 **Teal (`Pinecone Vector Retrieval`)** · 🟧 **Amber (`Score & Recency Router`)** · 🟦 **Cyan/Blue (`Tavily Web Search & Groq LLM Generation`)** · 🟥 **Red (`IP Rate Limiter`)** · 🩷 **Pink (`Medical-Only Guardrail & Refuse Nodes`)** · 🟢 **Emerald (`SQLite Persistent Memory`)**
## ✨ Core Highlights

- 🧬 **Hybrid Medical RAG (`LangGraph`)**: Retrieves clinical excerpts from *The Gale Encyclopedia of Medicine* (`data/Medical_book.pdf`) via **Pinecone** (`kb_min_score >= 0.70`) and routes to **Tavily** web search when internal match scores fall below `0.80` or when current/latest clinical guidelines are requested.
- 👁️ **Multimodal Vision & Voice Support**: Analyzes uploaded or live camera photos via **Groq Vision** (`qwen/qwen3.8-27b`), transcribes voice queries using **Groq Whisper** (`whisper-large-v3-turbo`), and reads answers aloud with browser **Web Speech TTS**.
- 💾 **Persistent SQLite Conversation Memory**: Maintains stateful multi-turn sessions using `langgraph-checkpoint-sqlite` (`SqliteSaver` in `medical_memory.db`) keyed by `X-Conversation-ID` / `thread_id`.
- 🛡️ **Clinical Safety Guardrails**: Enforces strict non-diagnostic system prompts, flags emergency symptoms immediately, and cites exact `[PDF p.N]` page numbers or external clinical URLs.
- 🚧 **Medical-Only Input Guardrail**: The first LangGraph node (`guard`) classifies every question with a small Groq model (`llama-3.1-8b-instant`, override via `GROQ_GUARD_MODEL`). Non-medical questions go to a `refuse` node and get a fixed *"This is a medical chatbot, kindly ask medical questions only"* reply, with no Pinecone, Tavily, or answer-LLM calls. If the classifier is unavailable, the bot fails open so standard workflows do not break.
- ⏱️ **IP Rate Limiting**: Each IP can make `2` `/api/chat` requests per hour (sliding window, `X-Forwarded-For` aware for cloud proxies). The 3rd request receives an HTTP `429` status code with contact information, preserving API quotas.

---
---
## 🔍 Observability & Tracing

1. **Deterministic Attribution**:
   - Every `/api/chat` response returns an attribution breakdown (`["pinecone", "tavily", "vision"]`), specific PDF page numbers with cosine similarity scores (`score`), clinical web URLs, and conversation thread IDs.
2. **LangSmith Integration**:
   - MedGuide AI supports native tracing via `langgraph>=0.3` and `langchain-groq>=0.3`. Adding your credentials to `.env` enables full node latency tracking, input validation telemetry, and graph visualization:
     ```env
     LANGCHAIN_TRACING_V2=true
     LANGCHAIN_ENDPOINT=https://api.smith.langchain.com
     LANGCHAIN_API_KEY=lsv2_pt_your_langsmith_key
     LANGCHAIN_PROJECT=MedGuide-AI
     ```

---

## ⚠️ Medical Disclaimer

> This project is designed as an educational reference architecture and **not** as a certified clinical diagnostic tool or medical device. Always consult a qualified medical professional for health conditions and immediately contact emergency services for acute symptoms.

---

## 📬 Contact & Inquiries

For demonstrations, code audits, or technical collaboration:
- **Email:** [shubham.shivare9112@gmail.com](mailto:shubham.shivare9112@gmail.com)
- **GitHub:** [@Shubh9966](https://github.com/Shubh9966)
