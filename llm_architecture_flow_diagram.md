# LLM Architecture & Data Flow Visualization

## Complete System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                USER REQUEST FLOW                                    │
└─────────────────────────────────────────────────────────────────────────────────────┘

[User Query] → [Kiro CLI] → [FastAPI Server] → [API Gateway] → [Authentication] → [Request Router]
                                ↓
                        ┌─────────────────┐
                        │   FastAPI       │
                        │  - Async/Await  │
                        │  - Pydantic     │
                        │  - Auto Docs    │
                        └─────────────────┘
                                ↓
                        ┌─────────────────┐
                        │  Authentication │
                        │  - IAM Roles    │
                        │  - MWinit       │
                        │  - API Keys     │
                        └─────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            PROCESSING PIPELINE                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Input Text    │ → │   Tokenization  │ → │   Embeddings    │ → │  Context Prep   │
│  "How to use    │    │  ["How", "to",  │    │ [0.1, 0.8, ...] │    │ System + User   │
│   AWS Lambda?"  │    │  "use", "AWS"]  │    │ [0.3, 0.2, ...] │    │   + History     │
└─────────────────┘    └─────────────────┘    └─────────────────┘    └─────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    CODE UNDERSTANDING (CodeBERT Integration)                        │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Code Detection │ → │   CodeBERT      │ → │ Code Embeddings │ → │ Enhanced Context│
│ if input.contains│    │ Pre-trained on  │    │ Specialized for │    │ Code + Natural  │
│ ("def", "class") │    │ Code + Comments │    │ Programming     │    │ Language Mixed  │
└─────────────────┘    └─────────────────┘    └─────────────────┘    └─────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         RETRIEVAL AUGMENTED GENERATION (RAG)                        │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Query Vector   │ → │  Vector Search  │ → │ Relevant Docs   │
│ [0.1, 0.8, ...] │    │   ChromaDB      │    │ AWS Lambda docs │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         ↓                       ↓                       ↓
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Document Store  │    │ Cosine Distance │    │ Top-K Results   │
│   ChromaDB      │    │ Vector Angles   │    │   (K=5-10)      │
│ ┌─────────────┐ │    │     cos(θ)      │    │ Ranked by       │
│ │Doc1: [vec1] │ │    │ = A·B/(|A||B|)  │    │ Similarity      │
│ │Doc2: [vec2] │ │    └─────────────────┘    └─────────────────┘
│ │Doc3: [vec3] │ │
│ └─────────────┘ │
└─────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              LLM PROCESSING                                         │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Context Window  │ → │ PyTorch         │ → │ Response Gen    │
│ System Prompt + │    │ Transformer     │    │ Token by Token  │
│ Retrieved Docs +│    │ Architecture    │    │                 │
│ User Query      │    │                 │    │ ┌─────────────┐ │
└─────────────────┘    │ ┌─────────────┐ │    │ │ Sampling    │ │
                       │ │Self-Attention│ │    │ │ Top-p/Top-k │ │
                       │ │   Layers     │ │    │ │ Temperature │ │
                       │ │              │ │    │ └─────────────┘ │
                       │ │ torch.nn.    │ │    └─────────────────┘
                       │ │ MultiheadAtt │ │
                       │ └─────────────┘ │
                       │ ┌─────────────┐ │
                       │ │Feed Forward │ │
                       │ │  Networks   │ │
                       │ │ torch.nn.   │ │
                       │ │ Linear      │ │
                       │ └─────────────┘ │
                       └─────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         PYTORCH TRAINING COMPONENTS                                 │
│                        (Used during model development)                              │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Forward Pass    │ → │ Loss Calculation│ → │ Backpropagation │ → │ Weight Updates  │
│                 │    │                 │    │                 │    │                 │
│ ┌─────────────┐ │    │ ┌─────────────┐ │    │ ┌─────────────┐ │    │ ┌─────────────┐ │
│ │ Input       │ │    │ │CrossEntropy │ │    │ │ Gradients   │ │    │ │ Optimizer   │ │
│ │ Embeddings  │ │    │ │Loss Function│ │    │ │ ∂L/∂W       │ │    │ │ Adam/SGD    │ │
│ │ ↓           │ │    │ │             │ │    │ │ ∂L/∂b       │ │    │ │ W = W - α∇W │ │
│ │ Transformer │ │    │ │ L = -Σy*log │ │    │ │ Chain Rule  │ │    │ │ Learning    │ │
│ │ Layers      │ │    │ │   (softmax) │ │    │ │ Applied     │ │    │ │ Rate: α     │ │
│ │ ↓           │ │    │ └─────────────┘ │    │ └─────────────┘ │    │ └─────────────┘ │
│ │ Logits      │ │    └─────────────────┘    └─────────────────┘    └─────────────────┘
│ └─────────────┘ │
└─────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           GRADIENT COMPUTATION DETAIL                               │
└─────────────────────────────────────────────────────────────────────────────────────┘

```python
# PyTorch Automatic Differentiation
import torch
import torch.nn as nn

class TransformerBlock(nn.Module):
    def forward(self, x):
        # Forward pass - computes output
        attention_out = self.self_attention(x)
        ffn_out = self.feed_forward(attention_out)
        return ffn_out
    
    def backward(self, grad_output):
        # Automatic gradient computation
        # PyTorch builds computational graph
        # Applies chain rule automatically
        
        # ∂L/∂W_ffn = ∂L/∂output × ∂output/∂W_ffn
        # ∂L/∂W_att = ∂L/∂attention × ∂attention/∂W_att
        pass

# Training loop with gradients
optimizer = torch.optim.Adam(model.parameters(), lr=1e-4)

for batch in dataloader:
    optimizer.zero_grad()           # Clear previous gradients
    
    outputs = model(batch.input)    # Forward pass
    loss = criterion(outputs, batch.targets)  # Compute loss
    
    loss.backward()                 # Backpropagation (compute gradients)
    optimizer.step()                # Update weights using gradients
```

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            LANGCHAIN INTEGRATION                                    │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   LangChain     │ → │   Prompt        │ → │   Chain         │ → │   Memory        │
│   Framework     │    │   Templates     │    │   Execution     │    │   Management    │
│                 │    │                 │    │                 │    │                 │
│ ┌─────────────┐ │    │ ┌─────────────┐ │    │ ┌─────────────┐ │    │ ┌─────────────┐ │
│ │   Agents    │ │    │ │System Prompt│ │    │ │Sequential   │ │    │ │Conversation │ │
│ │   Tools     │ │    │ │User Input   │ │    │ │Parallel     │ │    │ │Buffer       │ │
│ │   Memory    │ │    │ │Context      │ │    │ │Conditional  │ │    │ │Summary      │ │
│ └─────────────┘ │    │ └─────────────┘ │    │ └─────────────┘ │    │ └─────────────┘ │
└─────────────────┘    └─────────────────┘    └─────────────────┘    └─────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         FASTAPI SERVICE ARCHITECTURE                                │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ HTTP Request    │ → │ FastAPI Router  │ → │ Business Logic  │ → │ Response Model  │
│                 │    │                 │    │                 │    │                 │
│ ┌─────────────┐ │    │ ┌─────────────┐ │    │ ┌─────────────┐ │    │ ┌─────────────┐ │
│ │POST /chat   │ │    │ │@app.post    │ │    │ │RAG Pipeline │ │    │ │Pydantic     │ │
│ │Headers      │ │    │ │Validation   │ │    │ │LLM Call     │ │    │ │Serialization│ │
│ │Body JSON    │ │    │ │Auth Check   │ │    │ │Tool Exec    │ │    │ │JSON Response│ │
│ └─────────────┘ │    │ └─────────────┘ │    │ └─────────────┘ │    │ └─────────────┘ │
└─────────────────┘    └─────────────────┘    └─────────────────┘    └─────────────────┘

```python
# FastAPI Integration Example
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import torch
from transformers import AutoModel, AutoTokenizer

app = FastAPI()

class ChatRequest(BaseModel):
    message: str
    context: Optional[str] = None

class ChatResponse(BaseModel):
    response: str
    tokens_used: int
    model_version: str

@app.post("/chat", response_model=ChatResponse)
async def chat_endpoint(request: ChatRequest):
    try:
        # RAG retrieval
        relevant_docs = vector_db.similarity_search(request.message)
        
        # Context assembly
        context = build_context(relevant_docs, request.message)
        
        # LLM inference (PyTorch backend)
        with torch.no_grad():
            response = model.generate(context)
        
        return ChatResponse(
            response=response,
            tokens_used=len(tokenizer.encode(response)),
            model_version="claude-v2"
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              TOOL EXECUTION                                         │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Function Call   │ → │ Tool Execution  │ → │ Result Process  │ → │ Context Update  │
│ Detection       │    │                 │    │                 │    │                 │
│                 │    │ ┌─────────────┐ │    │ ┌─────────────┐ │    │ ┌─────────────┐ │
│ JSON Structure  │    │ │AWS CLI      │ │    │ │Format       │ │    │ │Add to       │ │
│ Parameters      │    │ │File Ops     │ │    │ │Validate     │ │    │ │Conversation │ │
│ Validation      │    │ │Web Search   │ │    │ │Error Handle │ │    │ │History      │ │
└─────────────────┘    │ └─────────────┘ │    │ └─────────────┘ │    │ └─────────────┘ │
                       └─────────────────┘    └─────────────────┘    └─────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        COMPLETE TECHNOLOGY STACK                                    │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND LAYER                                         │
│  Kiro CLI → FastAPI → Uvicorn ASGI Server                                          │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            APPLICATION LAYER                                        │
│  LangChain Orchestration → RAG Pipeline → Tool Integration                         │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              MODEL LAYER                                            │
│  PyTorch Models → CodeBERT (Code) → Transformers → Bedrock API                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            INFRASTRUCTURE LAYER                                     │
│  AWS EC2/Lambda → GPU Instances → Vector Databases → Storage                       │
└─────────────────────────────────────────────────────────────────────────────────────┘

## Technology Integration Summary

### **Request Flow with All Components**
```
1. User Query → Kiro CLI
2. CLI → FastAPI Server (Uvicorn)
3. FastAPI → Authentication & Validation
4. FastAPI → LangChain Orchestration
5. LangChain → RAG Pipeline (ChromaDB)
6. RAG → Context Assembly
7. Context → CodeBERT (if code detected)
8. Enhanced Context → Bedrock/PyTorch Model
9. Model → Transformer Processing
10. Response → FastAPI → CLI → User
```

### **Training vs Inference**
```
TRAINING (Model Development):
PyTorch → Forward Pass → Loss → Backpropagation → Gradient Updates

INFERENCE (Production):
FastAPI → Context → Model → Response (No gradients computed)
```

### **Code-Specific Processing**
```
Code Input → CodeBERT Tokenization → Code Embeddings → Enhanced Understanding
```
```

## Corrections to Your Understanding

### ❌ **Misconceptions:**

1. **"B-tree for nearest neighbors"** - Actually uses **vector similarity search** (cosine distance, not B-trees)
2. **"Neurons process requests"** - Modern LLMs use **Transformer architecture**, not traditional neural networks
3. **"Nearby neighbors added"** - This happens in **RAG retrieval**, not in the model itself
4. **"Top-p in model selection"** - Top-p is used in **response generation**, not model selection

### ✅ **Correct Understanding:**

1. **Authentication flow** - Correct (IAM, MWinit)
2. **Tokenization** - Correct (text → tokens)
3. **Context embedding** - Correct (but happens in RAG, not model)
4. **Probability-based generation** - Correct (Top-p, Top-k sampling)

## Where Vector Angles Are Used

### **Vector Similarity Search (ChromaDB)**
```python
# Cosine Similarity Calculation
def cosine_similarity(vec_a, vec_b):
    dot_product = np.dot(vec_a, vec_b)
    norm_a = np.linalg.norm(vec_a)
    norm_b = np.linalg.norm(vec_b)
    return dot_product / (norm_a * norm_b)

# This gives cos(θ) where θ is angle between vectors
# cos(0°) = 1.0 (identical)
# cos(90°) = 0.0 (orthogonal/unrelated)
```

### **Embedding Space Visualization**
```
High-dimensional space (e.g., 1536 dimensions for OpenAI embeddings)

Query: "AWS Lambda"     → [0.1, 0.8, 0.3, ...]
Doc1: "Lambda functions" → [0.2, 0.7, 0.4, ...] ← Small angle (similar)
Doc2: "EC2 instances"   → [0.9, 0.1, 0.2, ...] ← Large angle (different)
```

## Actual Data Flow Steps

### **1. Input Processing**
```
"How to use AWS Lambda?" → ["How", "to", "use", "AWS", "Lambda", "?"]
```

### **2. Query Embedding**
```
Tokens → Embedding Model → Vector [0.1, 0.8, 0.3, ..., 0.5] (1536 dims)
```

### **3. Vector Search (ChromaDB)**
```python
# Find similar documents using cosine similarity
query_vector = embed("How to use AWS Lambda?")
similar_docs = chromadb.query(
    query_embeddings=[query_vector],
    n_results=5
)
```

### **4. Context Assembly**
```
System Prompt + Retrieved Documents + User Query + Conversation History
```

### **5. LLM Processing (Transformer)**
```
Input → Self-Attention → Feed-Forward → Output Logits → Sampling → Tokens
```

### **6. Response Generation**
```
Tokens → Detokenization → "To use AWS Lambda, you need to..."
```

## Key Technologies Integration

### **LangChain Role:**
- **Orchestration framework** (not the model itself)
- **Prompt management**
- **Chain execution**
- **Tool integration**
- **Memory management**

### **ChromaDB Role:**
- **Vector database** for document storage
- **Similarity search** using cosine distance
- **Metadata filtering**
- **Persistent storage**

### **Bedrock Role:**
- **Model hosting** (AWS managed LLMs)
- **API access** to various models
- **Scaling and infrastructure**
- **Security and compliance**

### **Vector Embeddings Role:**
- **Semantic representation** of text
- **Similarity comparison** via angles/distance
- **Retrieval mechanism** for relevant context
- **Knowledge base indexing**

The vector angles (cosine similarity) are crucial for finding relevant documents, but the actual LLM processing uses attention mechanisms in Transformers, not traditional nearest neighbor search with B-trees.
