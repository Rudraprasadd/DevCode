# DevPilot

> An AI-powered GitHub codebase assistant that helps developers understand repositories through natural-language questions and source-grounded answers.

DevPilot connects to a user’s GitHub account, indexes selected repositories, and lets the user chat with their codebase. Instead of sending an entire repository to an LLM, it uses Retrieval-Augmented Generation (RAG) to find the most relevant code chunks first, then generates an answer from that context. Answers include clickable source-file citations so users can verify the result.

## Features

- **GitHub OAuth2 authentication** — Sign in with GitHub and access repositories the user has authorized.
- **Repository sync** — View public and private repositories available through the authenticated GitHub account.
- **Asynchronous indexing** — Indexing runs in the background while the UI displays progress, chunk count, and errors.
- **Code-aware RAG** — Supported code and configuration files are chunked, embedded, and stored for semantic search.
- **Repository-scoped search** — Each query retrieves context only from the selected repository.
- **Streaming chat** — LLM output is streamed token-by-token to the browser using Server-Sent Events (SSE).
- **Citations and history** — Chat sessions, messages, and source-file citations are persisted for later review.
- **Security controls** — Repository ownership validation, encrypted GitHub token storage, authenticated APIs, and configurable CORS.

## Architecture

```text
┌──────────────────────┐
│  Next.js Frontend    │
│  Dashboard + Chat UI │
└──────────┬───────────┘
           │ REST APIs / SSE
┌──────────▼───────────┐       ┌──────────────────┐
│ Spring Boot Backend  │◄─────►│ GitHub REST API  │
│ Security + Spring AI │       │ OAuth2 + Repos   │
└──────┬─────────┬─────┘       └──────────────────┘
       │         │
       │         └────────────────► OpenAI
       │                            Chat + Embeddings
       ▼
┌──────────────────────────────────────────┐
│ PostgreSQL + pgvector                     │
│ Users · Repositories · Chats · Embeddings │
└──────────────────────────────────────────┘
```

## How it works

### 1. Authentication and repository access

1. The user signs in through GitHub OAuth2.
2. Spring Security creates an authenticated server session.
3. DevPilot syncs repositories available to that GitHub user.
4. The GitHub access token is encrypted before it is persisted.

### 2. Repository indexing

When a user clicks **Index**, the backend starts a background job and returns immediately with an `INDEXING` status.

```text
Select repository
      ↓
Fetch GitHub repository tree
      ↓
Filter indexable files
      ↓
Fetch file content
      ↓
Split content into chunks
      ↓
Create embeddings
      ↓
Store vectors + metadata in pgvector
      ↓
Mark repository READY
```

The indexer ignores noisy or unsuitable content, including dependency/build directories (`node_modules`, `dist`, `build`, `target`), Git internals, lock files, hidden files, unsupported extensions, and files over the configured size limit.

Each chunk contains metadata including:

- Repository ID
- File path
- Language
- Chunk index

Embeddings are written in batches of 32. The configured vector store uses pgvector with an HNSW index and cosine distance for fast semantic retrieval.

### 3. RAG chat flow

```text
User question
      ↓
Validate authenticated user, chat session, and repository ownership
      ↓
Semantic search in pgvector (filtered by repository ID)
      ↓
Retrieve top 8 relevant code chunks
      ↓
Build prompt: retrieved code context + user question
      ↓
LLM generates a grounded response
      ↓
Stream response tokens to the UI using SSE
      ↓
Save the response and source citations
```

The prompt instructs the model to answer only from the retrieved code context and to state uncertainty when context is insufficient. This reduces hallucinations, while citations make answers easier to verify.

## Security

- GitHub OAuth2 protects sign-in and repository authorization.
- All application API endpoints require authentication except the OAuth flow endpoints.
- Repository ownership is checked before repository indexing, chat creation, message retrieval, and response generation.
- Vector search is filtered by `repoId`, preventing retrieval from another repository’s index.
- GitHub access tokens are encrypted before database storage.
- Server session cookies are HTTP-only and use `SameSite=Lax`.
- Credentialed CORS requests are restricted through configurable allowed origins.

## Tech stack

| Layer | Tools and technologies |
| --- | --- |
| Frontend | Next.js 16, React 19, TypeScript, Tailwind CSS, TanStack Query |
| Backend | Java 21, Spring Boot 4, Spring MVC, Spring Data JPA |
| AI / Orchestration | Spring AI, OpenAI `gpt-4o-mini`, OpenAI `text-embedding-3-small` |
| RAG / Vector search | PostgreSQL, pgvector, HNSW, cosine similarity |
| Authentication | Spring Security, GitHub OAuth2 |
| Integration | GitHub REST API |
| Streaming | Server-Sent Events (SSE) |
| Build tools | Maven, npm |
| Containerized service | Docker Compose for PostgreSQL + pgvector |

## Getting started

### Prerequisites

- Java 21+
- Node.js 20+
- Docker and Docker Compose
- An OpenAI API key
- A GitHub OAuth application

### 1. Clone the repository

```bash
git clone https://github.com/Rudraprasadd/DevCode.git
cd DevCode
```

### 2. Start PostgreSQL with pgvector

From the repository root:

```bash
docker compose up -d postgres
```

The local database is exposed on port `5433` by default.

### 3. Configure backend environment variables

Create `backend/.env`:

```properties
OPENAI_API_KEY=your_openai_api_key
GITHUB_CLIENT_ID=your_github_oauth_client_id
GITHUB_CLIENT_SECRET=your_github_oauth_client_secret

FRONTEND_URL=http://localhost:3000
CORS_ALLOWED_ORIGINS=http://localhost:3000

DB_URL=jdbc:postgresql://localhost:5433/devpilot
DB_USERNAME=postgres
DB_PASSWORD=postgres

# Use strong, unique values outside local development.
TOKEN_ENCRYPTOR_PASSWORD=change_this_to_a_strong_secret
TOKEN_ENCRYPTOR_SALT=deadbeefcafebabe
```

Configure the GitHub OAuth application callback URL as:

```text
http://localhost:8080/login/oauth2/code/github
```

### 4. Start the backend

```bash
cd backend
./mvnw spring-boot:run
```

The backend runs on `http://localhost:8080` by default.

### 5. Start the frontend

In another terminal:

```bash
cd client
npm install
npm run dev
```

Open [http://localhost:3000](http://localhost:3000), sign in with GitHub, sync repositories, index one, and start asking questions.

## Key configuration

| Setting | Default | Purpose |
| --- | --- | --- |
| `app.indexing.max-file-bytes` | `102400` | Maximum file size accepted by the indexer |
| `app.indexing.chunk-size` | `800` | Configured chunking size used by the token splitter |
| Retrieval top-K | `8` | Number of similar chunks used as LLM context |
| Vector batch size | `32` | Number of chunks sent to the vector store per batch |
| Embedding dimensions | `1536` | Dimensions used by `text-embedding-3-small` |
| Chat model | `gpt-4o-mini` | Model used for answer generation |

## API overview

| Method | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/api/auth/me` | Get current authenticated user |
| `GET` | `/api/repos` | List/sync GitHub repositories |
| `POST` | `/api/repos/{id}/index` | Start asynchronous repository indexing |
| `GET` | `/api/repos/{id}/status` | Read indexing progress/status |
| `POST` | `/api/chat/sessions` | Create a repository chat session |
| `GET` | `/api/chat/sessions?repositoryId={id}` | List repository chat sessions |
| `GET` | `/api/chat/sessions/{id}` | Read session messages |
| `POST` | `/api/chat/sessions/{id}/messages` | Send a question and receive an SSE stream |

## Future improvements

- Syntax-aware chunking by class, method, and function with exact line metadata.
- Hybrid retrieval (keyword + semantic search) and reranking.
- Retrieval confidence thresholds and automated RAG evaluations.
- Incremental GitHub indexing instead of full re-indexing.
- Secret scanning before code is embedded.
- CSRF protection and secure cookies for production HTTPS deployment.
- Per-user quotas, rate limits, retries, and observability for model/API usage.

## License

This project is intended for learning and portfolio use. Add a license file before distributing or accepting contributions.
