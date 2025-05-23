# MedAuth: Automated Prior Authorization Determination & Evaluation Workflow

## Agents / Agentic Workflows Name
-   **Auto-Determination Workflow (Orchestrated by `AutoPADeterminator`)**:
    -   **Policy Summarization Agent (Implicit within `AutoDeterminationEvaluator`)**: Utilizes an LLM (via `AzureOpenAIManager`) to summarize lengthy policy documents if they exceed context limits for the main determination task.
    -   **Prior Authorization Determination Agent (Core `AutoPADeterminator` logic)**: Employs an LLM (Azure OpenAI GPT-4o or O1 models via `AzureOpenAIManager`) guided by detailed prompts to analyze patient information, physician details, clinical data, and relevant (potentially summarized) policy texts. It then generates a prior authorization decision (Approved, Denied, or Needs More Information) along with a supporting rationale.
    -   **Determination Summarization Agent (Implicit within `AutoDeterminationEvaluator`)**: Uses an LLM to convert the detailed adjudication output into a structured JSON format.
-   **Auto-Determination Evaluation Pipeline (Orchestrated by `AutoDeterminationEvaluator`)**:
    -   **Test Case Preprocessing Agent**: Loads test scenarios from YAML files, prepares context data (patient info, physician info, clinical info, policy text), and invokes the `AutoPADeterminator` workflow for each case to generate a determination.
    -   **Azure AI Evaluation Agent**: Takes the generated determinations and ground truth data, then submits them to the Azure AI Foundry evaluation service. This agent uses a suite of built-in and custom evaluators to score the performance of the `AutoPADeterminator`.
    -   **Results Postprocessing Agent**: Aggregates and summarizes the evaluation metrics from Azure AI Foundry into a final report.

## Agent / Workflow Description
This system, part of the MedEvals framework, focuses on automating and evaluating prior authorization (PA) decisions in healthcare. The workflow involves two main parts: generating a PA determination using an AI agent (`AutoPADeterminator`) and then evaluating the performance of this agent using a dedicated pipeline (`AutoDeterminationEvaluator`).

**1. Automated Prior Authorization Determination (by `AutoPADeterminator`):**
   -   For a given PA request (including patient, physician, and clinical information, plus relevant payor policy text), the `AutoPADeterminator` agent takes charge.
   -   **Policy Summarization**: If the provided policy text is too long for the LLM's context window, it's first summarized by the **Policy Summarization Agent** using Azure OpenAI.
   -   **Determination Generation**: The **Prior Authorization Determination Agent** then processes the (summarized) policy along with the structured case information. It uses sophisticated prompt engineering (leveraging templates like `prior_auth_system_prompt.jinja` and `prior_auth_user_prompt.jinja`) to instruct an Azure OpenAI model (GPT-4o or a specialized O1 model).
   -   The LLM analyzes the information against policy criteria and produces a decision: "Approved," "Denied," or "Needs More Information," complete with a detailed rationale and assessment against each policy criterion.
   -   **Determination Summarization**: The output from the LLM is then processed by the **Determination Summarization Agent** to ensure it conforms to a structured JSON format, using prompts like `summarize_autodetermination_user.jinja`.

**2. Evaluation of the Auto-Determination (by `AutoDeterminationEvaluator`):**
   -   The `AutoDeterminationEvaluator` orchestrates the end-to-end evaluation.
   -   **Test Case Preprocessing**: It begins by loading predefined test cases from YAML files. Each case includes ground truth data. The evaluator then runs the `AutoPADeterminator` workflow (as described above) for every test case to get an AI-generated determination.
   -   **Azure AI Evaluation**: The generated determinations are compared against the ground truth answers. The **Azure AI Evaluation Agent** submits this data to Azure AI Foundry's evaluation service. This service utilizes a variety of metrics, including custom evaluators (e.g., for factual correctness, semantic similarity, fuzzy matching) and potentially standard Azure evaluators.
   -   **Results Postprocessing**: Finally, the evaluation scores and metrics are collected and summarized into a comprehensive report, allowing for quantitative assessment of the `AutoPADeterminator`'s accuracy and reliability.

This entire process is designed to rigorously test and validate AI-driven prior authorization decision-making within a controlled and measurable environment, leveraging Azure AI Foundry for robust evaluation.

## Domain / Industry
-   Healthcare
-   Health Insurance (Payer Operations)
-   Revenue Cycle Management
-   Clinical AI Application Evaluation

## Tools / Functions Used By Agents

### Auto-Determination Workflow (`AutoPADeterminator` & supporting logic in `AutoDeterminationEvaluator`)
-   **`AzureOpenAIManager`**:
    -   `generate_chat_response()` / `generate_chat_response_o1()`: Core functions for interacting with Azure OpenAI models (GPT-4o, O1-preview) to perform policy summarization, determination generation, and determination summarization.
-   **`PromptManager`**:
    -   `get_prompt()`: Loads Jinja2 templates.
    -   `create_prompt_pa()`: Constructs the main user prompt for PA determination using `prior_auth_user_prompt.jinja` or `prior_auth_o1_user_prompt.jinja`.
    -   `create_prompt_summary_policy()`: Constructs prompts for policy summarization using `summarize_policy_user.jinja` and `summarize_policy_system.jinja`.
    -   `create_prompt_summary_autodetermination()`: Constructs prompts for summarizing/structuring the LLM's determination output using `summarize_autodetermination_user.jinja` and `summarize_autodetermination_system.jinja`.
-   **Pydantic Models** (`src/pipeline/promptEngineering/models.py`): Used to structure patient, physician, and clinical information passed to prompts.
-   **JQ**: Used by `AutoDeterminationEvaluator.process_generated_output()` to parse and extract specific fields from the JSON output of the determination agent.

### Auto-Determination Evaluation Pipeline (`AutoDeterminationEvaluator`)
-   **`PipelineEvaluator` (Base Class)**: Provides the abstract structure (`preprocess`, `run_evaluations`, `post_processing`).
-   **`AIFoundryManager`**:
    -   `_initialize_project()`: Connects to the Azure AI Foundry project.
    -   Used to pass Azure AI project configuration to the `evaluate` SDK call.
-   **Azure AI Evaluation SDK (`azure.ai.evaluation.evaluate`)**:
    -   Orchestrates the evaluation run, submitting data and evaluator configurations to Azure AI Foundry.
    -   `custom_azure_ai_evaluations.custom_start_run`: Modified SDK function to include custom tags in evaluation runs.
-   **Custom Evaluators (`src/evals/custom/`)**:
    -   `FactualCorrectnessEvaluator`: Uses RAGAS and `AzureChatOpenAI` (from LangChain) for factual correctness scoring.
    -   `FuzzyEvaluator`: Uses `rapidfuzz.fuzz.ratio` for similarity scoring.
    -   `NgramSimilarityEvaluator`: Computes n-gram based similarity.
    -   `SemanticSimilarityEvaluator`: Uses Hugging Face Transformers (`AutoModel`, `AutoTokenizer` e.g., `bert-base-uncased`) for semantic similarity.
    -   `SequenceEvaluator`: Uses `difflib.SequenceMatcher` for sequence similarity.
    -   `SlidingFuzzyEvaluator`: Uses `rapidfuzz.fuzz.ratio` with a sliding window.
-   **YAML Parsing**: To load test case configurations (`_.yaml.example`, `policy-retrieval-hybrid-semantic-001.yaml`).
-   **File System Operations (`os`, `glob`, `shutil`)**: For managing test case files and temporary directories.
-   **Asynchronous Operations (`asyncio`)**: Used in `AutoDeterminationEvaluator` to run preprocessing and response generation.

### General Supporting Tools & Libraries
-   **Azure SDKs**: `azure-identity`, `azure-storage-blob`, `azure-search-documents`, `azure-ai-documentintelligence`.
-   **Data Handling**: `pandas`, `numpy`.
-   **Logging**: Custom logger (`src/utils/ml_logging.py`) with OpenTelemetry and Azure Monitor integration.
-   **Image Processing**: `SimpleITK`, `pydicom`, `Pillow (PIL)`, `scikit-image`, `nibabel` (for DICOM, NIFTI, and general image manipulation, though less central to the described Auto-Determination workflow, they are present in the broader repo).
-   **HTTP Requests**: `requests`.
-   **Environment Management**: `python-dotenv`.

## Architecture Design

```mermaid
graph TD
    A["Test Case Data (YAML)<br/>📄 Patient Info<br/>👨‍⚕️ Physician Info<br/>🏥 Clinical Info<br/>📋 Policy Text<br/>✅ Ground Truth Determination"] --> B{AutoDeterminationEvaluator}
    
    subgraph AutoPAD ["🤖 AutoDeterminationEvaluator Pipeline"]
        direction TB
        B1["1️⃣ Preprocess & Run Determination<br/>(For each test case)"] 
        B2["AutoPADeterminator Agent"]
        B3["2️⃣ Run Azure AI Evaluation"]
        B4["3️⃣ Postprocess Results"]
        
        B1 -.->|Invokes| B2
        B2 -.->|Generates Determination| B1
        B1 -->|Formatted Data<br/>(Generated & Ground Truth)| B3
        B3 -->|Metrics| B4
    end
    
    subgraph AgentLogic ["🧠 AutoPADeterminator Agent Logic"]
        direction TB
        C1["📥 Input: Case Data + Policy Text"]
        C2["📝 Policy Summarization<br/>(LLM Call via AzureOpenAIManager)<br/>(if policy is too long)"]
        C3["🔧 Prompt Engineering<br/>(Jinja Templates)<br/>prior_auth_user_prompt<br/>prior_auth_system_prompt"]
        C4["🤖 LLM Call for PA Determination<br/>(AzureOpenAIManager with GPT-4o/O1)"]
        C5["📄 Raw LLM Determination Text"]
        C6["🏗️ Determination Structuring<br/>(LLM Call via AzureOpenAIManager<br/>with summarize_autodetermination prompts)"]
        C7["📊 Final Structured Determination<br/>(JSON-like String)"]
        
        C1 --> C2
        C1 --> C3
        C2 --> C3
        C3 --> C4
        C4 --> C5
        C5 --> C6
        C6 --> C7
    end
    
    subgraph AzureStack ["☁️ Azure AI & Supporting Services"]
        direction TB
        D1["🧠 Azure OpenAI Service<br/>(LLMs for summarization & determination)"]
        D2["🏭 Azure AI Foundry<br/>(Project Config, Telemetry, Evaluation Logging)"]
        D3["⚙️ Custom Evaluators<br/>(RAGAS, Transformers, RapidFuzz, etc.)"]
    end
    
    E["📊 Evaluation Summary Report (JSON)"]
    
    %% Main flow connections
    A --> AutoPAD
    B2 --> AgentLogic
    B2 -.->|Uses| D1
    B3 -.->|Leverages| D2
    B3 -.->|Utilizes| D3
    B4 --> E
    
    %% Styling
    classDef inputNode fill:#e1f5fe,stroke:#0277bd,stroke-width:3px,color:#000
    classDef processNode fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#000
    classDef agentNode fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px,color:#000
    classDef azureNode fill:#fff3e0,stroke:#f57c00,stroke-width:2px,color:#000
    classDef outputNode fill:#fce4ec,stroke:#c2185b,stroke-width:3px,color:#000
    
    class A inputNode
    class B1,B3,B4 processNode
    class B2,C1,C2,C3,C4,C5,C6,C7 agentNode
    class D1,D2,D3 azureNode
    class E outputNode
    
    %% Subgraph styling
    style AutoPAD fill:#f8f9fa,stroke:#495057,stroke-width:2px
    style AgentLogic fill:#f1f8e9,stroke:#388e3c,stroke-width:2px
    style AzureStack fill:#fff8e1,stroke:#f57c00,stroke-width:2px
```

**Workflow Explanation:**
The diagram outlines the MedEvals workflow for automated prior authorization (PA) determination and its subsequent evaluation:
1.  **Test Case Input**: Evaluation starts with test cases defined in YAML, containing patient, physician, clinical information, policy texts, and ground truth determinations.
2.  **`AutoDeterminationEvaluator` Pipeline**:
    * **Preprocessing & Determination (`B1_PreProcess`)**: The evaluator loads each test case and invokes the `AutoPADeterminator` agent.
    * **`AutoPADeterminator` Agent (`B2_AutoPAD` & `B2_AutoPAD_Internals`)**:
        * Receives case data and policy text.
        * If necessary, it first summarizes long policy texts using an LLM via `AzureOpenAIManager`.
        * It then uses prompt engineering (Jinja templates) to construct detailed prompts for an Azure OpenAI model (GPT-4o or O1).
        * An LLM call generates the raw PA determination (Approved/Denied/Needs Info + rationale).
        * This raw output is further processed (potentially by another LLM call with structuring prompts) to produce a final, structured JSON-like determination.
    * **Run Azure AI Evaluation (`B3_RunEval`)**: The generated structured determinations, along with ground truth from test cases, are fed into the Azure AI Evaluation SDK. This step leverages Azure AI Foundry for logging and utilizes a suite of custom evaluators (e.g., RAGAS for factual correctness, Transformer models for semantic similarity) to score the agent's performance.
    * **Postprocess Results (`B4_PostProcess`)**: The evaluator aggregates all metrics and scores into a final JSON summary report.
3.  **Azure AI & Supporting Services**: The entire workflow relies on Azure OpenAI Service for LLM capabilities and Azure AI Foundry for evaluation infrastructure and metric logging. Custom-built evaluators enhance the assessment capabilities.
