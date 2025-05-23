# Potpie AI Agent Platform

## Agents / Agentic Workflows Name

-   **Core Platform Orchestration**:
    -   **API Service (FastAPI)**: Entry point for all user interactions, routing requests to appropriate controllers.
    -   **Celery Task Queue & Workers**: Manages asynchronous tasks, primarily for repository parsing and knowledge graph construction.
-   **Code Understanding & Knowledge Base Construction Workflow**:
    -   **Parsing Controller**: Handles user requests to parse new repositories.
    -   **Parsing Service (Celery Task)**:
        -   Clones/accesses code repositories (GitHub or local).
        -   Uses `CodeGraphService` (or `blar_graph`) to build a detailed code knowledge graph in Neo4j.
        -   Invokes `InferenceService` to generate embeddings and docstrings for code nodes using LLMs, enriching the Neo4j graph.
        -   Utilizes `SearchService` to create/update full-text search indices in PostgreSQL.
-   **Agent-Driven Interaction Workflow**:
    -   **Conversation Controller**: Manages user chat sessions and history (persisted in PostgreSQL).
    -   **Supervisor Agent (using `AutoRouterAgent`)**: Analyzes user queries and routes them to the most suitable specialized agent.
    -   **System Agents (Chat Agents)**:
        -   **Codebase Q&A Agent (`QnAAgent`)**: Answers questions about the codebase using the knowledge graph and code context.
        -   **Debugging Agent (`DebugAgent`)**: Assists in debugging code by analyzing stack traces and code context.
        -   **Unit Test Agent (`UnitTestAgent`)**: Generates unit test plans and code for specified functions/classes.
        -   **Integration Test Agent (`IntegrationTestAgent`)**: Generates integration test plans and code for component interactions.
        -   **Low-Level Design Agent (`LowLevelDesignAgent`)**: Creates detailed low-level design plans for new features.
        -   **Code Changes Agent (Blast Radius Agent - `BlastRadiusAgent`)**: Analyzes the impact of code changes in a branch.
        -   **Code Generation Agent (`CodeGenAgent`)**: Generates new code snippets or modifies existing code based on requirements.
        -   **General Purpose Agent (`GeneralPurposeAgent`)**: Handles queries not requiring specific codebase access, often using web search tools.
    -   **Custom Agent Workflow**:
        -   **Custom Agent Controller & Service**: Allows users to define, create, update, and manage custom agents (role, goal, tasks, tools). Stored in PostgreSQL.
        -   **Custom Agent Runtime (`RuntimeAgent`)**: Executes these user-defined agents, typically using frameworks like CrewAI, providing access to platform tools and LLMs.
-   **Supporting Services**:
    -   **LLM Provider Service**: Manages interactions with various LLMs (OpenAI, Anthropic, etc.) via LiteLLM and Portkey.
    -   **Tool Service**: Aggregates and provides a suite of tools for agents to use (e.g., knowledge graph querying, code fetching, web search, GitHub API interaction).
    -   **Chat History Service**: Manages and retrieves conversation history from PostgreSQL.
    -   **Project Service**: Manages project metadata (repository info, parsing status) in PostgreSQL.
    -   **Auth Service & API Key Service**: Handles user authentication and API key management.
    -   **Secret Manager**: Manages API keys for external services (LLMs, Linear, etc.), storing them securely (GCP Secret Manager or encrypted in DB).

## Agent / Workflow Description

Potpie is an AI-powered developer platform designed to understand and interact with codebases. It builds a knowledge graph from code repositories and deploys a suite of specialized AI agents (both pre-built and custom) to assist with various software development tasks such as Q&A, debugging, code generation, and impact analysis.

1.  **Code Ingestion and Knowledge Graph Construction**:
    *   A user submits a repository (GitHub URL or local path during development) via the **Parsing Controller**.
    *   This triggers an asynchronous **Celery Task** managed by the **Parsing Service**.
    *   The service clones/accesses the repository, then uses a `CodeGraphService` (or `blar_graph` for Python/JS/TS) to parse the code structure, identify entities (functions, classes, files), and their relationships.
    *   This structural information is stored as a knowledge graph in a **Neo4j database**.
    *   The `InferenceService` then processes nodes in this graph, using LLMs (via **LLM Provider Service**) to generate docstrings and vector embeddings for code snippets and docstrings. These enrichments are stored back in Neo4j.
    *   A `SearchService` simultaneously builds or updates full-text search indices in **PostgreSQL** to enable keyword-based code search.
    *   The `ProjectService` updates the parsing status of the project in PostgreSQL.

2.  **Agent-Driven Code Interaction**:
    *   A user initiates a conversation through an API endpoint, handled by the **Conversation Controller**.
    *   The `ConversationService` retrieves/updates conversation history (from PostgreSQL via `ChatHistoryService`) and passes the user's query to the **Supervisor Agent**.
    *   The **Supervisor Agent** (implemented with `AutoRouterAgent`) intelligently routes the query to the most appropriate specialized agent. This could be:
        *   One of the **System Agents** (e.g., `QnAAgent` for understanding code, `CodeGenAgent` for writing code, `UnitTestAgent` for generating tests).
        *   A **Custom Agent Runtime** if the query is directed at a user-defined agent.
    *   The selected agent executes its task by:
        *   Leveraging the **LLM Provider Service** for core reasoning and generation, which interfaces with various LLMs like GPT, Claude, etc., through LiteLLM and potentially Portkey.
        *   Utilizing tools from the **Tool Service**. These tools allow agents to:
            *   Query the Neo4j knowledge graph (e.g., `ask_knowledge_graph_queries`, `get_node_neighbours_from_node_id`).
            *   Fetch specific code snippets or file structures (`get_code_from_node_id`, `get_code_file_structure`).
            *   Analyze code changes (`change_detection`).
            *   Interact with GitHub (create PRs, add comments via `GitHubTools`).
            *   Search the web (`web_search_tool`, `webpage_extractor`).
            *   Manage issues in Linear (`get_linear_issue`, `update_linear_issue`).
    *   The agent's response, often generated as a stream, is sent back to the user.

3.  **Custom Agent Creation and Execution**:
    *   Users can define their own agents through the **Custom Agent Controller**.
    *   The `CustomAgentService` stores these agent definitions (role, goal, backstory, tasks, selected tools) in PostgreSQL.
    *   When a custom agent is invoked, the **Custom Agent Runtime** (e.g., using `RuntimeAgent` with CrewAI) executes its defined tasks, using the same LLM Provider and Tool Services available to system agents.

## Domain / Industry

-   Software Development
-   Developer Tools
-   AI-Assisted Engineering
-   DevOps & Code Intelligence

## Tools / Functions Used By Agents

### Knowledge Graph & Code Analysis Tools
-   `get_code_from_probable_node_name`: Fetches code/docstring by a likely name (uses SearchService then Neo4j).
-   `get_code_from_node_id`: Fetches code/docstring for a specific graph node ID (Neo4j, CodeProviderService).
-   `get_code_from_multiple_node_ids`: Batched version for multiple node IDs.
-   `ask_knowledge_graph_queries`: Performs vector similarity search on docstring embeddings in Neo4j.
-   `get_nodes_from_tags`: Retrieves nodes from Neo4j based on assigned tags (e.g., API, DATABASE).
-   `get_code_graph_from_node_id`: Fetches a local subgraph around a given node from Neo4j.
-   `get_code_file_structure`: Retrieves the directory and file structure of a repository (CodeProviderService).
-   `get_node_neighbours_from_node_id`: Gets directly connected nodes (callers, callees) from Neo4j.
-   `intelligent_code_graph`: Fetches a code graph and uses an LLM to filter/prune it for relevance.
-   `fetch_file`: Retrieves the raw content of a specified file (CodeProviderService).
-   `change_detection`: Analyzes git diffs between branches, identifies changed functions, and their entry points. (CodeProviderService, Neo4j, SearchService).

### External Service Interaction Tools
-   **GitHub Tools**:
    -   `github_tool` (GitHubContentFetcher): Fetches content of GitHub issues or PRs.
    -   `github_create_branch`: Creates a new branch in a GitHub repository.
    -   `github_update_branch` (GitHubUpdateFileTool): Updates/creates a file in a GitHub branch.
    -   `github_create_pull_request`: Creates a new pull request on GitHub.
    -   `github_add_pr_comments`: Adds review comments to a GitHub PR.
-   **Web Tools**:
    -   `webpage_extractor` (WebpageExtractorTool): Extracts content from a given URL using Firecrawl API.
    -   `web_search_tool`: Performs a web search using an LLM with search capabilities (e.g., Perplexity via OpenRouter).
-   **Linear Issue Tracking Tools**:
    -   `get_linear_issue`: Fetches details of a specific Linear issue.
    -   `update_linear_issue`: Updates a Linear issue (e.g., status, assignee, comment).
-   **LLM Interaction**:
    -   `think_tool`: Allows an agent to perform a "thinking step" using an LLM call to process information or plan next steps.

### Core Platform Services (Internal "Tools" for Agents)
-   **LLMProviderService**: Centralized access to LLMs (OpenAI, Anthropic, etc.) via LiteLLM, handles API key management via `SecretManager`.
-   **CodeProviderService**: Abstracts file/repository access from GitHub or local file system.
-   **ChatHistoryService**: Provides conversation history to maintain context.

### Agent Frameworks
-   **CrewAI**: Used by `RuntimeAgent` for orchestrating custom agent tasks.
-   **PydanticAI**: Used by `PydanticRagAgent` as an alternative agent execution framework.
-   **Langchain**: Potentially used by `LangchainRagAgent` (though less emphasized in the current structure).

## Architecture Design

```mermaid
graph TD
    User[User (Web UI/VSCode/API)] -->|Requests| API[FastAPI Service]

    subgraph API
        Router[API Router]
        Auth[Auth Service]
        Controllers[Controllers (Parsing, Conversation, Project, CustomAgent)]
    end

    Controllers -->|Invoke| Services[Core Services]

    subgraph Services
        ProjectSvc[Project Service]
        ConversationSvc[Conversation Service]
        CustomAgentSvc[Custom Agent Service]
        LLMSvc[LLM Provider Service]
        ToolSvc[Tool Service]
        HistorySvc[Chat History Service]
        SecretMgr[Secret Manager]
        SearchSvc[Search Service]
    end

    Controllers -->|Dispatch Parse Task| CeleryQueue[Celery Task Queue]
    CeleryQueue -->|Assign Task| CeleryWorker[Celery Worker (ParsingService)]

    CeleryWorker -->|Access Code| CodeProvider[Code Provider Service]
    CodeProvider -->|GitHub/Local| GitRepos[Git Repositories]
    CeleryWorker -->|Build Graph| GraphBuilder[Graph Builder (CodeGraphService/blar_graph)]
    GraphBuilder -->|Store/Update| Neo4j[Neo4j Knowledge Graph]
    CeleryWorker -->|Enrich Graph| InferenceSvc[Inference Service (Embeddings/Docstrings)]
    InferenceSvc --> LLMSvc
    InferenceSvc --> Neo4j
    CeleryWorker -->|Index Data| SearchSvc

    ConversationSvc --> Supervisor[Supervisor Agent]
    Supervisor -->|Route Query| SystemAgents[System Agents]
    Supervisor -->|Route Query| CustomAgentRuntime[Custom Agent Runtime (CrewAI/PydanticAI)]

    SystemAgents --> LLMSvc
    SystemAgents --> ToolSvc
    SystemAgents --> HistorySvc

    CustomAgentRuntime --> LLMSvc
    CustomAgentRuntime --> ToolSvc
    CustomAgentRuntime --> HistorySvc

    ToolSvc --> KGTools[KG & Code Tools]
    ToolSvc --> ExtSVCIntegrations[External Service Tools (GitHub, Web, Linear)]
    ToolSvc --> ThinkTool[Think Tool]

    KGTools --> Neo4j
    KGTools --> CodeProvider
    KGTools --> SearchSvc
    ExtSVCIntegrations --> GitHubAPI[GitHub API]
    ExtSVCIntegrations --> Firecrawl[Firecrawl API]
    ExtSVCIntegrations --> LinearAPI[Linear API]
    ExtSVCIntegrations --> LLMSvcPxy[LLM Provider (for Web Search)]
    ThinkTool --> LLMSvc

    LLMSvc --> LiteLLM[LiteLLM Gateway]
    LLMSvcPxy --> LiteLLM
    LiteLLM -->|Access| LLMs[External LLMs (OpenAI, Anthropic, etc.)]

    Auth --> Postgres[PostgreSQL DB]
    ProjectSvc --> Postgres
    ConversationSvc --> Postgres
    CustomAgentSvc --> Postgres
    HistorySvc --> Postgres
    SearchSvc --> Postgres
    SecretMgr -->|Encrypted Keys| Postgres
    SecretMgr -->|Or| GCPSecretManager[GCP Secret Manager]

    CeleryQueue -- Broker/Backend --> Redis[Redis Cache & Broker]
    ToolSvc -->|Cache API results?| Redis
    ProjectSvc -->|Cache File Structure?| Redis
```

**Architecture Description:**

The Potpie AI Agent Platform is a modular system designed to provide AI-driven assistance for software development by creating and managing agents that interact with codebases.

1.  **User Interaction & API Layer**: Users interface with Potpie via a Web UI, VSCode extension, or direct API calls. The FastAPI-based API Service acts as the central gateway, handling authentication (via `AuthService`) and routing requests to various `Controllers` (Parsing, Conversation, Project, Custom Agent).

2.  **Core Services**: A suite of services underpins the platform's functionality:
    *   `ProjectService` and `ConversationService` manage project metadata and chat session states, respectively, persisting data in PostgreSQL.
    *   `CustomAgentService` handles the lifecycle of user-defined agents.
    *   `LLMProviderService` abstracts interactions with diverse Large Language Models (e.g., OpenAI, Anthropic) using LiteLLM, possibly through a proxy like Portkey. It retrieves API keys via `SecretManager`.
    *   `ToolService` acts as a registry and dispatcher for tools available to agents.
    *   `ChatHistoryService` maintains conversation context, stored in PostgreSQL.
    *   `SearchService` provides full-text search capabilities over indexed code.
    *   `SecretManager` securely handles API keys for external services, with options for GCP Secret Manager or encrypted storage in PostgreSQL.

3.  **Asynchronous Code Processing (Knowledge Base Construction)**:
    *   When a repository is submitted, the `ParsingController` dispatches a task to a `Celery Task Queue`.
    *   A `CeleryWorker` picks up the task and executes the `ParsingService`. This service:
        *   Retrieves code using `CodeProviderService` (interfacing with GitHub or local Git).
        *   Constructs a code knowledge graph using `GraphBuilder` (custom logic or `blar_graph`) and stores it in a Neo4j database.
        *   Invokes `InferenceService` to generate docstrings and vector embeddings for code elements (using LLMs), which are then added to the Neo4j graph.
        *   Triggers `SearchService` to update PostgreSQL-based search indices.

4.  **Agent Orchestration and Execution**:
    *   User queries within a conversation are managed by `ConversationService`, which passes them to the `SupervisorAgent`.
    *   The `SupervisorAgent` (likely using `AutoRouterAgent` logic) analyzes the query and routes it to the most appropriate agent:
        *   **System Agents**: Pre-defined agents specialized for tasks like Q&A (`QnAAgent`), debugging (`DebugAgent`), code generation (`CodeGenAgent`), test creation (`UnitTestAgent`, `IntegrationTestAgent`), LLD (`LowLevelDesignAgent`), and change analysis (`BlastRadiusAgent`).
        *   **Custom Agent Runtime**: Executes agents defined by users, leveraging frameworks like CrewAI (`RuntimeAgent`) or PydanticAI.
    *   All agents interact with `LLMProviderService` for AI capabilities and `ToolService` to perform actions.

5.  **Tools Ecosystem**:
    *   The `ToolService` provides agents access to:
        *   **Knowledge Graph & Code Tools**: For querying Neo4j and fetching code/structure via `CodeProviderService`.
        *   **External Service Tools**: For interacting with GitHub API (PRs, issues, branches), Firecrawl API (webpage extraction), Linear API (issue tracking), and LLM-powered web search.
        *   **Think Tool**: An internal LLM call for an agent to process information or plan.

6.  **Data Storage and Caching**:
    *   **PostgreSQL**: The primary relational database for users, projects, conversations, custom agent definitions, search indices, and encrypted secrets.
    *   **Neo4j**: Stores the detailed code knowledge graph.
    *   **Redis**: Functions as the Celery message broker and results backend, and potentially for caching frequently accessed data (e.g., project file structures, API responses).

This architecture enables Potpie to create a rich, queryable representation of codebases and empower various AI agents to perform complex software engineering tasks by combining LLM reasoning with specialized tools and contextual data from the knowledge graph and conversation history.# Potpie AI Agent Platform
