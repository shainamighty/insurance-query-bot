# My Notes: Insurance Query Bot — Interview Prep

Personal notes on how this project actually works, so I can explain it confidently without re-reading the code every time.

## The one-line pitch

A RAG (Retrieval-Augmented Generation) app: upload a PDF insurance policy, ask a question in plain English, get an answer that's grounded ONLY in that document — not the model's general knowledge. Built collaboratively with a teammate; I know both the backend pipeline and the reasoning behind the design choices.

## The core flow (what I'd draw on a whiteboard)

```
User uploads PDF + types question
        ↓
Backend hashes the PDF bytes (SHA-256) → this hash = unique doc ID
        ↓
Have I seen this exact file before? (check Pinecone for that namespace)
   NO  → extract text → chunk it → embed chunks → store in Pinecone
   YES → skip straight to querying
        ↓
Embed the user's question
        ↓
Similarity search in Pinecone (top 5 closest chunks) for THIS document only
        ↓
Stuff those 5 chunks into a prompt: "answer ONLY using this context"
        ↓
Gemini generates the answer
        ↓
Answer shown in chat UI
```

## Why it's built this way (the "why" questions I should be ready for)

**Q: Why hash the file instead of using a filename or ID?**
Filenames aren't reliable — two users could name totally different files "policy.pdf," or the same file could get uploaded twice with different names. Hashing the raw bytes means identical files always produce the identical hash, so the "have I seen this before" check is content-based, not name-based. It also means I don't need a database just to track "which documents have I already processed" — Pinecone's own namespace listing does that job.

**Q: Why not just dump the whole PDF text into the prompt instead of chunking + embedding?**
Two reasons: (1) LLM context windows have limits, and insurance policies can be long; (2) even within budget, giving the model 40 pages every time is slow and expensive per query. Chunking + embedding lets me pull only the 5 most relevant paragraphs for each specific question — cheaper, faster, and the answers are more focused because the model isn't wading through irrelevant clauses.

**Q: Why does the splitter try paragraph breaks first, then lines, then sentences, then just hard-wrap?**
Because cutting a chunk off mid-sentence loses meaning — if a policy clause about "claim exclusions" gets sliced in half between two chunks, neither chunk fully represents the rule. Trying paragraph boundaries first keeps semantically complete ideas together as much as possible; the hard-wrap fallback only kicks in if a paragraph is so long there's no clean break at all.

**Q: Why store the chunk's own text inside Pinecone's metadata instead of a separate database?**
Pinecone's similarity search returns vectors + whatever metadata you attached to them — it does NOT hand back your original text. So metadata is the only place to keep the actual chunk text retrievable from the search result. Storing it there means one system (Pinecone) handles both "find the closest match" and "here's the text of that match," instead of syncing two separate stores.

**Q: What's the API key doing — is it OpenAI/Google's key?**
No — that's a separate, app-level `Authorization: Bearer` key I check myself in the endpoint. It's just gatekeeping for who can call *my* API; it has nothing to do with the Gemini or Pinecone keys, which are configured separately as env vars and never touch the client.

**Q: What would you improve if you had more time?**
Things I can say honestly: add caching per-question (right now the same question on the same doc re-runs the whole embed+search+generate cycle every time), add streaming responses instead of waiting for the full answer, handle multi-file / multi-turn conversations where follow-up questions reference earlier answers, and add proper error handling for malformed or scanned (non-text) PDFs.

## Files, briefly

- **`main.py`** — FastAPI app, the only HTTP-facing file. Has the endpoint, the doc-ID hashing, the Gemini prompt function, and the orchestration logic that ties everything together.
- **`data_processor.py`** — the actual document toolkit: PDF text extraction, chunking, embedding, and Pinecone upsert. `main.py` imports functions from here; a couple of things in this file (like the `__main__` test block) aren't used by the live API.
- **`frontend/`** — React + Vite chat UI. `ChatWindow.jsx` holds basically all the logic (file state, question state, loading state, the submit handler that builds the request and appends the response to the chat).
- **`ProcFile`** — one line, tells the hosting platform how to start the backend (`uvicorn main:app`).

## Things to double check before an interview

- [ ] Re-read `main.py` top to bottom once more so I can trace it live if asked to walk through code
- [ ] Be ready to explain top_k=5 as a tunable parameter, not a magic number — trade-off between more context vs. more noise/cost
- [ ] Know roughly what `gemini-embedding-001` vs `gemini-2.5-flash` are each doing (one embeds, one generates — don't mix them up)
- [ ] Be able to say clearly: "I worked on X specifically, my teammate worked on Y" if asked to divide credit
