# Lab: build a RAG assistant over real documents

Two days, four hours each. You start with documents, a Dockerfile and an empty `app/`
folder, and you finish with a working assistant that answers policy questions with citations
and refuses to invent anything.

There is no front end to build and no API to write. The page is Streamlit, which means it is
Python calling your own functions, in the same container as everything else.

Eleven files in `app/`, six on day one and five on day two.

## What you are given

```
data/documents/       37 HR policy documents in six formats, and manifest.json
data/evaluation.json  the 22 questions your assistant must get right
app/__init__.py       empty - every other file in here, you write
Dockerfile            the image your code runs in
docker-compose.yml    Postgres with pgvector, plus your app
requirements.txt      the libraries
.env.example          copy to .env and put your Anthropic key in it
```

Nothing in `app/` exists yet except `__init__.py`. That is the lab.

## What you are building

```
DAY 1  the offline half: turn documents into something searchable

   data/documents/  ──►  read  ──►  chunk  ──►  embed  ──►  PostgreSQL + pgvector

DAY 2  the online half: turn a question into an answer

   question  ──►  embed  ──►  search  ──►  Claude  ──►  answer with citations
```

## Day 1 - ingestion

| # | File | What it does | Roughly |
|---|---|---|---|
| 1 | `config.py` | Every setting, read from environment variables | 20 min |
| 2 | `documents.py` | Read PDF, DOCX, CSV, HTML, TXT and Markdown into sections | 60 min |
| 3 | `chunking.py` | Cut sections into pieces small enough to search | 30 min |
| 4 | `embeddings.py` | Turn text into vectors | 20 min |
| 5 | `search_index.py` | Create the pgvector table, write chunks, search it | 60 min |
| 6 | `ingest.py` | Run the whole pipeline, and check the manifest first | 45 min |

**End of day 1 you can prove it works:**

```bash
docker compose run --rm ingest
docker compose exec postgres psql -U hr_app -d hr -c \
  "SELECT title, section FROM hr_policy_chunks LIMIT 5;"
```

242 chunks from 37 documents. If you get fewer, something is being silently dropped - which
is the single most common RAG bug and the reason `ingest.py` refuses to finish when a
document produces no chunks.

## Day 2 - answering

| # | File | What it does | Roughly |
|---|---|---|---|
| 7 | `retrieval.py` | Embed the question, search, throw away weak matches | 45 min |
| 8 | `assistant.py` | Give Claude the chunks and the rules; get an answer with citations | 75 min |
| 9 | `conversations.py` | Store chats in Postgres so a refresh does not lose them | 30 min |
| 10 | `main.py` | The Streamlit page: a chat box, the answers, the sources | 40 min |
| 11 | `evaluate.py` | Run the golden set; measure the relevance threshold | 45 min |

**End of day 2:**

```bash
docker compose up -d
# open http://localhost:8501
docker compose exec app python -m app.evaluate     # 22/22
```

## The four ideas this lab is really about

Everything else is plumbing. These are the parts that separate a working assistant from a
demo that falls over the first time somebody asks it something awkward.

**1. Retrieval can succeed and still be wrong.** Two documents in this corpus are the same
policy at different dates. One says you may carry over 5 days, the other says 10. Both read
as confidently as each other. Your search has to exclude the superseded one, and your
evaluation has to check that it did.

**2. Vector search never says "I don't know".** It always returns its closest matches, even
for "hello". If you do not add a relevance floor yourself, the assistant will summarise
whichever policy is least unlike "hello" and look broken. You will measure where that floor
belongs rather than guessing it.

**3. The hardest question is "how many days do I have left?"** Retrieval works perfectly -
the time off policy comes back with a high score - and a model trying to be helpful will
produce a number it has no way of knowing. No threshold catches this. It takes an explicit
rule in the prompt, and its own evaluation cases.

**4. An answer you cannot check is a rumour.** Every answer cites the document, section and
version it came from, and the citation numbers have to match the sources shown underneath.

## Rules for the two days

- Type the code. Do not paste it. You will read it more carefully.
- Run each file as you finish it. Six broken files at once is not debuggable.
- When something does not work, look at the data before the code: `psql` the table, print a
  chunk, count the rows.
