# Technical Architecture Proposals
*Concrete Implementation Plans for Agentic Web Experiences*

**Date**: January 22, 2026
**Version**: 1.0
**Related**: [AGENTIC_EXPERIENCES_VISION.md](./AGENTIC_EXPERIENCES_VISION.md)

---

## Architecture Evolution: From Polling to Real-Time

### Current Architecture (Polling-Based)

```
┌─────────┐         HTTP Polling (3s)         ┌─────────┐
│         │ ──────────────────────────────────▶│         │
│ React   │                                     │ FastAPI │
│ Frontend│◀────────────────────────────────── │ Backend │
│         │   JSON Response (status, answer)   │         │
└─────────┘                                     └─────────┘
                                                     │
                                                     │ Celery Tasks
                                                     ▼
                                                ┌─────────┐
                                                │  Redis  │
                                                └─────────┘
```

**Limitations**:
- 3-second delay for updates
- Inefficient (constant HTTP requests)
- No true real-time collaboration
- High server load with many users

---

### Proposed Architecture (WebSocket-Based)

```
┌─────────┐         WebSocket (Real-time)      ┌─────────┐
│         │◀──────────────────────────────────▶│         │
│ React   │         bi-directional              │ FastAPI │
│ Frontend│         event streaming             │ Backend │
│         │                                     │ +Socket │
└─────────┘                                     └─────────┘
    │                                                │
    │ WebSocket Rooms                                │ Pub/Sub
    ▼                                                ▼
┌─────────────────────────────────────────────────────────┐
│                   Redis Pub/Sub                         │
│  Channels: agent_updates, task_progress, screenshots    │
└─────────────────────────────────────────────────────────┘
```

**Benefits**:
- Instant updates (< 100ms latency)
- 90% reduction in network traffic
- Enables real-time collaboration
- Foundation for multi-user workspaces

---

## Proposal 1: Real-Time Agent Canvas

### Feature Overview

A **visual, interactive workspace** where users see agents working in real-time, can inspect their state, and intervene when needed.

### UI Mockup (Component Structure)

```
┌────────────────────────────────────────────────────────────┐
│  🎯 AgenticSeek Canvas                            [⚙️] [👤] │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ 💬 Chat & Input                                      │  │
│  │                                                      │  │
│  │  User: "Research competitors and build comparison"  │  │
│  │  Assistant: "I'll delegate to research and coder..." │  │
│  │  [                Input field                    ] 📤 │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ 🤖 Active Agents (3)                         [Pause] │  │
│  │                                                      │  │
│  │  🌐 BrowserAgent    [████████░░] Scraping site 2/3  │  │
│  │     └─ Visited: competitor1.com, competitor2.com    │  │
│  │                                                      │  │
│  │  💻 CoderAgent      [████████░░] Writing code       │  │
│  │     └─ Files: comparison.py, data.json              │  │
│  │                                                      │  │
│  │  📋 PlannerAgent    [██████████] Planning complete   │  │
│  │     └─ Tasks: 3/3 delegated                         │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ 📦 Artifacts (5)                         [Grid View] │  │
│  │                                                      │  │
│  │  📄 comparison.py         📊 competitor_data.json   │  │
│  │  📸 screenshot_1.png      📸 screenshot_2.png       │  │
│  │  📝 summary.md                                       │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### Implementation Plan

#### Backend Changes

**File**: `api.py` (New WebSocket Endpoint)

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import Dict, Set
import json
import asyncio

app = FastAPI()

# Connection manager for WebSocket clients
class ConnectionManager:
    def __init__(self):
        self.active_connections: Dict[str, Set[WebSocket]] = {}

    async def connect(self, websocket: WebSocket, session_id: str):
        await websocket.accept()
        if session_id not in self.active_connections:
            self.active_connections[session_id] = set()
        self.active_connections[session_id].add(websocket)

    def disconnect(self, websocket: WebSocket, session_id: str):
        self.active_connections[session_id].remove(websocket)
        if not self.active_connections[session_id]:
            del self.active_connections[session_id]

    async def broadcast_to_session(self, session_id: str, message: dict):
        if session_id in self.active_connections:
            for connection in self.active_connections[session_id]:
                try:
                    await connection.send_json(message)
                except:
                    pass  # Handle disconnected clients

manager = ConnectionManager()

@app.websocket("/ws/{session_id}")
async def websocket_endpoint(websocket: WebSocket, session_id: str):
    await manager.connect(websocket, session_id)
    try:
        while True:
            # Keep connection alive, receive client messages
            data = await websocket.receive_text()
            # Handle client commands (pause, resume, etc.)
            await handle_client_command(data, session_id)
    except WebSocketDisconnect:
        manager.disconnect(websocket, session_id)

# Agent state broadcaster
async def broadcast_agent_state(session_id: str, agent_name: str, state: dict):
    """
    Called by agents to broadcast their state changes
    """
    message = {
        "type": "agent_state",
        "agent_name": agent_name,
        "state": state,
        "timestamp": time.time()
    }
    await manager.broadcast_to_session(session_id, message)
```

**File**: `sources/agents/agent.py` (Add State Broadcasting)

```python
class Agent:
    def __init__(self, name, prompt_path, provider, verbose=False, tools=None):
        # ... existing code ...
        self.session_id = None  # Set by interaction manager
        self.state_broadcaster = None  # WebSocket broadcaster

    async def update_state(self, state_update: dict):
        """
        Broadcast state changes to connected clients
        """
        if self.state_broadcaster:
            await self.state_broadcaster(
                self.session_id,
                self.name,
                {
                    "status": state_update.get("status", "working"),
                    "progress": state_update.get("progress", 0),
                    "current_task": state_update.get("current_task", ""),
                    "artifacts": state_update.get("artifacts", [])
                }
            )

    async def run(self, query: str):
        await self.update_state({"status": "starting", "progress": 0})
        # ... existing run logic ...
        await self.update_state({"status": "processing", "progress": 50})
        # ... more logic ...
        await self.update_state({"status": "complete", "progress": 100})
```

#### Frontend Changes

**File**: `frontend/agentic-seek-front/src/hooks/useWebSocket.js`

```javascript
import { useEffect, useRef, useState } from 'react';

export const useWebSocket = (sessionId) => {
  const [agentStates, setAgentStates] = useState({});
  const [artifacts, setArtifacts] = useState([]);
  const [messages, setMessages] = useState([]);
  const wsRef = useRef(null);

  useEffect(() => {
    // Connect to WebSocket
    const ws = new WebSocket(`ws://localhost:7777/ws/${sessionId}`);
    wsRef.current = ws;

    ws.onopen = () => {
      console.log('WebSocket connected');
    };

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);

      switch (data.type) {
        case 'agent_state':
          setAgentStates(prev => ({
            ...prev,
            [data.agent_name]: data.state
          }));
          break;

        case 'artifact_created':
          setArtifacts(prev => [...prev, data.artifact]);
          break;

        case 'message':
          setMessages(prev => [...prev, data.message]);
          break;

        default:
          console.log('Unknown message type:', data.type);
      }
    };

    ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };

    ws.onclose = () => {
      console.log('WebSocket disconnected');
      // Implement reconnection logic
      setTimeout(() => {
        // Reconnect
      }, 3000);
    };

    return () => {
      ws.close();
    };
  }, [sessionId]);

  const sendCommand = (command) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify(command));
    }
  };

  return {
    agentStates,
    artifacts,
    messages,
    sendCommand,
    isConnected: wsRef.current?.readyState === WebSocket.OPEN
  };
};
```

**File**: `frontend/agentic-seek-front/src/components/AgentCanvas.js`

```javascript
import React from 'react';
import { useWebSocket } from '../hooks/useWebSocket';
import AgentCard from './AgentCard';
import ArtifactGallery from './ArtifactGallery';

export const AgentCanvas = ({ sessionId }) => {
  const { agentStates, artifacts, messages, sendCommand, isConnected } =
    useWebSocket(sessionId);

  const handlePauseAgent = (agentName) => {
    sendCommand({
      type: 'pause_agent',
      agent_name: agentName
    });
  };

  return (
    <div className="agent-canvas">
      <div className="connection-status">
        {isConnected ? '🟢 Connected' : '🔴 Disconnected'}
      </div>

      <div className="chat-section">
        {messages.map((msg, idx) => (
          <div key={idx} className={`message message-${msg.type}`}>
            {msg.content}
          </div>
        ))}
      </div>

      <div className="active-agents">
        <h3>🤖 Active Agents ({Object.keys(agentStates).length})</h3>
        {Object.entries(agentStates).map(([name, state]) => (
          <AgentCard
            key={name}
            name={name}
            state={state}
            onPause={() => handlePauseAgent(name)}
          />
        ))}
      </div>

      <div className="artifacts-section">
        <h3>📦 Artifacts ({artifacts.length})</h3>
        <ArtifactGallery artifacts={artifacts} />
      </div>
    </div>
  );
};
```

---

## Proposal 2: Vector Memory Integration

### Feature Overview

Enable agents to **remember and learn from past interactions** using semantic search over conversation history.

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Agent Memory System                   │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────┐         ┌────────────┐                  │
│  │ Short-Term │         │ Long-Term  │                  │
│  │  Memory    │────────▶│   Memory   │                  │
│  │ (Current)  │ Summary │  (Vector)  │                  │
│  └────────────┘         └────────────┘                  │
│                                │                         │
│                                │ Embedding               │
│                                ▼                         │
│                         ┌─────────────┐                 │
│                         │  ChromaDB   │                 │
│                         │  (Vectors)  │                 │
│                         └─────────────┘                 │
│                                │                         │
│                                │ Semantic Search         │
│                                ▼                         │
│                         ┌─────────────┐                 │
│                         │  Retrieved  │                 │
│                         │   Context   │                 │
│                         └─────────────┘                 │
│                                │                         │
│                                ▼                         │
│                         ┌─────────────┐                 │
│                         │   Agent     │                 │
│                         │   Prompt    │                 │
│                         └─────────────┘                 │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Implementation

**File**: `sources/vector_memory.py` (New)

```python
from chromadb import Client, Settings
from chromadb.config import Settings
from sentence_transformers import SentenceTransformer
from typing import List, Dict
import uuid

class VectorMemory:
    """
    Long-term memory using vector embeddings for semantic search
    """
    def __init__(self, collection_name: str = "agent_memory"):
        # Initialize ChromaDB
        self.client = Client(Settings(
            chroma_db_impl="duckdb+parquet",
            persist_directory="./chroma_db"
        ))

        # Create or get collection
        self.collection = self.client.get_or_create_collection(
            name=collection_name,
            metadata={"description": "Agent conversation memory"}
        )

        # Load embedding model (local, ~400MB)
        self.embedder = SentenceTransformer('all-MiniLM-L6-v2')

    def add_memory(self, text: str, metadata: Dict = None):
        """
        Add a memory to the vector database
        """
        # Generate embedding
        embedding = self.embedder.encode(text).tolist()

        # Store in ChromaDB
        self.collection.add(
            embeddings=[embedding],
            documents=[text],
            metadatas=[metadata or {}],
            ids=[str(uuid.uuid4())]
        )

    def search(self, query: str, n_results: int = 5) -> List[Dict]:
        """
        Semantic search over memories
        """
        # Generate query embedding
        query_embedding = self.embedder.encode(query).tolist()

        # Search
        results = self.collection.query(
            query_embeddings=[query_embedding],
            n_results=n_results
        )

        # Format results
        memories = []
        for i, doc in enumerate(results['documents'][0]):
            memories.append({
                'text': doc,
                'metadata': results['metadatas'][0][i],
                'distance': results['distances'][0][i]
            })

        return memories

    def get_relevant_context(self, query: str, max_tokens: int = 1000) -> str:
        """
        Get relevant context for a query, respecting token limits
        """
        memories = self.search(query, n_results=10)

        context = "Relevant past experiences:\n\n"
        total_tokens = 0

        for memory in memories:
            # Rough token estimation (4 chars ≈ 1 token)
            tokens = len(memory['text']) // 4
            if total_tokens + tokens > max_tokens:
                break

            context += f"- {memory['text']}\n"
            total_tokens += tokens

        return context
```

**File**: `sources/memory.py` (Update Existing)

```python
from sources.vector_memory import VectorMemory

class Memory:
    def __init__(
        self,
        system_prompt: str,
        recover_last_session: bool = False,
        memory_compression: bool = True,
        model_provider: str = "unknown",
        use_vector_memory: bool = True  # NEW
    ):
        # ... existing code ...

        # Add vector memory
        if use_vector_memory:
            self.vector_memory = VectorMemory(
                collection_name=f"agent_memory_{model_provider}"
            )
        else:
            self.vector_memory = None

    def add_message(self, role: str, content: str):
        """
        Add message to both short-term and long-term memory
        """
        # Existing short-term memory
        self.messages.append({"role": role, "content": content})

        # Add to vector memory for long-term storage
        if self.vector_memory:
            self.vector_memory.add_memory(
                text=content,
                metadata={
                    "role": role,
                    "timestamp": time.time(),
                    "model": self.model_provider
                }
            )

    def get_context_augmented_prompt(self, current_query: str) -> str:
        """
        Augment prompt with relevant past context
        """
        if not self.vector_memory:
            return current_query

        # Retrieve relevant context
        context = self.vector_memory.get_relevant_context(
            query=current_query,
            max_tokens=1000
        )

        # Augment prompt
        augmented = f"{context}\n\nCurrent query: {current_query}"
        return augmented
```

**Usage Example**:

```python
# In agent initialization
agent = BrowserAgent(
    name="Friday",
    prompt_path="prompts/base/browser_agent.txt",
    provider=llm_provider,
    verbose=True,
    browser=browser
)

# When processing query
augmented_query = agent.memory.get_context_augmented_prompt(user_query)
response = await agent.process(augmented_query)

# Example output:
# Relevant past experiences:
# - Last week, you searched for restaurants in Paris and found Le Comptoir
# - You prefer vegetarian options based on previous searches
#
# Current query: Find me a good restaurant in Lyon
```

**Benefits**:
- Agents remember user preferences
- Reduce repeated work
- Personalized experiences
- Foundation for learning

---

## Proposal 3: Multi-Agent Parallel Execution

### Feature Overview

Run multiple agents **simultaneously** on different aspects of a task, dramatically reducing completion time.

### Current vs. Proposed Flow

**Current (Sequential)**:
```
PlannerAgent creates plan
  ↓
BrowserAgent executes (30s)
  ↓
CoderAgent executes (20s)
  ↓
FileAgent executes (10s)
  ↓
Total: ~60 seconds
```

**Proposed (Parallel)**:
```
PlannerAgent creates plan
  ├─▶ BrowserAgent (30s) ──┐
  ├─▶ CoderAgent (20s) ────┤→ Merge results
  └─▶ FileAgent (10s) ─────┘
Total: ~30 seconds (50% faster)
```

### Implementation

**File**: `sources/agents/planner_agent.py` (Update)

```python
import asyncio
from typing import List, Dict

class PlannerAgent(Agent):
    # ... existing code ...

    async def execute_plan_parallel(
        self,
        agents_tasks: List[Dict]
    ) -> Dict[int, str]:
        """
        Execute agent tasks in parallel where possible
        """
        # Analyze task dependencies
        task_graph = self.build_dependency_graph(agents_tasks)

        # Execute in topological order (respecting dependencies)
        results = {}
        for level in task_graph:
            # Tasks in same level have no dependencies, run in parallel
            level_tasks = [
                self.execute_single_task(task, results)
                for task in level
            ]

            # Wait for all tasks in this level to complete
            level_results = await asyncio.gather(*level_tasks)

            # Merge results
            for task, result in zip(level, level_results):
                results[task['id']] = result

        return results

    def build_dependency_graph(self, tasks: List[Dict]) -> List[List[Dict]]:
        """
        Build a dependency graph from tasks
        Returns list of levels, where each level can be executed in parallel
        """
        # Simple implementation: group by 'need' field
        levels = []
        remaining_tasks = tasks.copy()
        completed_ids = set()

        while remaining_tasks:
            # Find tasks with no unmet dependencies
            current_level = []
            for task in remaining_tasks:
                dependencies = task.get('need', [])
                if all(dep in completed_ids for dep in dependencies):
                    current_level.append(task)

            if not current_level:
                # Circular dependency or invalid graph
                # Fall back to sequential
                return [[task] for task in remaining_tasks]

            levels.append(current_level)

            # Mark as completed
            for task in current_level:
                completed_ids.add(task['id'])
                remaining_tasks.remove(task)

        return levels

    async def execute_single_task(
        self,
        task: Dict,
        previous_results: Dict[int, str]
    ) -> str:
        """
        Execute a single agent task
        """
        agent_name = task['agent'].lower()
        agent = self.agents[agent_name]

        # Prepare prompt with dependencies
        prompt = self.make_prompt(
            task=task['task'],
            agent_infos_dict={
                dep_id: previous_results[dep_id]
                for dep_id in task.get('need', [])
                if dep_id in previous_results
            }
        )

        # Execute agent
        result = await agent.run(prompt)
        return result
```

**Example Plan with Dependencies**:

```json
{
  "plan": [
    {
      "id": 1,
      "agent": "web",
      "task": "Search for competitor pricing",
      "need": []
    },
    {
      "id": 2,
      "agent": "file",
      "task": "Load our pricing spreadsheet",
      "need": []
    },
    {
      "id": 3,
      "agent": "coder",
      "task": "Create comparison chart",
      "need": [1, 2]
    }
  ]
}
```

**Execution**:
- **Level 1** (parallel): Task 1 and Task 2 run simultaneously
- **Level 2** (after level 1): Task 3 runs with results from 1 and 2

---

## Proposal 4: Tool Marketplace Architecture

### Feature Overview

Enable **community-contributed tools** that extend agent capabilities.

### Architecture

```
┌────────────────────────────────────────────────────────────┐
│                     Tool Marketplace                        │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────┐         ┌─────────────────────┐  │
│  │   Tool Registry     │         │   Tool Sandbox      │  │
│  │   (PostgreSQL)      │         │   (Docker)          │  │
│  │                     │         │                     │  │
│  │  - Tool metadata    │         │  - Isolated exec    │  │
│  │  - Versions         │         │  - Resource limits  │  │
│  │  - Ratings          │         │  - Security checks  │  │
│  └─────────────────────┘         └─────────────────────┘  │
│           │                                │                │
│           │                                │                │
│           ▼                                ▼                │
│  ┌────────────────────────────────────────────────────┐   │
│  │            Tool Discovery API                       │   │
│  │  GET /api/tools?category=finance                    │   │
│  │  POST /api/tools/install                            │   │
│  │  GET /api/tools/:id/usage                           │   │
│  └────────────────────────────────────────────────────┘   │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### Tool Definition Format

**File**: `tools/community/stock_price_tool.yaml`

```yaml
name: stock_price_fetcher
version: 1.0.0
author: john_doe
description: Fetches real-time stock prices from Yahoo Finance
category: finance
tags: [stocks, finance, api]

dependencies:
  - yfinance==0.2.28

permissions:
  network: true
  filesystem: false

parameters:
  - name: ticker
    type: string
    required: true
    description: Stock ticker symbol (e.g., AAPL, TSLA)

  - name: period
    type: string
    required: false
    default: "1d"
    description: Time period (1d, 5d, 1mo, 1y)

returns:
  type: object
  schema:
    price: number
    change: number
    volume: number

code_file: stock_price_tool.py
```

**File**: `tools/community/stock_price_tool.py`

```python
import yfinance as yf
from typing import Dict

def execute(ticker: str, period: str = "1d") -> Dict:
    """
    Fetch stock price data
    """
    try:
        stock = yf.Ticker(ticker)
        hist = stock.history(period=period)

        if hist.empty:
            return {"error": f"No data found for ticker {ticker}"}

        current_price = hist['Close'].iloc[-1]
        prev_price = hist['Close'].iloc[0] if len(hist) > 1 else current_price
        change = ((current_price - prev_price) / prev_price) * 100

        return {
            "ticker": ticker,
            "price": float(current_price),
            "change_percent": float(change),
            "volume": int(hist['Volume'].iloc[-1])
        }
    except Exception as e:
        return {"error": str(e)}
```

### Tool Installation Flow

```python
# File: sources/tool_manager.py (New)

import yaml
import docker
import hashlib
from pathlib import Path
from typing import Dict, List

class ToolManager:
    """
    Manages community tools
    """
    def __init__(self, tools_directory: str = "./tools/community"):
        self.tools_dir = Path(tools_directory)
        self.tools_dir.mkdir(parents=True, exist_ok=True)
        self.docker_client = docker.from_env()
        self.installed_tools = {}

    def install_tool(self, tool_id: str) -> bool:
        """
        Install a tool from the marketplace
        """
        # 1. Download tool package
        tool_package = self.download_tool(tool_id)

        # 2. Verify integrity (hash check)
        if not self.verify_tool(tool_package):
            raise SecurityError("Tool integrity check failed")

        # 3. Scan for security issues
        if not self.security_scan(tool_package):
            raise SecurityError("Security scan failed")

        # 4. Extract to tools directory
        tool_path = self.tools_dir / tool_id
        self.extract_tool(tool_package, tool_path)

        # 5. Load metadata
        metadata = self.load_tool_metadata(tool_path)

        # 6. Build Docker image if needed
        if metadata.get('permissions', {}).get('network'):
            self.build_sandbox_image(tool_id, tool_path)

        # 7. Register tool
        self.installed_tools[tool_id] = {
            'path': tool_path,
            'metadata': metadata,
            'sandboxed': bool(metadata.get('permissions'))
        }

        return True

    async def execute_tool(
        self,
        tool_id: str,
        parameters: Dict
    ) -> Dict:
        """
        Execute a community tool in sandbox
        """
        if tool_id not in self.installed_tools:
            raise ValueError(f"Tool {tool_id} not installed")

        tool_info = self.installed_tools[tool_id]

        if tool_info['sandboxed']:
            # Execute in Docker sandbox
            result = await self.execute_in_sandbox(
                tool_id=tool_id,
                parameters=parameters,
                timeout=30
            )
        else:
            # Execute directly (trusted tools only)
            result = await self.execute_direct(
                tool_id=tool_id,
                parameters=parameters
            )

        return result

    async def execute_in_sandbox(
        self,
        tool_id: str,
        parameters: Dict,
        timeout: int = 30
    ) -> Dict:
        """
        Execute tool in isolated Docker container
        """
        container = self.docker_client.containers.run(
            image=f"agentic-tool-{tool_id}",
            command=json.dumps(parameters),
            detach=True,
            mem_limit="512m",  # Resource limits
            cpu_quota=50000,
            network_mode="bridge",
            remove=True
        )

        try:
            # Wait for execution with timeout
            result = container.wait(timeout=timeout)
            logs = container.logs().decode()

            return json.loads(logs)
        except docker.errors.ContainerError as e:
            return {"error": f"Tool execution failed: {str(e)}"}
```

**Integration with Agents**:

```python
# In agent code
from sources.tool_manager import ToolManager

class BrowserAgent(Agent):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.tool_manager = ToolManager()

    async def process(self, query: str):
        # Check if a community tool can help
        if "stock price" in query.lower():
            # Use community tool
            result = await self.tool_manager.execute_tool(
                tool_id="stock_price_fetcher",
                parameters={"ticker": "AAPL", "period": "1d"}
            )
            return f"AAPL stock price: ${result['price']}"

        # Fall back to regular processing
        return await super().process(query)
```

---

## Proposal 5: Mobile-First Agent Experience

### Feature Overview

Bring agentic experiences to mobile with **voice-first interactions** and **mobile-optimized UI**.

### React Native Architecture

```
┌────────────────────────────────────────────────────────────┐
│                   React Native App                          │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │               Voice Interface                        │  │
│  │                                                      │  │
│  │   🎤 [Hold to speak]                                │  │
│  │                                                      │  │
│  │   🔊 Agent speaking...                               │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │             Active Agents                           │  │
│  │                                                      │  │
│  │   🌐 BrowserAgent                                    │  │
│  │      Researching competitors...                      │  │
│  │      [View Progress]                                 │  │
│  │                                                      │  │
│  │   💻 CoderAgent                                      │  │
│  │      Writing comparison script...                    │  │
│  │      [View Code]                                     │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              Recent Tasks                            │  │
│  │                                                      │  │
│  │   ✅ Research competitors                            │  │
│  │   ✅ Build comparison                                │  │
│  │   ⏳ Generate report                                 │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### Key Components

**File**: `mobile-app/src/hooks/useVoiceRecognition.ts`

```typescript
import { useState, useEffect } from 'react';
import Voice from '@react-native-voice/voice';

export const useVoiceRecognition = () => {
  const [isListening, setIsListening] = useState(false);
  const [transcript, setTranscript] = useState('');
  const [error, setError] = useState(null);

  useEffect(() => {
    Voice.onSpeechStart = () => setIsListening(true);
    Voice.onSpeechEnd = () => setIsListening(false);
    Voice.onSpeechResults = (e) => {
      setTranscript(e.value[0]);
    };
    Voice.onSpeechError = (e) => {
      setError(e.error);
    };

    return () => {
      Voice.destroy().then(Voice.removeAllListeners);
    };
  }, []);

  const startListening = async () => {
    try {
      await Voice.start('en-US');
    } catch (e) {
      setError(e);
    }
  };

  const stopListening = async () => {
    try {
      await Voice.stop();
    } catch (e) {
      setError(e);
    }
  };

  return {
    isListening,
    transcript,
    error,
    startListening,
    stopListening
  };
};
```

**File**: `mobile-app/src/screens/VoiceAgentScreen.tsx`

```typescript
import React, { useState } from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';
import { useVoiceRecognition } from '../hooks/useVoiceRecognition';
import { useWebSocket } from '../hooks/useWebSocket';

export const VoiceAgentScreen = () => {
  const { isListening, transcript, startListening, stopListening } =
    useVoiceRecognition();
  const { agentStates, sendCommand } = useWebSocket();

  const handleVoiceCommand = async () => {
    if (isListening) {
      await stopListening();
      // Send transcript to backend
      sendCommand({
        type: 'query',
        query: transcript
      });
    } else {
      await startListening();
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Voice Assistant</Text>

      <TouchableOpacity
        style={[
          styles.micButton,
          isListening && styles.micButtonActive
        ]}
        onPress={handleVoiceCommand}
      >
        <Text style={styles.micIcon}>
          {isListening ? '🎤' : '🔴'}
        </Text>
      </TouchableOpacity>

      {transcript && (
        <View style={styles.transcriptBox}>
          <Text>{transcript}</Text>
        </View>
      )}

      <View style={styles.agentsContainer}>
        {Object.entries(agentStates).map(([name, state]) => (
          <View key={name} style={styles.agentCard}>
            <Text style={styles.agentName}>{name}</Text>
            <Text style={styles.agentStatus}>{state.status}</Text>
          </View>
        ))}
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 20,
    backgroundColor: '#1a1a1a'
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    color: '#fff',
    marginBottom: 20
  },
  micButton: {
    width: 100,
    height: 100,
    borderRadius: 50,
    backgroundColor: '#333',
    justifyContent: 'center',
    alignItems: 'center',
    alignSelf: 'center',
    marginVertical: 20
  },
  micButtonActive: {
    backgroundColor: '#e63946'
  },
  micIcon: {
    fontSize: 40
  },
  transcriptBox: {
    padding: 15,
    backgroundColor: '#2a2a2a',
    borderRadius: 10,
    marginVertical: 10
  },
  agentsContainer: {
    marginTop: 20
  },
  agentCard: {
    padding: 15,
    backgroundColor: '#2a2a2a',
    borderRadius: 10,
    marginBottom: 10
  },
  agentName: {
    fontSize: 18,
    fontWeight: 'bold',
    color: '#fff'
  },
  agentStatus: {
    fontSize: 14,
    color: '#aaa',
    marginTop: 5
  }
});
```

### Deployment Strategy

1. **React Native**: Build iOS + Android apps
2. **Backend**: Same FastAPI backend (WebSocket)
3. **Voice**: Native speech-to-text (iOS Speech Framework, Android SpeechRecognizer)
4. **Notifications**: Push notifications for long-running tasks
5. **Offline Mode**: Cache agent results for offline viewing

---

## Performance Optimization

### Caching Strategy

```python
# File: sources/caching.py (New)

from functools import wraps
import redis
import json
import hashlib

class AgentCache:
    """
    Cache agent responses to reduce redundant work
    """
    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = redis.from_url(redis_url)
        self.ttl = 3600  # 1 hour

    def cache_key(self, agent_name: str, query: str) -> str:
        """Generate cache key from agent name and query"""
        query_hash = hashlib.md5(query.encode()).hexdigest()
        return f"agent:{agent_name}:{query_hash}"

    def get(self, agent_name: str, query: str):
        """Get cached result"""
        key = self.cache_key(agent_name, query)
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)
        return None

    def set(self, agent_name: str, query: str, result: str):
        """Cache result"""
        key = self.cache_key(agent_name, query)
        self.redis.setex(key, self.ttl, json.dumps(result))

# Decorator for caching agent responses
def cached_agent_response(func):
    @wraps(func)
    async def wrapper(self, query: str, *args, **kwargs):
        cache = AgentCache()

        # Check cache
        cached = cache.get(self.name, query)
        if cached:
            return cached

        # Execute agent
        result = await func(self, query, *args, **kwargs)

        # Cache result
        cache.set(self.name, query, result)

        return result
    return wrapper
```

---

## Monitoring & Observability

### Agent Execution Tracing

```python
# File: sources/tracing.py (New)

import time
from typing import Dict, List
from dataclasses import dataclass, asdict
import json

@dataclass
class AgentTrace:
    agent_name: str
    query: str
    start_time: float
    end_time: float = None
    status: str = "running"
    steps: List[Dict] = None
    error: str = None

    def duration(self) -> float:
        if self.end_time:
            return self.end_time - self.start_time
        return time.time() - self.start_time

class TracingManager:
    """
    Track and visualize agent execution
    """
    def __init__(self):
        self.traces: Dict[str, AgentTrace] = {}

    def start_trace(self, trace_id: str, agent_name: str, query: str):
        """Start tracing an agent execution"""
        self.traces[trace_id] = AgentTrace(
            agent_name=agent_name,
            query=query,
            start_time=time.time(),
            steps=[]
        )

    def add_step(self, trace_id: str, step: Dict):
        """Add a step to the trace"""
        if trace_id in self.traces:
            self.traces[trace_id].steps.append({
                **step,
                'timestamp': time.time()
            })

    def end_trace(self, trace_id: str, status: str = "success", error: str = None):
        """Complete a trace"""
        if trace_id in self.traces:
            trace = self.traces[trace_id]
            trace.end_time = time.time()
            trace.status = status
            trace.error = error

    def get_trace(self, trace_id: str) -> Dict:
        """Get trace data"""
        if trace_id in self.traces:
            return asdict(self.traces[trace_id])
        return None

    def export_trace(self, trace_id: str, format: str = "json") -> str:
        """Export trace for analysis"""
        trace = self.get_trace(trace_id)
        if format == "json":
            return json.dumps(trace, indent=2)
        # Add other formats (OpenTelemetry, etc.)

# Usage in agents
from sources.tracing import TracingManager

tracer = TracingManager()

async def agent_run(self, query: str):
    trace_id = f"{self.name}_{time.time()}"
    tracer.start_trace(trace_id, self.name, query)

    try:
        tracer.add_step(trace_id, {"action": "parsing_query"})
        # ... agent logic ...

        tracer.add_step(trace_id, {"action": "executing_tool", "tool": "browser"})
        # ... more logic ...

        tracer.end_trace(trace_id, status="success")
    except Exception as e:
        tracer.end_trace(trace_id, status="error", error=str(e))
        raise
```

---

## Security Considerations

### Sandboxing Community Tools

```dockerfile
# File: docker/tool-sandbox.Dockerfile

FROM python:3.10-slim

# Create non-root user
RUN useradd -m -u 1000 sandbox

# Install dependencies
RUN pip install --no-cache-dir \
    requests \
    beautifulsoup4

# Set resource limits
USER sandbox
WORKDIR /app

# Copy tool code
COPY tool.py /app/

# Entrypoint
ENTRYPOINT ["python", "tool.py"]
```

### Rate Limiting

```python
# File: sources/rate_limiter.py (New)

from collections import defaultdict
import time

class RateLimiter:
    """
    Rate limit agent executions per user
    """
    def __init__(self, max_requests: int = 10, window: int = 60):
        self.max_requests = max_requests
        self.window = window
        self.requests = defaultdict(list)

    def is_allowed(self, user_id: str) -> bool:
        """Check if request is allowed"""
        now = time.time()

        # Clean old requests
        self.requests[user_id] = [
            req_time for req_time in self.requests[user_id]
            if now - req_time < self.window
        ]

        # Check limit
        if len(self.requests[user_id]) >= self.max_requests:
            return False

        # Record request
        self.requests[user_id].append(now)
        return True
```

---

## Next Steps

1. **Prototype**: Build WebSocket POC (1 week)
2. **Test**: Validate with 10 users (1 week)
3. **Iterate**: Based on feedback (2 weeks)
4. **Deploy**: Roll out to production (1 week)

---

**Document Status**: Draft for Review
**Last Updated**: January 22, 2026
**Authors**: Claude Code (Agentic Exploration Session)
