# Enable Ollama Embeddings for GraphRAG

## Problem

GraphRAG is designed to work with OpenAI-compatible APIs for both language models and embeddings and Ollama currently has their own way of doing embeddings.

## Solution: Embeddings Proxy

To bridge this gap, let's use an embeddings proxy. This proxy acts as a middleware between GraphRAG and Ollama, translating Ollama's embedding responses into a format that GraphRAG expects.

## Design
### Class Diagram
```mermaid
classDiagram
    FastAPI <|-- EmbeddingRequest
    FastAPI <|-- EmbeddingResponse
    class EmbeddingRequest {
        +Union~str, List[str]~ input
        +str model
    }
    class EmbeddingResponse {
        +str object
        +List~dict~ data
        +str model
        +dict usage
    }
    class EmbeddingProxyApp {
        +create_embedding(request: EmbeddingRequest) EmbeddingResponse
    }
```
### Sequence Diagram
```mermaid
sequenceDiagram
    participant User
    participant FastAPI
    participant create_embedding
    participant OllamaAPI

    User->>FastAPI: POST /v1/embeddings
    FastAPI->>create_embedding: create_embedding(request)
    create_embedding->>create_embedding: Check if input is str or List
    alt Single str input
        create_embedding->>create_embedding: Convert str to List[str]
    end
    create_embedding->>OllamaAPI: POST /api/embeddings
    OllamaAPI-->>create_embedding: Response with Embedding
    alt Response Success
        create_embedding->>create_embedding: Append to embeddings list
    else Response Failure
        create_embedding->>FastAPI: Raise HTTPException    
    end
    create_embedding->>FastAPI: Return EmbeddingResponse
    FastAPI-->>User: EmbeddingResponse
```
### Activity Diagram
```mermaid
stateDiagram-v2
    [*] --> ServerStart
    ServerStart --> ParseArguments: Run Main
    ParseArguments --> ConfigureServer: Set Port/Host/Reload
    ConfigureServer --> WaitForRequest: Start FastAPI Server
    
    WaitForRequest --> ValidateRequest: POST /v1/embeddings
    
    ValidateRequest --> ConvertInput: Validate EmbeddingRequest
    ConvertInput --> SingleToList: If input is string
    ConvertInput --> CreateRequests: If input is list
    SingleToList --> CreateRequests
    
    CreateRequests --> ProcessEmbeddings: Create Ollama requests
    
    state ProcessEmbeddings {
        [*] --> NextRequest
        NextRequest --> CallOllama: For each request
        CallOllama --> CheckResponse: POST to Ollama API
        CheckResponse --> HandleError: Status != 200
        CheckResponse --> ProcessResponse: Status == 200
        ProcessResponse --> CreateEmbedding: Parse JSON
        CreateEmbedding --> NextRequest: Add to embeddings list
        HandleError --> RaiseHTTPException
    }
    
    ProcessEmbeddings --> PrepareResponse: All embeddings processed
    PrepareResponse --> SendResponse: Create EmbeddingResponse
    SendResponse --> WaitForRequest: Return response
    
    RaiseHTTPException --> WaitForRequest: Return error
```
