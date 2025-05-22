# AI Documentation Generation & Azure DevOps Deployment Workflow

## Agents / Agentic Workflows Name
-   **Document Generation Agent (LLM)**: Generates content for different document sections (Business Requirements, Technical Requirements, User Stories, Tasks) using prompts and previous stage outputs. It utilizes `litellm.completion` via `get_llm_response` in `documentation.py`.
-   **Document Validation Agent (LLM)**: Reviews generated document sections against predefined templates and criteria. It uses `litellm.completion` via `get_llm_response` in `documentation.py`'s `validate_document` endpoint.
-   **Azure DevOps Deployment Agent**: Orchestrates the deployment of generated user stories and tasks to Azure DevOps. This agent is invoked via the `deploy_to_azure` endpoint.
    -   **LLMProcessor (for Azure DevOps)**: A sub-agent used by the Azure DevOps Deployment Agent. It processes markdown from user stories and tasks, converting it into a structured JSON format suitable for Azure DevOps API. This uses `LLMProcessor.get_llm_response` from `devops.py`.
-   **Word Document Formatter Agent**: Converts the markdown content from all generated document stages into a final, downloadable Microsoft Word document using `python-docx` via `format_markdown_to_docx` in `documentation.py`.
-   **(Implied) Orchestrator**: A system (likely FastAPI routing and service logic in `documentation.py`) that manages the overall workflow, user interactions, progression through documentation stages, and invokes the specialized agents as needed.

## Agent / Workflow Description
This workflow automates the creation of comprehensive project documentation and facilitates its deployment to Azure DevOps. The process unfolds in several stages:

1.  The user initiates the workflow by providing an **Initial Project Description** which serves as the prompt for the "Business Requirements" stage.
2.  The **Orchestrator** guides the process. The **Document Generation Agent** is invoked for each documentation stage:
    * **Business Requirements**: Generated based on the initial user prompt.
    * **Technical Requirements**: Generated using the approved Business Requirements as input.
    * **User Stories**: Generated based on both Business and Technical Requirements.
    * **Tasks**: Generated from the User Stories.
3.  At each stage, the generated markdown content is stored in the database (`ProjectDocument` model).
4.  Users can review and edit the generated content.
5.  The **Document Validation Agent** can be optionally invoked to review the content of any stage, providing structured feedback based on predefined templates. This feedback can inform user edits or regeneration requests.
6.  Users can choose to **Regenerate** content for a stage based on the original content, validation feedback, or an entirely new custom prompt.
7.  Once all documentation stages are satisfactory, the **Word Document Formatter Agent** can compile all markdown sections into a single, formatted `.docx` file for download.
8.  Alternatively, or in addition, the user stories and tasks can be deployed to Azure DevOps. The **Azure DevOps Deployment Agent** takes over:
    * It uses its internal **LLMProcessor (for Azure DevOps)** to convert the markdown for user stories and tasks into a structured JSON format.
    * It then uses the `AzureDevOpsAPI` tools to create a new project in Azure DevOps and populate it with work items (User Stories and Tasks, maintaining parent-child relationships). Details of these created Azure DevOps work items are stored in the database (`AzureDevOpsWorkItem` model).

## Domain / Industry
-   Software Development
-   Project Management
-   Agile Development
-   Technical Documentation
-   DevOps Automation

## Tools / Functions Used By Agents

### Document Generation Agent:
-   `litellm.completion` (accessed via `get_llm_response` in `backend/app/api/routes/documentation.py`):
    -   Generates text content for Business Requirements, Technical Requirements, User Stories, and Tasks.
    -   Supports various LLM providers (Azure OpenAI, OpenAI, Ollama) based on application settings (`app.core.config.settings`).
-   **Inputs**: Stage-specific prompts, content from previous documentation stages (e.g., Business Requirements text is input for Technical Requirements generation).

### Document Validation Agent:
-   `litellm.completion` (accessed via `get_llm_response` within the `validate_document` endpoint in `backend/app/api/routes/documentation.py`):
    -   Reviews document sections against structured validation templates (prompts define the review criteria for each stage).
-   **Inputs**: Markdown content of the document stage to be validated, stage-specific validation template/prompt.

### Azure DevOps Deployment Agent (and its LLMProcessor):
-   **LLMProcessor's `get_llm_response`** (from `backend/app/api/routes/devops.py`, invoked by `azuredevops.py`):
    -   Converts markdown representations of User Stories and Tasks into a structured JSON format required by the Azure DevOps API.
-   **`AzureDevOpsAPI` methods** (from `backend/app/api/routes/devops.py`):
    -   `create_project(name, description)`: Creates a new project in Azure DevOps.
    -   `get_work_item_types(project_name)`: Fetches available work item types for a project.
    -   `determine_work_item_type(available_types, desired_type)`: Selects the appropriate work item type name (e.g., "User Story", "Task") based on project settings.
    -   `create_work_item(project_name, work_item_type, title, description, parent_id)`: Creates individual work items (User Stories, Tasks) and links tasks to parent user stories.
-   **Database Interaction**:
    -   Saves created Azure DevOps work item details (ID, type, title, URL, parent link) to the `AzureDevOpsWorkItem` table.

### Word Document Formatter Agent:
-   `python-docx` library functions (via `format_markdown_to_docx` in `backend/app/api/routes/documentation.py`):
    -   Parses markdown elements (headers, tables, paragraphs, lists).
    -   Constructs a `.docx` document reflecting the structure and content of the generated markdown.

### Orchestrator (Implied System Logic):
-   FastAPI routing (`backend/app/api/routes/documentation.py`, `backend/app/api/routes/azuredevops.py`): Manages API endpoints for different workflow steps.
-   Database session management (`SessionDep`): For storing and retrieving `ProjectDocument` and `AzureDevOpsWorkItem` data.
-   Manages state between stages (e.g., passing Business Requirements content to the Technical Requirements generation step).

## Architecture Design
```mermaid
graph TD
    User[User Portal] -->|1. Initial Project Desc.| API_GenBR["API: /documentation/generate <br> (Stage: business_requirements)"]
    API_GenBR --> Agent_GenBR[Document Generation Agent <br> (LLM for Business Reqs)]
    Agent_GenBR -->|Markdown: BR| DB_StoreBR[(DB: ProjectDocument.business_reqs)]
    DB_StoreBR --> UserReviewBR{User Review/Edit BR}

    UserReviewBR -->|Approved BR| API_GenTR["API: /documentation/generate <br> (Stage: technical_requirements)"]
    API_GenTR --> Agent_GenTR[Document Generation Agent <br> (LLM for Technical Reqs)]
    Agent_GenTR -->|Markdown: TR| DB_StoreTR[(DB: ProjectDocument.technical_reqs)]
    DB_StoreTR --> UserReviewTR{User Review/Edit TR}

    UserReviewTR -->|Approved TR & BR| API_GenUS["API: /documentation/generate <br> (Stage: user_stories)"]
    API_GenUS --> Agent_GenUS[Document Generation Agent <br> (LLM for User Stories)]
    Agent_GenUS -->|Markdown: US| DB_StoreUS[(DB: ProjectDocument.user_stories)]
    DB_StoreUS --> UserReviewUS{User Review/Edit US}

    UserReviewUS -->|Approved US| API_GenTasks["API: /documentation/generate <br> (Stage: tasks)"]
    API_GenTasks --> Agent_GenTasks[Document Generation Agent <br> (LLM for Tasks)]
    Agent_GenTasks -->|Markdown: Tasks| DB_StoreTasks[(DB: ProjectDocument.tasks)]
    DB_StoreTasks --> UserReviewTasks{User Review/Edit Tasks}

    subgraph Optional Validation at each stage
        direction LR
        StageContent[Current Stage Markdown] --> API_ValidateDoc["API: /documentation/validate"]
        API_ValidateDoc --> Agent_Validate[Document Validation Agent <br> (LLM with Validation Template)]
        Agent_Validate -->|Validation Feedback| UserReviewWithFeedback{User Review with Feedback}
        UserReviewWithFeedback -->|User decides to regenerate/edit| RegenModal[Regenerate Options Modal UI]
        RegenModal --> API_RegenDoc["API: /documentation/generate <br> (with new/refined prompt)"]
        API_RegenDoc --> StageContent
    end

    UserReviewTasks -->|Request Final Document| API_FinalDoc["API: /documentation/final"]
    API_FinalDoc --> Agent_WordFormatter[Word Document Formatter Agent <br> (python-docx)]
    Agent_WordFormatter -->|Generated .docx| UserDownload[User Downloads Document]

    UserReviewTasks -->|Request Azure DevOps Deployment| API_DeployAzureDoc["API: /documentation/deploy-azure"]
    API_DeployAzureDoc --> Agent_AzureDevOps[Azure DevOps Deployment Agent]
    Agent_AzureDevOps --> LLMProcessor_ADO[LLMProcessor for ADO <br> (Markdown to JSON)]
    LLMProcessor_ADO -->|Structured JSON| AzureDevOpsAPI[AzureDevOpsAPI Tools <br> (create_project, create_work_item)]
    AzureDevOpsAPI --> ADO_External[Azure DevOps Service]
    AzureDevOpsAPI --> DB_StoreADOWorkItems[(DB: AzureDevOpsWorkItem)]
    ADO_External --> UserAccessADO[User Accesses ADO Project]

    subgraph External Services Utilized
        Agent_GenBR ----> LLM_Service[LLM Service (OpenAI, Azure, Ollama via LiteLLM)]
        Agent_GenTR ----> LLM_Service
        Agent_GenUS ----> LLM_Service
        Agent_GenTasks ----> LLM_Service
        Agent_Validate ----> LLM_Service
        LLMProcessor_ADO ----> LLM_Service
    end
```
The architecture outlines a sequential, multi-agent process for generating detailed project documentation, with options for validation, user editing, and final deployment to Azure DevOps or compilation into a Word document.

---

# AI Project Assistant (Chatbot)

## Agents / Agentic Workflows Name
-   **Chat AI Agent (LLMProcessor)**: Generates responses to user queries within a chat interface. It uses `LLMProcessor.chat_completion` from `devops.py`, which leverages `litellm.completion`.
-   **(Implied) Context Assembler**: A system component that gathers chat history and relevant project document context (business requirements, technical requirements, user stories, tasks status) to provide a rich prompt to the Chat AI Agent.

## Agent / Workflow Description
This workflow provides an AI-powered chat assistant to help users with project management tasks and analysis of their project documents.

1.  The user selects a specific project document they wish to discuss.
2.  The user sends a message through the chat interface (endpoint: `/documentation/chat/message/{document_id}`).
3.  The **Context Assembler** prepares the input for the AI:
    * It retrieves the last 10 messages from the chat history for the selected document from the database (`ChatMessage` model).
    * It fetches the content of the selected `ProjectDocument` (Business Requirements, Technical Requirements, User Stories, Tasks).
    * It constructs a detailed system message that instructs the LLM on its role (AI Project Assistant specializing in software development project management and requirements analysis), desired response formatting (markdown, Mermaid diagrams), and key areas of knowledge based on the document's content.
4.  The **Chat AI Agent** (`LLMProcessor.chat_completion`) receives the full conversation history (including the new user message) and the comprehensive context.
5.  The Chat AI Agent processes this input using the configured LLM (e.g., GPT-4o, Azure OpenAI, Ollama) to generate a relevant and contextually aware response.
6.  The system saves both the user's message and the AI's response to the database.
7.  The AI's response, along with token usage statistics for the interaction, is displayed back to the user in the chat interface.

## Domain / Industry
-   Software Development
-   Project Management
-   AI-Assisted Work
-   Requirements Analysis Support

## Tools / Functions Used By Agents

### Chat AI Agent (LLMProcessor):
-   `LLMProcessor.chat_completion` (from `backend/app/api/routes/devops.py`):
    -   Generates conversational responses.
    -   Utilizes `litellm.completion` to interface with various LLM providers (Azure OpenAI, OpenAI, Ollama based on `app.core.config.settings`).
-   **Inputs**:
    -   Formatted conversation history.
    -   Detailed context string including a system message, content from the relevant `ProjectDocument` sections (Business Requirements, Technical Requirements, User Stories, Tasks, Constraints/Risks), and current status of these sections.

### Context Assembler (Implied System Logic in `backend/app/api/routes/chat.py`):
-   Database Access (`SessionDep`, `get_db`):
    -   Retrieves `ProjectDocument` content for context.
    -   Fetches `ChatMessage` history.
-   String Formatting: Constructs the system message and integrates document content into the context provided to the LLM.

### External Services:
-   LLM Services (Azure OpenAI, OpenAI, Ollama): Accessed via `LLMProcessor` and `litellm`.

## Architecture Design
```mermaid
graph TD
    User[User via UI] -->|1. Selects Document & Sends Message| API_SendMessage["API: /chat/message/{document_id}"]
    
    API_SendMessage --> DB_GetDoc[DB: Fetch ProjectDocument by ID]
    API_SendMessage --> DB_GetHistory[DB: Fetch ChatMessage History (last 10)]
    
    subgraph ContextPreparation [Context Assembler]
        DB_GetDoc --> FormatContext[Format Document Content for Prompt]
        DB_GetHistory --> FormatContext
        SystemMsgDefine[Define System Message (Role, Formatting Guide)] --> FormatContext
    end
    
    FormatContext -->|Formatted Prompt (System Msg + Doc Context + History + User Msg)| Agent_ChatAI[Chat AI Agent <br> (LLMProcessor.chat_completion)]
    
    Agent_ChatAI --> LLM_Service[LLM Service <br> (OpenAI, Azure, Ollama via LiteLLM)]
    LLM_Service -->|LLM Response| Agent_ChatAI
    
    Agent_ChatAI -->|AI's Text Response, Token Usage| API_SendMessage
    
    API_SendMessage --> DB_SaveUserMsg[DB: Store User's ChatMessage]
    API_SendMessage --> DB_SaveAIMsg[DB: Store AI's ChatMessage]
    
    API_SendMessage -->|AI Response & Usage Stats to UI| User
```
The AI Project Assistant enhances user interaction by providing an intelligent chatbot capable of understanding and discussing project details within the context of specific documentation.
