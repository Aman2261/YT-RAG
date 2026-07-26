# YouTube video summarizer + Q&A

A small RAG (retrieval-augmented generation) application that takes a YouTube
video URL, generates a summary, and lets you ask follow-up questions about
the video's content — with answers grounded in the transcript and cited by
timestamp.

## How it works

1. **Transcript extraction** — pulls the video's existing captions using
   `youtube-transcript-api`. If no transcript is available (captions
   disabled, private video), the app surfaces a clear error rather than
   failing silently.
2. **Chunking** — the transcript is split into overlapping ~1000-character
   chunks (`RecursiveCharacterTextSplitter`), each tagged with an
   approximate timestamp.
3. **Embeddings + vector store** — each chunk is embedded locally with
   `sentence-transformers/all-MiniLM-L6-v2` (free, no API calls) and stored
   in an in-memory Chroma collection.
4. **Summarization** — a map-reduce summary: each chunk is summarized
   individually, then those partial summaries are combined into one
   coherent summary of the full video.
5. **Q&A (RAG)** — each question is embedded and matched against the
   stored chunks via semantic similarity search. The top-k most relevant
   chunks are passed to the LLM as context, so answers are grounded in
   the actual transcript instead of the model's general knowledge, and
   include the timestamp(s) they came from.

## Why these design choices

- **RAG instead of stuffing the whole transcript into the prompt**: long
  videos can easily exceed context limits, and retrieval keeps each answer
  focused on the most relevant parts of the video rather than diluting the
  prompt with irrelevant text.
- **Map-reduce summarization**: summarizing each chunk first (map) then
  combining those summaries (reduce) lets the pipeline handle videos of
  any length without hitting a single-call token limit.
- **Local embeddings (`sentence-transformers`) instead of a paid API**:
  keeps the project runnable at zero cost with no rate limits, while
  Groq is used only for the LLM calls (summarization + Q&A), which have a
  generous free tier.
- **Semantic (dense) retrieval rather than keyword search**: matches
  questions to transcript chunks by meaning, not exact wording — e.g. a
  question about "how performance scales" can retrieve a chunk about
  "time complexity" even with no shared keywords.

## Setup

1. Clone this repo and install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

2. Get a free API key from [console.groq.com](https://console.groq.com)
   and set it as an environment variable:
   ```bash
   cp .env.example .env
   # then edit .env and add your key
   ```
   Or export it directly:
   ```bash
   export GROQ_API_KEY=your_key_here
   ```

3. Run the app:
   ```bash
   streamlit run app.py
   ```

4. Paste a YouTube URL, click **Summarize**, then ask questions in the chat
   box below the summary.

## Project structure

```
├── main.py           # Core pipeline: transcript, chunking, embeddings, summarization, Q&A
├── app.py             # Streamlit UI
├── requirements.txt
├── .env.example
└── README.md
```

`main.py` can also be run standalone from the command line for quick
testing without the UI:
```bash
python main.py
```

## Known limitations

- Only works on videos that have captions available (manual or
  auto-generated). No audio-transcription fallback is implemented.
- The vector store is in-memory and per-session — nothing persists once
  the app restarts. Fine for a demo; a real deployment would use a
  persistent vector database.
- Timestamp matching for chunks is approximate (based on matching the
  first few words of a chunk back to its original transcript segment).

## Possible extensions

- Add a Whisper fallback for videos without captions.
- Persist the vector store per video so re-visiting a video skips
  re-processing.
- Add a "jump to timestamp" link in the Q&A source citations.
- Support multiple videos in one session for cross-video Q&A.
