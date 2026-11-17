# Document Q&A Pipeline

A small document-question-answering service built with Fastify, LangChain, and LangGraph. It accepts a PDF and a natural-language question, sends both to a vision-capable model through OpenRouter, and returns the model's answer.

## How It Works

1. The Fastify server accepts a multipart `POST /chat` request containing a PDF and a question.
2. The server validates the upload, limits the file to 10 MB, and encodes its contents as Base64.
3. The LangGraph workflow passes the question and document to its `answerGeneration` node.
4. The node builds a multimodal prompt and calls the configured model through the OpenAI-compatible OpenRouter API.
5. The endpoint returns the uploaded filename, question, and generated answer as JSON.

The graph currently has a single node and a linear `START -> answerGeneration -> END` flow. The included sample document is *A Comprehensive Overview of Large Language Models* (arXiv: [2307.06435](https://arxiv.org/abs/2307.06435)).

## Requirements

- Node.js `>=24.10.0` and pnpm `12.14.0` as specified by `package.json`.
- An OpenRouter API key with access to a model capable of processing the supplied PDF input.

Use Node.js 24 with NVM (`nvm use 24`) for both direct execution and the LangGraph CLI.

## Setup

```bash
pnpm install
cp .env.example .env
```

Set `OPENROUTER_API_KEY` in `.env`. LangSmith tracing is optional; configure `LANGSMITH_API_KEY`, `LANGCHAIN_TRACING_V2`, and `LANGCHAIN_PROJECT` there if you want to send traces to LangSmith.

## Run

Start the server:

```bash
pnpm start
```

The server listens on `http://localhost:4000`. On startup, `src/index.ts` also submits the bundled PDF to `/chat` with a sample prompt and prints the answer. This demonstration requires the API key and a successful OpenRouter request; a failure is logged without preventing the server from starting.

For watch mode with the Node inspector:

```bash
pnpm dev
```

To run the graph using the LangGraph development server:

```bash
pnpm langgraph:serve
```

`langgraph.json` registers the `doc_analysis` entry point at `src/graph/factory.ts` under the export name `graph`. That export currently contains an object with both the compiled graph and the model client; verify this shape against the LangGraph CLI before relying on the CLI development server.

## HTTP API

### `POST /chat`

Send `multipart/form-data` with these fields:

| Field | Type | Requirements |
| --- | --- | --- |
| `question` | Text | Required; at least 3 characters |
| `file` | File | Required; PDF (`application/pdf`); up to 10 MB |

Example (send the question before the file):

```bash
curl -X POST http://localhost:4000/chat \
	-F 'question=Summarize the main ideas in this paper' \
	-F 'file=@docs/a-comprehensive-overview-of-large-language-models.pdf;type=application/pdf'
```

Successful response:

```json
{
	"filename": "a-comprehensive-overview-of-large-language-models.pdf",
	"question": "Summarize the main ideas in this paper",
	"answer": "..."
}
```

Invalid or missing uploads and invalid questions return a `400` response. Unexpected request or server errors return `500`. Model-generation failures are caught by the graph node and included in its generated message, so they may be returned in the `answer` field rather than as an HTTP error.

## Configuration

The model, temperature, and OpenRouter provider routing are configured in `src/config.ts`. The current model is `google/gemini-2.5-flash-lite-preview-09-2025`; alternative vision-capable models are listed there as examples. OpenRouter is accessed through LangChain's `ChatOpenAI` integration using `https://openrouter.ai/api/v1`.

`langgraph.json` names the graph `doc_analysis`, points to its compiled graph export, and tells the LangGraph CLI to load environment variables from `.env`. `tsconfig.json` enables strict TypeScript checking, targets ES2022, and uses no-emit mode.

## Project Structure

```text
docs/                 Bundled sample research paper (PDF)
src/
	config.ts            OpenRouter model and request configuration
	index.ts             Application entry point and sample request
	server.ts            Fastify server and POST /chat route
	graph/
		factory.ts         Builds the graph and its model service
		graph.ts           Graph state schema and workflow definition
		nodes/
			answerGenerationNode.ts  Generates document-grounded answers
	services/
		openrouterService.ts       Multimodal OpenRouter client
langgraph.json         LangGraph CLI graph registration
package.json           Scripts, dependencies, and Node.js requirement
tsconfig.json          TypeScript compiler options
```

## Tests

`package.json` defines `pnpm test` and `pnpm test:dev` for Node's test runner. There is no `tests/` directory in the current project tree, so no project test files are currently included.