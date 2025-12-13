---
title: "MCP Authentication in Python with FastAPI
date: 2025-12-13 12:00:00 +0400
categories: [MCP, Python]
tags: [mcp, fastapi, python, authentication, ai, api]
description: Learn how to build a secure MCP server using FastAPI with token-based authentication to enable AI agents to interact with your applications safely.
author: aadhil
image:
  path: "https://cdn.analyticsvidhya.com/wp-content/uploads/2025/05/FastAPI-to-MCP.jpg"
  alt: "FastAPI MCP"
math: false
mermaid: true
render_with_liquid: false
---

## Introduction to MCP and Authentication

The Model Context Protocol (MCP) is an emerging standard that enables AI agents to interact with applications through a secure, context aware interface. Think of it as an API tailored for AI allowing seamless automation without custom integrations. Major players like Anthropic, OpenAI, and Google are adopting it to future proof AI-agent communications.

In this simple guide, we'll build a basic MCP server using FastAPI and add token based authentication to protect it. This keeps your endpoints secure while letting authorized AI agents access them. We'll skip complex OAuth setups for brevity and focus on a straightforward bearer token approach.

By the end, you'll have a running MCP server at `/mcp` that requires a valid token.

## Prerequisites

- Python 3.8+ with pip
- Basic familiarity with FastAPI
- Install the required packages:

```bash
pip install uvicorn fastapi fastapi-mcp python-jose[cryptography] python-multipart
```

We'll use python-jose for simple token handling (in a real app, pair it with JWT libraries).

> Optional: Create a `.env` file to store your secret token key.
{: .prompt-tip }

## Step-by-Step Setup

### Step 1: Create a Basic FastAPI App

Start with a simple app. Create `main.py`:

```python
from fastapi import FastAPI

app = FastAPI(title="Simple MCP Server")

@app.get("/")
async def root():
    return {"message": "Welcome to MCP with Authentication!"}
```

Run it:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

Visit `http://localhost:8000` to confirm it's working.

### Step 2: Integrate MCP with FastAPI

Add the MCP server using the fastapi-mcp library. Update `main.py`:

```python
from fastapi import FastAPI
from fastapi_mcp import FastApiMCP

app = FastAPI(title="Simple MCP Server")

# Your existing routes here

mcp = FastApiMCP(app, name="My Secure MCP")
mcp.mount()  # Mounts MCP at /mcp
```

Restart the server. Now, your app's endpoints (like `/`) are exposed as MCP tools at `http://localhost:8000/mcp`. AI agents can query them via Server-Sent Events (SSE).

### Step 3: Add Token-Based Authentication

We'll use HTTP Bearer tokens for simplicity. Clients must send `Authorization: Bearer <your-token>` in requests.

#### Server-Side: Secure the MCP Mount

First, create a dependency to validate tokens. Add this to `main.py`:

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import hashlib  # For simple token hashing (use JWT in production)

security = HTTPBearer()

SECRET_TOKEN = "your-super-secret-token"  # Load from .env in production

async def verify_token(credentials: HTTPAuthorizationCredentials = Depends(security)):
    token = credentials.credentials
    if hashlib.sha256(token.encode()).hexdigest() != hashlib.sha256(SECRET_TOKEN.encode()).hexdigest():
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials",
            headers={"WWW-Authenticate": "Bearer"},
        )
    return credentials
```

Now, protect the MCP server:

```python
from fastapi_mcp import AuthConfig

mcp = FastApiMCP(
    app,
    name="My Secure MCP",
    auth_config=AuthConfig(
        dependencies=[Depends(verify_token)],  # Apply auth to all MCP requests
    ),
)
mcp.mount()
```

#### Client-Side: Testing with a Simple MCP Client

To test, use a tool like curl or an MCP client. For example, with curl:

```bash
curl -H "Authorization: Bearer your-super-secret-token" http://localhost:8000/mcp
```

In a real MCP client config (e.g., JSON for an AI agent):

```json
{
  "mcpServers": {
    "local-mcp": {
      "url": "http://localhost:8000/mcp",
      "headers": {
        "Authorization": "Bearer your-super-secret-token"
      }
    }
  }
}
```

If the token is wrong, you'll get a 401 error. Success? Your root endpoint is now queryable via MCP!

## Key Components Explained

- **FastAPI-MCP**: Handles the MCP protocol, auto exposing your routes as AI friendly tools. It supports SSE for real-time streaming.
- **HTTPBearer**: Extracts tokens from headers securely.
- **AuthConfig**: Applies dependencies (like our verifier) to the entire MCP mount point.

> **Security Note:** This is a basic hash check—upgrade to JWT for expiration, scopes, and revocation. Always use environment variables for secrets.
{: .prompt-warning }

No databases or external services needed here, keeping it lightweight.

## Conclusion

You've now built a simple, authenticated MCP server in under 100 lines of code! This setup secures your FastAPI app for AI agents while following MCP standards. For production, add JWT/OAuth (see the fastapi mcp docs for examples) and error logging.

MCP is poised to revolutionize AI integrations—start experimenting today. Questions? Check the MCP spec or contribute to open-source tools like fastapi-mcp.

Happy coding! 🚀
