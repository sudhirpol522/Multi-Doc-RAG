# MultiDocChat

A production-ready multi-document RAG system built with FastAPI and LangChain. Upload multiple documents (PDF, DOCX, TXT), ask questions, and get accurate answers using retrieval-augmented generation with conversational context awareness.

## Features

Multi-document ingestion with intelligent text chunking and vector embeddings using FAISS for fast semantic search. Conversational RAG with chat history and MMR retrieval for diverse relevant results. LLM-as-a-Judge evaluation framework with custom correctness evaluators using Gemini 2.5 Pro. Comprehensive unit and integration tests with pytest. Automated CI/CD pipeline deploying to AWS ECS Fargate via GitHub Actions.

## Technology Stack

Backend: FastAPI, LangChain with LCEL, Python 3.12
Vector Database: FAISS
Embeddings: Google Generative AI text-embedding-004
LLMs: Google Gemini 2.0 Flash, Groq
Testing: Pytest with unit and integration tests
Evaluation: LangSmith with custom LLM judges
DevOps: Docker, GitHub Actions, AWS ECS Fargate

## Prerequisites

Python 3.12 or higher
Google API Key for embeddings and LLM
Groq API Key for alternative LLM provider
LangSmith API Key for evaluations (optional)

## Installation

Clone the repository and navigate to the project directory.

Create a virtual environment:

```bash
python -m venv .venv
source .venv/bin/activate
```

On Windows use:

```bash
.venv\Scripts\activate
```

Install dependencies:

```bash
pip install -r requirements.txt
```

## Configuration

Create a .env file in the project root with the following variables:

```
GOOGLE_API_KEY=your_google_api_key
GROQ_API_KEY=your_groq_api_key
LLM_PROVIDER=google
ENV=local
PORT=8000
LANGSMITH_API_KEY=your_langsmith_key
```

Required variables:

GOOGLE_API_KEY - For embeddings and Gemini LLM. Get it from https://makersuite.google.com/app/apikey

GROQ_API_KEY - For Groq LLM service. Get it from https://console.groq.com/keys

Optional variables:

LLM_PROVIDER - Choose between google or groq. Default is google.

ENV - Set to local for development or production for deployment. Default is local.

PORT - Server port number. Default is 8000.

LANGSMITH_API_KEY - Required only for running evaluations. Get it from https://smith.langchain.com/settings

## Usage

Start the server:

```bash
uvicorn main:app --reload
```

Open your browser and navigate to http://localhost:8000

Upload documents using the drag and drop interface or choose files button. Supported formats are PDF, DOCX, and TXT.

After indexing completes, ask questions about your documents in the chat interface.

## API Endpoints

GET / - Serves the web interface

GET /health - Health check endpoint

POST /upload - Upload documents for indexing. Accepts multipart form data with files. Returns session ID and indexing status.

POST /chat - Send chat messages. Requires JSON body with session_id and message. Returns answer based on document context.

## How It Works

Document Upload: Files are saved to data/session_id directory, split into chunks using RecursiveCharacterTextSplitter with configurable size and overlap, embedded using Google text-embedding-004, and stored in a FAISS index at faiss_index/session_id.

Retrieval: User questions are processed through a conversational chain that rewrites queries considering chat history, retrieves relevant document chunks using MMR search for diversity, and generates answers grounded in the retrieved context.

Session Management: Each upload creates a unique session with isolated document storage. Chat history is maintained in memory per session. Browser stores session ID in localStorage for continuity.

## Testing

Run all tests:

```bash
pytest
```

Run with coverage report:

```bash
pytest --cov=multi_doc_chat
```

Run specific test categories:

```bash
pytest tests/unit
pytest tests/integration
```

The test suite includes unit tests for document ingestion, chunking, FAISS index management, and RAG retrieval logic. Integration tests cover API endpoints including upload, chat, error handling, and session validation.

## Evaluations

Run LangSmith evaluations on the RAG system:

```bash
python run_evaluations.py
```

Run with specific evaluator:

```bash
python run_evaluations.py --evaluator correctness
```

Run with custom parameters:

```bash
python run_evaluations.py --evaluator correctness --chunk-size 500 --k 10
```

Available evaluators:

correctness - Custom LLM-as-a-Judge using Gemini 2.5 Pro for semantic alignment scoring

cot_qa - Chain-of-Thought QA evaluator for reasoning quality assessment

all - Run all available evaluators

Evaluation requires LANGSMITH_API_KEY and GOOGLE_API_KEY in your .env file.

## Docker Deployment

Build the Docker image:

```bash
docker build -t multi-doc-chat .
```

Run the container:

```bash
docker run -p 8000:8080 --env-file .env multi-doc-chat
```

## CI/CD Pipeline

The project includes automated CI/CD workflows using GitHub Actions.

Continuous Integration: Runs on every push and pull request to main branch. Sets up Python 3.12 and UV package manager. Installs dependencies with locked versions. Executes pytest test suite with dummy API keys.

Continuous Deployment: Triggers after successful CI on main branch. Builds Docker image and pushes to Amazon ECR. Deploys to AWS ECS Fargate with zero-downtime rolling updates. Uses OIDC for secure AWS authentication.

Required GitHub secrets: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, and API keys should be stored in AWS Secrets Manager.

## Project Structure

```
multi_doc_chat/
  config/ - YAML configuration files
  src/
    document_ingestion/ - File upload and indexing logic
    document_chat/ - RAG retrieval and conversation handling
  utils/ - Model loader, file operations, document processing
  prompts/ - Prompt templates and registry
  logger/ - Structured logging
  exception/ - Custom exception classes
tests/
  unit/ - Component-level tests
  integration/ - API endpoint tests
.github/workflows/ - CI/CD pipeline definitions
static/ - Frontend assets
templates/ - HTML templates
```

## Architecture

The system uses a session-based architecture where each document upload creates an isolated namespace. Documents are processed through a pipeline of text extraction, recursive chunking, embedding generation, and vector storage. Queries are handled by an LCEL-based conversational chain that contextualizes questions using chat history, retrieves relevant chunks with MMR for diversity, and generates grounded responses using the configured LLM.

## Configuration Files

config.yaml controls embedding models, retrieval settings including search type and MMR parameters, and LLM configurations for multiple providers with temperature and token limits.

## Logging and Monitoring

The application uses structured logging with contextual information including session IDs, operation timings, and error details. Logs are output in JSON format for easy parsing and monitoring.

## Limitations and Future Enhancements

Current chat history is in-memory and resets on server restart. For production, implement persistent storage with PostgreSQL or Redis. Add user authentication and authorization. Implement rate limiting and quota management. Support for more document formats including images and tables. Add streaming responses for better user experience. Implement query expansion and reranking techniques.

## License

This project is provided as-is for educational and reference purposes.

## Support

For issues and questions, please open an issue in the repository.
