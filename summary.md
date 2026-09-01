# Insurance Query Bot 

## 1. What this project even is, in plain words

Imagine you have a 40-page insurance policy PDF and you want to ask "what's the waiting period for maternity claims?" instead of reading all 40 pages.

This app lets you upload the PDF, type that question, and get an answer that comes ONLY from that specific document — not from the AI's general knowledge, not guessed, not from some other policy. If the answer isn't in the document, it should say so (not make something up).

This pattern — "answer questions using only a specific document as the source of truth" — is called **RAG: Retrieval-Augmented Generation**. "Retrieval" = finding the relevant part of the document. "Generation" = the AI writing a natural-language answer using that retrieved part. RAG exists because AI models can't just read a whole book every time you ask one question about it — it's slow, expensive, and the model might get confused by irrelevant parts. So instead, you find the SPECIFIC relevant chunk first, then only show the model that chunk.

## 2. Why you can't just paste the whole PDF into a chat prompt

Two reasons:
- **Context window limits**: every AI model has a maximum amount of text it can read at once. A big enough document might not even fit.
- **Cost and speed**: even if it fits, sending 40 pages of text to the AI on every single question is slow and expensive, especially if you're going to ask 10 questions about the same document.

So instead, the strategy is: break the document into small pieces once, store those pieces in a searchable way, and for each question only pull out the 5 pieces that are actually relevant to that question.

## 3. Key concept: what is an "embedding"?

An embedding is a way of turning text into a list of numbers (a vector) that captures its *meaning*. Two pieces of text that mean similar things will have embedding vectors that are mathematically "close" to each other, even if they use completely different words.

Example: "cancellation policy" and "how do I cancel my plan" would have embeddings close to each other, because they mean similar things — even though barely any words match.

This is the trick that makes "find the relevant part of the document" possible without literally searching for exact keyword matches.

## 4. Key concept: what is a "vector database" (Pinecone)?

A normal database is good at finding exact matches ("find the row where id = 5"). A vector database is built to answer a different question: "out of everything stored here, which items have embeddings closest to THIS embedding?" That's exactly what we need — "which chunks of this policy document are most relevant to this specific question?"

**Pinecone** is the vector database this project uses. You store embeddings in it, and later ask it "give me the top 5 closest matches to this new embedding" — it does the math and hands back the closest ones.

**Namespace** = a way of keeping different documents separate inside Pinecone, like folders. Each uploaded PDF gets its own namespace so that a search for "policy A's chunks" never accidentally returns chunks from "policy B."

## 5. The full pipeline, step by step

### Step 1 — Someone uploads a PDF + types a question
This happens in the React frontend (`ChatWindow.jsx`). It bundles the file and question together and sends it to the backend.

### Step 2 — "Have I seen this exact file before?"
The backend takes the raw bytes of the PDF and runs them through **SHA-256**, a hashing algorithm. A hash is like a fingerprint: the exact same file always produces the exact same hash, and even a tiny change to the file produces a totally different hash.

This hash becomes the Pinecone **namespace** for that document. Why do this instead of using the filename? Because filenames are unreliable — two different files could share a name, or the same file could be uploaded under different names. Hashing the actual content means "have I seen this file before" is based on what's IN the file, not what it's CALLED.

The backend checks: does Pinecone already have chunks stored under this hash's namespace?
- If **yes** → skip straight to Step 5 (no need to reprocess a file we've already handled).
- If **no** → continue to Step 3.

### Step 3 — Extract text from the PDF
Using a library called **PyMuPDF** (imported as `fitz`), the backend pulls the plain text out of the PDF pages.

### Step 4 — Chunk the text, then embed it, then store it
**Chunking**: the extracted text is broken into pieces (roughly 1000 characters each, with 200 characters of overlap between consecutive chunks so context doesn't get lost right at the boundary). 

The chunking isn't a blind "just cut every 1000 characters" — it tries to break at natural points in this order of preference: paragraph breaks first, then line breaks, then sentence endings, then just spaces — and only does a hard, mid-word cut as an absolute last resort. This matters because if a chunk got cut off in the middle of a sentence like "claims are excluded if the policyholder..." you'd lose the second half of that rule, and the AI might misread the meaning.

**Embedding**: each chunk of text gets sent to Google's `gemini-embedding-001` model, which converts it into a vector of numbers (see Concept 3 above).

**Storing**: each chunk's vector gets uploaded ("upserted") into Pinecone, tagged with that document's namespace. Crucially, the actual chunk TEXT itself is also stored as "metadata" attached to the vector — because Pinecone's search only gives back vectors + whatever metadata you attached, never the original text on its own. So storing the text as metadata is the only way to later get the actual words back out of a search result.

### Step 5 — Answer the question
The user's typed question also gets turned into an embedding (same `gemini-embedding-001` model, so it's comparable to the document's chunk embeddings).

Pinecone is asked: "within this document's namespace, give me the 5 chunks whose embeddings are closest to this question's embedding" (this is the `top_k=5` similarity search).

Those 5 chunks' text gets joined together into one block of "context."

### Step 6 — Generate the actual answer
That context block + the original question get sent to Google's `gemini-2.5-flash` model, with an instruction along the lines of: "answer this question using ONLY the following context, don't use outside knowledge." This is the one place in the whole project where prompt-writing/prompt engineering actually happens.

### Step 7 — Send it back to the user
The generated answer travels back through the API to the frontend, which adds it to the chat window as a new message bubble.

## 6. Where each piece of logic actually lives (the files)

- **`main.py`** — the only file that talks to the internet (FastAPI). Contains:
  - The setup code that loads API keys and connects to Gemini + Pinecone (runs once when the app starts)
  - The hashing function (Step 2)
  - The prompt-building function that calls Gemini for the final answer (Step 6)
  - The orchestration function that runs Steps 2 through 6 in order
  - The actual `/query/run` endpoint — checks the app's own `Authorization: Bearer <API_KEY>` header (this is MY app's own gatekeeping, completely separate from the Gemini/Pinecone API keys), rejects anything that isn't a PDF, and kicks off the whole pipeline

- **`data_processor.py`** — the toolbox of functions that do the actual document work: pulling text out of PDFs, chunking it, generating embeddings, and pushing chunks into Pinecone. `main.py` calls into these functions rather than doing this work itself.

- **`requirements.txt`** — the list of Python libraries this needs: `fastapi`/`uvicorn` (the web server itself), `PyMuPDF` (PDF reading), `google-generativeai` (talking to Gemini), `pinecone` (talking to the vector database), `python-dotenv` (reading the `.env` file where secret keys live). A couple of listed dependencies aren't actually used anymore — leftovers from earlier versions.

- **`ProcFile`** — one line telling a hosting service (like Render) how to start the server in production.

- **`frontend/`** — the React chat interface:
  - `ChatWindow.jsx` is where basically everything happens: it tracks the selected file, the typed question, and a loading spinner state; when you hit send, it packages the file + question and sends it to the backend, then adds the response to the visible chat.
  - `App.jsx` just holds which chat is currently active and passes it down — it doesn't render much itself.
  - The rest (`main.jsx`, `vite.config.js`, CSS files) are mostly standard project scaffolding, not unique logic.

## 6.5 Extra file-by-file details (the finer points, in case I get asked specifics)

- **`main.py` client setup** — the `.env` loading, Gemini SDK config, and Pinecone client all run once, right when the app starts up (at import time), not on every request. So if an API key is wrong, the app would fail immediately on startup, not on the first user request.

- **`data_processor.py` has some unused code** — there's a `create_document_id` function and an `if __name__ == "__main__":` block in there that were used for standalone testing while building this, but the live API doesn't call them. Good to know so I don't get caught off guard if asked "what does this function do" about something that's actually dead code.

- **`requirements.txt` has leftover dependencies** — `requests` is listed but not really used anymore (it was for the old version where you gave a PDF URL instead of uploading a file directly). `openai` is also listed but nothing in the code actually calls it — a leftover from before deciding to use Gemini instead. Worth knowing so I don't claim I used OpenAI anywhere.

- **`.gitignore` (root)** — keeps `.env` (real API keys), `venv/` (the Python virtual environment folder), and `__pycache__/` (compiled Python bytecode) out of git. This matters because `.env` holds actual secret keys — if it got committed, anyone could see and misuse them.

- **`index.html`** — just a single empty `<div id="root">` and a script tag. React takes over from there and renders everything inside that div. The page's browser tab title also gets set in this file.

- **`main.jsx`** — the actual entry point of the React app. It wraps everything in `React.StrictMode`, which is a development-only tool that helps catch bugs by intentionally double-running some functions during development — it has zero effect on the real, deployed version users see.

- **`App.jsx`** — very thin. It holds one piece of state (which chat is currently open) and passes it down to `ChatWindow`, but doesn't render any visible UI itself beyond a wrapper div.

- **`App.css` vs `index.css`** — `App.css` has all the actual visual design (chat bubbles, input bar, colors, the attach button). `index.css` is mostly just what Vite includes by default when you create a new React project (basic light/dark mode styles) — it's still loaded, but it's not really doing the visual work; `App.css` is.

- **`vite.config.js`** — minimal, just turns on the plugin that lets Vite understand JSX and enables "Fast Refresh" (auto-reloading the browser when you edit code during development).

- **`eslint.config.js`** — this is a code-quality linter config. It enforces React Hooks rules (catches common mistakes with `useState`/`useEffect`) and warns about unused variables — except variable names starting with a capital letter or underscore, which is a common convention meaning "yes I know this is unused, I meant to leave it here."

- **`package.json` / `package-lock.json`** — lists the frontend's dependencies: React 19, Vite 5, and a library called `uuid` (used to generate unique IDs for each chat session). The ESLint tools mentioned above are listed separately as "devDependencies" — meaning they're only needed while developing, not in the actual deployed app.

- **`.gitignore` (frontend)** — keeps `node_modules/` (all the downloaded JS packages — huge, and regenerable from `package.json`), `dist/` (the built/compiled output folder), and the frontend's own `.env` file (which holds the same shared `API_KEY` needed to talk to the backend) out of git.

## 7. Questions I should be ready to answer out loud (with the short answers)

**"Walk me through what happens when a user uploads a file and asks a question."**
→ Recite Section 5 above, step by step, in my own words.

**"Why hash the file instead of just using its name?"**
→ Filenames aren't reliable (duplicates, renames). Hashing the actual bytes means identical content is always recognized as identical, regardless of what it's called.

**"Why chunk the document instead of feeding the whole thing to the model?"**
→ Context window limits + cost/speed. Only sending the 5 most relevant chunks per question is cheaper and keeps the answer focused.

**"How does the app know which chunks are relevant to a question?"**
→ The question gets embedded into a vector too, and Pinecone finds the document's chunks whose vectors are mathematically closest to it — that's the whole trick behind semantic (meaning-based) search instead of literal keyword matching.

**"What's the API_KEY header for — is that Google's key?"**
→ No. That's my own app-level authorization check — a gate I built myself to control who can call my endpoint. It's completely separate from the Gemini and Pinecone API keys, which live in environment variables and are never exposed to whoever's calling my API.

**"What would you improve given more time?"**
→ Honest answers: cache repeated questions on the same document instead of rerunning the whole pipeline every time; stream the answer back instead of waiting for the full response; support follow-up questions that reference earlier answers in the same conversation; handle scanned/image-only PDFs that have no extractable text (would need OCR).

**"What did you personally build vs. your teammate?"**
→ [Fill this in honestly before the interview — know your actual contribution so you don't fumble this live.]

## 8. One-paragraph summary if someone says "just describe it in 30 seconds"

"It's a RAG-based Q&A tool for insurance policy PDFs. When you upload a document, it gets hashed, chunked, turned into embeddings, and stored in a Pinecone vector database under a namespace unique to that file. When you ask a question, the question also gets embedded, Pinecone returns the most semantically relevant chunks, and those get passed to Gemini along with the question so it can generate an answer grounded only in that document — not the model's general knowledge."
