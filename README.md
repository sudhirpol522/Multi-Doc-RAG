# MultiDocChat
<img width="1917" height="873" alt="image" src="https://github.com/user-attachments/assets/a2b2a09f-00e4-4446-85a3-ca1fad838406" />


A production-ready multi-document RAG system built with FastAPI and LangChain. Upload multiple documents (PDF, DOCX, TXT), ask questions, and get accurate answers using retrieval-augmented generation with conversational context awareness.

## Features

Multi-document ingestion with intelligent text chunking and vector embeddings using FAISS for fast semantic search. Conversational RAG with chat history and MMR retrieval for diverse relevant results. LLM-as-a-Judge evaluation framework with custom correctness evaluators using Gemini 2.5 Pro. Comprehensive unit and integration tests with pytest. Automated CI/CD pipeline with semantic versioning deploying to Kubernetes via ArgoCD.

## Technology Stack

Backend: FastAPI, LangChain with LCEL, Python 3.12

Vector Database: FAISS

Embeddings: Google Generative AI text-embedding-004

LLMs: Google Gemini 2.0 Flash, Groq

Testing: Pytest with unit and integration tests

Evaluation: LangSmith with custom LLM judges

DevOps: Docker, GitHub Actions, Kubernetes, ArgoCD, Minikube

## Prerequisites

Python 3.12 or higher

UV package manager (recommended) - Install from https://github.com/astral-sh/uv

Google API Key for embeddings and LLM

Groq API Key for alternative LLM provider

LangSmith API Key for evaluations (optional)

## Installation

Clone the repository and navigate to the project directory.

### Option 1: Using UV (Recommended)

```bash
# Install uv if not already installed
curl -LsSf https://astral.sh/uv/install.sh | sh

# Sync dependencies (creates virtual environment automatically)
uv sync

# Run the application
uv run uvicorn main:app --reload
```

### Option 2: Using pip

Create a virtual environment:

```bash
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
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
# With uv
uv run uvicorn main:app --reload

# Or with standard Python
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
# With uv
uv run pytest

# Or with standard Python
pytest
```

Run with coverage report:

```bash
uv run pytest --cov=multi_doc_chat
```

Run specific test categories:

```bash
uv run pytest tests/unit
uv run pytest tests/integration
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

The project uses a modern GitOps workflow with semantic versioning.

### Continuous Integration

Runs on every push and pull request to main branch:
- Sets up Python 3.12 and UV package manager
- Installs dependencies with locked versions (uv.lock)
- Executes pytest test suite with dummy API keys
- Ensures code quality before deployment

### Continuous Deployment

Triggers automatically after successful CI on main branch:
- Auto-increments semantic version (1.0.0 → 1.0.1 → 1.0.2)
- Builds Docker image with version tag
- Pushes to Docker Hub (sudhirpol/multi-doc-chat:1.x.x)
- Creates Git tag for version tracking
- ArgoCD Image Updater detects new version
- Automatically updates Kubernetes manifests
- Deploys to Minikube cluster with rolling updates

### Required GitHub Secrets

- `DOCKERHUB_USERNAME`: Docker Hub username
- `DOCKERHUB_TOKEN`: Docker Hub access token

### Kubernetes Deployment

For local development and testing, the application can be deployed to Minikube with ArgoCD:

```bash
# See MINIKUBE_ARGOCD_SETUP.md for complete setup instructions
# See VERSIONING.md for versioning strategy details
# See QUICK_SETUP.md for quick reference commands
```

Key features:
- GitOps workflow with ArgoCD
- Semantic versioning (1.x.x pattern)
- Automatic image updates via ArgoCD Image Updater
- Health checks and resource limits
- ConfigMaps for configuration
- Secrets management for API keys

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
k8s/ - Kubernetes manifests (deployment, service, configmap, secrets)
argocd/ - ArgoCD application configuration
static/ - Frontend assets
templates/ - HTML templates
```

## Architecture

The system uses a session-based architecture where each document upload creates an isolated namespace. Documents are processed through a pipeline of text extraction, recursive chunking, embedding generation, and vector storage. Queries are handled by an LCEL-based conversational chain that contextualizes questions using chat history, retrieves relevant chunks with MMR for diversity, and generates grounded responses using the configured LLM.

## Configuration Files

config.yaml controls embedding models, retrieval settings including search type and MMR parameters, and LLM configurations for multiple providers with temperature and token limits.

## Logging

The application uses structured logging with contextual information including session IDs, operation timings, and error details. Logs are output in JSON format for easy parsing and monitoring.

## Limitations and Future Enhancements

Current chat history is in-memory and resets on server restart. For production, implement persistent storage with PostgreSQL or Redis. Add user authentication and authorization. Implement rate limiting and quota management. Support for more document formats including images and tables. Add streaming responses for better user experience. Implement query expansion and reranking techniques.

## License

This project is provided as-is for educational and reference purposes.

## Support

For issues and questions, please open an issue in the repository.
