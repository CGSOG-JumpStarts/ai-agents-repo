# 1. RedTeaming Framework

## Agents / Agentic Workflows Name

-   **RedTeaming Orchestrator**: The main `RedTeaming` class that manages the red teaming process.
-   **ScenarioGenerator Agent**: Generates safety/security scenarios or requirements based on agent description and risk category.
-   **TestCaseGenerator Agent**: Generates adversarial test cases for each scenario.
-   **Evaluator Agent**: Evaluates the agent's response against the generated scenarios.
-   **LLMGenerator Tool**: A utility used by the above agents to interact with various Large Language Models (LLMs).

## Agent / Workflow Description

This framework automates the red teaming process for AI agents to identify vulnerabilities, biases, and potential misuses. The workflow is as follows:

1.  The **RedTeaming Orchestrator** is initialized with LLM configurations (model, provider, API keys, temperatures).
2.  The user calls the `run()` method, providing:
    *   A description of the AI agent to be tested.
    *   A list of `detectors` (risk categories like "stereotypes", "harmful_content", or custom descriptions).
    *   A `response_model` function, which is the AI agent under test.
    *   Optionally, `examples` (user-defined test inputs) or a `model_input_format` (for auto-generating test cases).
3.  For each specified `detector`:
    *   The **ScenarioGenerator Agent** is invoked. It uses the `LLMGenerator Tool` to call an LLM (e.g., GPT-4, Grok-2) with the agent's description and the current detector's issue description (obtained via `get_issue_description`). The LLM generates a set of specific scenarios/requirements that the agent should adhere to for that risk category.
    *   For each generated `scenario`:
        *   If user `examples` are not provided, the **TestCaseGenerator Agent** is invoked. It uses the `LLMGenerator Tool` to call an LLM, providing the agent description, category, current scenario, and desired input format. The LLM generates a number of adversarial test case inputs.
        *   The `response_model` (the agent being tested) is then called with each generated test case (or each user-provided example relevant to the detector).
        *   The **Evaluator Agent** takes the original agent description, the conversation (test case + agent's response), and the current scenario. It uses the `LLMGenerator Tool` to call an LLM, which assesses whether the agent's response met the requirements of the scenario. The evaluation yields a pass/fail and a reason.
4.  All results (detector, scenario, test message, agent response, evaluation score, and reason) are compiled into a Pandas DataFrame.
5.  The results DataFrame is saved as a CSV file with a timestamped name.
6.  Optionally, the results can be uploaded to the RagaAI Catalyst platform using the `UploadResult` utility.

## Domain / Industry

-   AI Safety and Security
-   AI Testing and Evaluation
-   LLM Red Teaming
-   General (applicable to AI agents across various domains)

## Tools / Functions Used By Agents

### RedTeaming Orchestrator (`RedTeaming` class):
-   `run()`: Main entry point to start the red teaming process.
-   `_run_with_examples()`: Handles execution when user provides example test cases.
-   `_run_without_examples()`: Handles execution when test cases need to be auto-generated.
-   `validate_detectors()`: Checks if provided detectors are supported.
-   `get_issue_description()`: Retrieves detailed descriptions for built-in detectors.
-   `_save_results_to_csv()`: Saves the final report.
-   `upload_result()`: Uploads results to RagaAI platform.
-   **Dependencies**: `ScenarioGenerator`, `TestCaseGenerator`, `Evaluator`, `LLMGenerator`, `UploadResult`, `pandas`, `tomli`, `tqdm`, `rich`.

### ScenarioGenerator Agent (`ScenarioGenerator` class):
-   `generate_scenarios()`: Creates a list of requirements/scenarios for an AI agent.
    -   `_create_input_template()`: Formats the input prompt for the LLM.
    -   `_validate_scenarios()`: Validates and normalizes the LLM's output.
-   **Tool Used**: `LLMGenerator` (to call an LLM like GPT-4 or Grok-2).

### TestCaseGenerator Agent (`TestCaseGenerator` class):
-   `generate_test_cases()`: Creates adversarial inputs (test cases).
    -   `_create_input_template()`: Formats the input prompt for the LLM.
    -   `_validate_test_cases()`: Validates the LLM's output against the expected format.
-   **Tool Used**: `LLMGenerator` (to call an LLM like GPT-4 or Grok-2).

### Evaluator Agent (`Evaluator` class):
-   `evaluate_conversation()`: Assesses if an agent's conversation meets scenario requirements.
    -   `_create_input_template()`: Formats the input prompt for the LLM.
    -   `_validate_evaluation()`: Validates the LLM's evaluation output.
-   **Tool Used**: `LLMGenerator` (to call an LLM like GPT-4 or Grok-2).

### LLMGenerator Tool (`LLMGenerator` class in `llm_generator.py`):
-   `generate_response()`: Generic function to get a JSON response from a configured LLM.
    -   `_validate_api_key()`, `_validate_azure_keys()`: Ensures API credentials are set.
    -   `get_xai_response()`: Specific handler for XAI models.
-   **External Services**:
    -   OpenAI API (e.g., GPT-4)
    -   XAI API (e.g., Grok-2)
    -   Any LLM supported by `litellm`.

### UploadResult (`UploadResult` class):
-   `upload_result()`: Uploads the CSV results to RagaAI Catalyst.
-   **Tool Used**: `ragaai_catalyst.Dataset` for interacting with the RagaAI platform.

## Architecture Design

```mermaid
graph TD
    User[User] -->|Agent Desc, Detectors, Response Model Func, Test Config| RTOrchestrator[RedTeaming Orchestrator]

    subgraph RedTeamingFramework["Red Teaming Framework"]
        RTOrchestrator
        ScenarioGen[ScenarioGenerator Agent]
        TestCaseGen[TestCaseGenerator Agent]
        EvalAgent[Evaluator Agent]
        LLMGenTool[LLMGenerator Tool]
        UploadUtil[UploadResult Utility]
    end

    AgentUnderTest[Agent Under Test (Provided by User)]
    LLMAPIs["LLM APIs (OpenAI, XAI, LiteLLM)"]
    RagaPlatform["RagaAI Catalyst Platform"]
    LocalFS["Local Filesystem (CSV Report)"]

    RTOrchestrator -->|Agent Desc, Detector Category| ScenarioGen
    ScenarioGen -->|Prompts| LLMGenTool
    LLMGenTool -->|API Call| LLMAPIs
    LLMAPIs -->|Generated Scenarios JSON| LLMGenTool
    LLMGenTool -->|Parsed Scenarios| ScenarioGen
    ScenarioGen -->|Scenarios List| RTOrchestrator

    RTOrchestrator -- If no examples -->|Agent Desc, Category, Scenario, Input Format| TestCaseGen
    TestCaseGen -->|Prompts| LLMGenTool
    LLMGenTool -->|API Call| LLMAPIs
    LLMAPIs -->|Generated Test Cases JSON| LLMGenTool
    LLMGenTool -->|Parsed Test Cases| TestCaseGen
    TestCaseGen -->|Adversarial Inputs| RTOrchestrator

    RTOrchestrator -->|Test Input / User Example| AgentUnderTest
    AgentUnderTest -->|Agent Response| RTOrchestrator

    RTOrchestrator -->|Agent Desc, Conversation, Scenario| EvalAgent
    EvalAgent -->|Prompts| LLMGenTool
    LLMGenTool -->|API Call| LLMAPIs
    LLMAPIs -->|Evaluation JSON (Pass/Fail, Reason)| LLMGenTool
    LLMGenTool -->|Parsed Evaluation| EvalAgent
    EvalAgent -->|Evaluation Outcome| RTOrchestrator

    RTOrchestrator -->|Save Report| LocalFS
    RTOrchestrator -->|Upload Report (Optional)| UploadUtil
    UploadUtil -->|API Call| RagaPlatform

```

# 2. Agentic Tracing Framework

## Agents / Agentic Workflows Name

-   **AgenticTracing Orchestrator**: The main `AgenticTracing` class that initializes and manages the tracing process.
-   **LLMTracer**: A component (mixin) responsible for tracing Large Language Model (LLM) calls.
-   **ToolTracer**: A component (mixin) for tracing calls to tools or standard functions.
-   **AgentTracer**: A component (mixin) for tracing the execution of designated "agent" functions or classes, managing hierarchical relationships.
-   **CustomTracer**: A component (mixin) for tracing arbitrary custom Python functions, with an option for variable state tracing.
-   **NetworkTracer**: A dedicated component that patches network libraries to capture HTTP/socket calls.
-   **UserInteractionTracer**: A component that patches built-in I/O (print, input) and file operations (open) to record interactions.
-   **SystemMonitor**: A utility that runs in the background to collect system (OS, environment) and resource (CPU, memory, disk, network) metrics.
-   **TraceUploader**: An asynchronous utility responsible for uploading the collected trace data and zipped source code to the RagaAI Catalyst platform.

## Agent / Workflow Description

The Agentic Tracing Framework is designed to provide deep observability into the execution of AI agent systems. It captures detailed information about LLM calls, tool usage, agent interactions, network activity, user I/O, and system resource consumption.

1.  **Initialization**: The user initializes the `AgenticTracing` orchestrator (often via a simpler `Tracer` class wrapper), providing project and dataset names, and optionally configuring which components to auto-instrument (LLMs, tools, agents, network, I/O).
2.  **Tracing Start (`start()` method)**:
    *   A unique trace ID is generated.
    *   The `SystemMonitor` starts collecting OS/environment information and begins background monitoring of CPU, memory, disk, and network usage at specified intervals.
    *   Based on `auto_instrumentation` settings, the framework patches relevant libraries:
        *   `LLMTracer`: Patches common LLM SDKs (OpenAI, Anthropic, Google GenAI, VertexAI, LiteLLM, LangChain LLMs, LlamaIndex LLMs) to intercept API calls.
        *   `ToolTracer`: Patches LangChain tool execution methods.
        *   `NetworkTracer`: Patches `requests`, `urllib`, `http.client`, `socket` to capture network calls.
        *   `UserInteractionTracer`: Patches `builtins.print`, `builtins.input`, `builtins.open` for I/O and file access.
    *   The user can also manually instrument specific functions/classes using decorators like `@trace_agent`, `@trace_llm`, `@trace_tool`, `@trace_custom`.
3.  **During Application Execution**:
    *   When a patched/decorated function is called, the respective tracer (`LLMTracer`, `ToolTracer`, `AgentTracer`, `CustomTracer`) intercepts the call.
    *   It records start time, input arguments, and other relevant metadata (e.g., model name for LLMs, tool type for tools).
    *   `AgentTracer` manages a context to build a hierarchy of calls, noting parent-child relationships between agents and their sub-components (other agents, LLMs, tools).
    *   `NetworkTracer` and `UserInteractionTracer` capture their respective events (network requests, print outputs, file reads/writes) and associate them with the currently active component (agent, tool, LLM, or custom function) if `auto_instrument_network` or `auto_instrument_file_io`/`auto_instrument_user_interaction` are enabled.
    *   Upon completion (or error) of the traced function, end time, output (or error details), and resource usage (memory) are recorded.
    *   A "component" data structure is created for each traced call, containing all captured information, including a unique hash of the function's source code (for agents/tools/custom).
4.  **Adding Span Attributes**: Users can enrich spans by adding tags, metadata, custom metrics, ground truth (`gt`), and context using `tracer.span("span_name").add_...()` methods.
5.  **Tracing Stop (`stop()` method)**:
    *   All monkey patches are reverted. Background monitoring by `SystemMonitor` is stopped.
    *   The collected components (spans), along with aggregated system/resource metrics, network calls, user interactions, and trace-level metadata (total cost, total tokens), are compiled into a single `Trace` object.
    *   The `FileTracker` identifies all unique source code files involved in the trace. These files (excluding virtual environment and RagaAI Catalyst library files) are packaged into a zip archive by `zip_list_of_unique_files`. A hash of the combined source code is generated.
    *   The complete `Trace` object is serialized to a JSON file in a temporary directory.
    *   If a `post_processor` function is registered, it's applied to the trace JSON file (e.g., for data masking).
    *   The `TraceUploader` utility is invoked to asynchronously upload the trace JSON file and the code zip archive to the RagaAI Catalyst platform.

## Domain / Industry

-   AI/LLM Observability & Monitoring
-   AI Agent Development & Debugging
-   MLOps for Agentic Systems
-   General (applicable to developing and monitoring any AI agent-based application)

## Tools / Functions Used By Agents

### AgenticTracing Orchestrator (`AgenticTracing` class):
-   `start()`: Initiates tracing, activates monitoring, and applies patches.
-   `stop()`: Finalizes and saves the trace, and queues it for upload.
-   `add_component()`: Adds a captured component (span) to the trace data.
-   `register_post_processor()`: Allows custom functions to modify the trace JSON before upload.
-   `instrument_llm_calls()`, `instrument_tool_calls()`, etc.: Methods to enable/disable auto-instrumentation for different component types.
-   **Internal Utilities**: `NetworkTracer`, `UserInteractionTracer`, `SystemMonitor`, `TrackName` (for file tracking), `zip_list_of_unique_files` (for code packaging), `SpanAttributes` (for managing span-specific custom data).
-   **Mixins**: Leverages `LLMTracerMixin`, `ToolTracerMixin`, `AgentTracerMixin`, `CustomTracerMixin` for specific tracing logic.

### LLMTracer (`LLMTracerMixin`):
-   `trace_llm()`: Decorator for manual LLM call tracing.
-   `trace_llm_call_sync()` / `trace_llm_call()`: Wraps LLM calls, extracts metadata (model, parameters, input/output, token usage, cost).
-   Patches methods in `openai`, `litellm`, `anthropic`, `google.generativeai`, `vertexai`, `langchain_community.llms`, `langchain_openai`, etc.
-   Uses `llm_utils` for extracting information from LLM responses and calculating costs.

### ToolTracer (`ToolTracerMixin`):
-   `trace_tool()`: Decorator for manual tool/function tracing.
-   `_trace_sync_tool_execution()` / `_trace_tool_execution()`: Wraps tool calls, records inputs/outputs, memory usage.
-   Patches LangChain tool execution methods (e.g., `BaseTool.run`, `StructuredTool.invoke`).

### AgentTracer (`AgentTracerMixin`):
-   `trace_agent()`: Decorator for tracing agent functions/classes.
-   `_trace_sync_agent_execution()` / `_trace_agent_execution()`: Wraps agent calls, manages parent-child call hierarchy using `contextvars`, records agent-specific info.

### CustomTracer (`CustomTracerMixin`):
-   `trace_custom()`: Decorator for tracing generic Python functions.
-   `_trace_sync_custom_execution()` / `_trace_custom_execution()`: Wraps custom function calls, optionally traces local variable states.

### NetworkTracer:
-   `activate_patches()` / `deactivate_patches()`: Manages monkey-patching of network libraries.
-   `record_call()`: Logs network request details (URL, method, status, headers, body, timing).
-   **Patched Libraries**: `requests`, `urllib.request`, `http.client`, `socket`.

### UserInteractionTracer:
-   `traced_input()`, `traced_print()`, `traced_open()`: Wrappers for built-in I/O functions.
-   `trace_file_operation()`: Records details of file read/write operations.

### SystemMonitor:
-   `get_system_info()`: Gathers OS, Python environment, and installed package information.
-   `get_resources()`: Captures initial CPU, memory, disk, and network statistics.
-   `track_memory_usage()`, `track_cpu_usage()`, etc.: Methods called periodically by background threads to log resource usage over time.
-   **Dependencies**: `platform`, `psutil`, `pkg_resources`.

### TraceUploader (`trace_uploader.py`):
-   `submit_upload_task()`: Queues a trace file and associated code zip for upload using a `ThreadPoolExecutor`.
-   `process_upload()`: Handles the multi-step upload to RagaAI Catalyst:
    1.  `create_dataset_schema_with_trace()`: Ensures the dataset schema exists on the platform.
    2.  `upload_trace_metric()`: Uploads any trace-level or span-level custom metrics.
    3.  `UploadAgenticTraces.upload_agentic_traces()`: Uploads the main trace JSON data.
    4.  `upload_code()`: Uploads the zipped source code.
-   **Dependencies**: `requests`.

## Architecture Design

```mermaid
graph TD
    UserApplication["User's AI Application Code"] -- Interacts with --> PatchedOrDecoratedCode["Patched Libraries / Decorated Functions"]

    subgraph AgenticTracingFramework ["Agentic Tracing Framework"]
        direction LR
        MainTracer["AgenticTracing Orchestrator (main_tracer.py)"]

        subgraph CoreTracers ["Core Tracing Logic (Mixins)"]
            direction TB
            LLMTracerMixin["LLMTracerMixin"]
            ToolTracerMixin["ToolTracerMixin"]
            AgentTracerMixin["AgentTracerMixin"]
            CustomTracerMixin["CustomTracerMixin"]
        end

        subgraph UtilityTracers ["Utility Tracers & Monitors"]
            direction TB
            NetworkTracerUtil["NetworkTracer"]
            UserInteractionTracerUtil["UserInteractionTracer"]
            SystemMonitorUtil["SystemMonitor"]
        end
        
        FileHandling["File & Code Management"]
        TraceDataManagement["Trace Data & Component Management"]
        AsyncUploader["TraceUploader (Background Process)"]
    end

    subgraph ExternalDependencies ["External Dependencies & Services"]
        PythonBuiltins["Python Built-ins (print, input, open)"]
        LLMLibraries["LLM SDKs (OpenAI, Anthropic, etc.)"]
        NetworkLibraries["Network Libraries (requests, http.client)"]
        LangChainCore["LangChain Core/Tools (if used)"]
        HostSystem["Host System (OS, CPU, Memory)"]
        RagaPlatform["RagaAI Catalyst Platform"]
    end

    UserApplication -- Calls & Interactions --> MainTracer

    MainTracer -- Manages & Uses --> CoreTracers
    MainTracer -- Manages & Uses --> UtilityTracers
    MainTracer -- Uses --> FileHandling
    MainTracer -- Manages --> TraceDataManagement
    MainTracer -- "Trace JSON, Code ZIP" --> AsyncUploader

    CoreTracers -- Intercept & Wrap --> PatchedOrDecoratedCode
    NetworkTracerUtil -- Patches --> NetworkLibraries
    UserInteractionTracerUtil -- Patches --> PythonBuiltins

    PatchedOrDecoratedCode -- Original Call --> LLMLibraries
    PatchedOrDecoratedCode -- Original Call --> LangChainCore
    
    SystemMonitorUtil -- Monitors --> HostSystem
    FileHandling -- Reads --> UserApplication # To get source code
    
    AsyncUploader -- Uploads Data --> RagaPlatform
    
    %% Styling
    classDef orchestrator fill:#f9f,stroke:#333,stroke-width:2px;
    classDef mixin fill:#lightcoral,stroke:#333,stroke-width:1px;
    classDef utility fill:#lightblue,stroke:#333,stroke-width:1px;
    classDef external fill:#lightgreen,stroke:#333,stroke-width:1px;

    class MainTracer orchestrator;
    class CoreTracers,UtilityTracers,FileHandling,TraceDataManagement,AsyncUploader utility;
    class PatchedOrDecoratedCode,PythonBuiltins,LLMLibraries,NetworkLibraries,LangChainCore,HostSystem,RagaPlatform external;
```

# 3. Synthetic Data Generation Framework

## Agents / Agentic Workflows Name

-   **SyntheticDataGenerator (Orchestrator)**: The main `SyntheticDataGeneration` class that orchestrates the data generation processes.
-   **QnAGenerator Agent**: An implicit agent within the `generate_qna` method, responsible for creating question-answer pairs.
-   **ExampleGenerator Agent**: An implicit agent within the `generate_examples` and `generate_examples_from_csv` methods, responsible for creating diverse examples based on instructions and feedback.
-   **LLM Interaction Component**: Internal methods (`_generate_llm_response`, `_generate_internal_response`, `_generate_raw_llm_response`) that handle communication with LLMs.

## Agent / Workflow Description

This framework provides tools for generating synthetic data, such as question-answer pairs and varied examples, leveraging Large Language Models (LLMs). It supports multiple LLM providers and can process various document types as input.

**Core Workflows:**

1.  **Question-Answer (Q&A) Generation (`generate_qna` method):**
    *   The user provides input `text` (or a path to a document), the desired `question_type` ('simple', 'mcq', 'complex'), the number of Q&A pairs `n`, and `model_config`.
    *   The `SyntheticDataGenerator` validates the input text (e.g., checks for emptiness, minimum token length using `tiktoken`).
    *   It initializes an LLM client based on `model_config` (supports Groq, Gemini, OpenAI, Azure via LiteLLM, or a custom internal proxy).
    *   To ensure quality and manage API limits, Q&A pairs are generated in batches (default batch size of 5).
    *   For each batch, a system prompt tailored to the `question_type` and batch size is constructed using `_get_system_message`.
    *   The `_generate_batch_response` (or `_generate_internal_response` if an internal proxy is used) method calls the LLM (via `_generate_llm_response` which uses `litellm.completion`) with the system prompt and input text.
    *   The LLM's JSON response (a list of Q&A objects) is parsed into a Pandas DataFrame.
    *   If generation fails (e.g., API errors, invalid JSON), retries are attempted. Catastrophic API key errors will halt the process.
    *   After initial batch generation, responses are aggregated, and duplicates (based on 'Question') are removed.
    *   If the number of unique Q&A pairs is less than `n`, a replenishment phase begins, generating more Q&A pairs to meet the target `n`, while avoiding existing questions.
    *   The final output is a Pandas DataFrame containing exactly `n` unique Q&A pairs.

2.  **Interactive Example Generation (`generate_examples` method):**
    *   The user provides a `user_instruction`, optional `user_examples` (seed examples), optional `user_context`, the desired `no_examples`, `model_config`, and `max_iter` for refinement.
    *   The system enters an iterative loop (up to `max_iter` times):
        *   It generates a set of examples using `_generate_examples` (initial call) or `_generate_examples_iter` (subsequent calls). These methods use `_generate_raw_llm_response` which calls an LLM with tailored system prompts (`_get_init_ex_gen_prompt` or `_get_iter_ex_gen_prompt`).
        *   The generated examples are printed to the console.
        *   The user is prompted (via `input()`) to provide comma-separated indices of relevant and irrelevant examples from the generated set.
        *   This feedback (lists of relevant and irrelevant examples) is used in the next iteration to refine the LLM's generation.
    *   The loop continues until `max_iter` is exhausted or sufficient relevant examples are collected.
    *   If, after iterations, the number of collected relevant examples is less than `no_examples`, a final generation step is performed to fill the gap.
    *   Returns a list of the final generated examples.

3.  **Example Generation from CSV (`generate_examples_from_csv` method):**
    *   The user provides a `csv_path`, `dst_csv_path` (optional), `no_examples`, and `model_config`.
    *   The input CSV must contain a `user_instruction` column, and can optionally have `user_examples` and `user_context` columns.
    *   For each row in the input CSV:
        *   It calls the `generate_examples` method with the data from the current row.
        *   The list of generated examples is added as a new column (or multiple rows if each example is on a new row) to the original row's data.
    *   The augmented DataFrame is saved to `dst_csv_path`.

**Helper Functionality:**
-   `process_document()`: Takes a file path or text string. If it's a path, it reads PDF, TXT, Markdown, or CSV files and extracts their text content.
-   `_initialize_client()`: Configures API clients for various LLM providers (Groq, Gemini, OpenAI, Azure).

## Domain / Industry

-   Synthetic Data Generation
-   AI Data Augmentation & Preparation
-   Test Data Creation for LLM Applications
-   Prompt Engineering Assistance
-   General LLM Application Development

## Tools / Functions Used By Agents

### SyntheticDataGenerator (Orchestrator):
-   `generate_qna()`: Orchestrates question-answer pair generation.
    -   Calls `_initialize_client()`, `validate_input()`, `_get_system_message()`, `_generate_batch_response()`.
-   `generate_examples()`: Orchestrates interactive example generation.
    -   Calls `_initialize_client()`, `_generate_examples()`, `_generate_examples_iter()`, `_get_valid_examples()`.
-   `generate_examples_from_csv()`: Reads a CSV and calls `generate_examples()` for each row.
-   `process_document()`: Handles input document reading.
    -   Calls `_read_pdf()`, `_read_text()`, `_read_markdown()`, `_read_csv()`.
-   `_generate_llm_response()`: Primary interface to `litellm.completion` for external LLM calls.
-   `_generate_internal_response()`: Interface to an internal LLM proxy via `internal_api_completion`.
-   `_generate_raw_llm_response()`: A lower-level wrapper for LLM calls used by example generation.
-   **Dependencies**: `openai`, `tiktoken`, `litellm`, `groq`, `pypdf`, `markdown`, `pandas`, `tqdm`.

### LLM Interaction Components:
-   `litellm.completion`: Used for making calls to various external LLMs (OpenAI, Groq, Gemini, Azure).
-   `internal_api_completion` (from `.internal_api_completion`): Used if `internal_llm_proxy` is configured in `generate_qna`.
-   `proxy_api_completion` (from `.proxy_call`): Used if `api_base` is provided for Gemini in `generate_qna`.

### External Services / Libraries:
-   LLM APIs: OpenAI, Groq, Google Gemini, Azure OpenAI Service (via LiteLLM).
-   Custom/Internal LLM Proxies (if configured).
-   `pypdf`: For reading PDF files.
-   `markdown`: For processing Markdown files.
-   `pandas`: For handling and returning Q&A data as DataFrames, and for CSV processing.
-   `tiktoken`: For validating input text length based on tokens.

## Architecture Design

```mermaid
graph TD
    UserRequest[User Request] -->|Input (Text/File), Config (Type, N, Model)| SDG[SyntheticDataGeneration Class]

    subgraph SDG_InternalWorkflow["SDG Internal Workflow"]
        direction LR
        SDG_Orchestrator["SDG Orchestrator Methods\n(generate_qna, generate_examples, etc.)"]
        DocParser["process_document()"]
        PromptBuilder["Prompt Construction\n(_get_system_message, _get_init_ex_gen_prompt, etc.)"]
        LLMInterface["LLM Call Interface\n(_generate_llm_response, _generate_internal_response)"]
        ResponseParser["Response Parsing & Formatting\n(_parse_response, DataFrame creation)"]
        FeedbackLoop["Interactive Feedback Loop (for generate_examples)"]
    end

    subgraph LLM_Access_Layer["LLM Access Layer"]
        LiteLLM["litellm.completion"]
        InternalProxyCall["internal_api_completion"]
        CustomProxyCall["proxy_api_completion"]
    end

    subgraph External_LLMs_And_Services["External LLMs & Services"]
        OpenAI_API["OpenAI API"]
        Groq_API["Groq API"]
        Gemini_API["Google Gemini API"]
        Azure_OpenAI_API["Azure OpenAI API"]
        OtherLiteLLMLinkedAPIs["Other LiteLLM-Supported APIs"]
        InternalLLMService["Internal LLM Service (Optional)"]
    end
    
    subgraph SupportingLibraries["Supporting Libraries"]
        PandasLib["pandas"]
        TiktokenLib["tiktoken"]
        PyPDFLib["pypdf"]
        MarkdownLib["markdown (Python library)"]
    end

    SDG_Orchestrator -- Uses --> DocParser
    DocParser -- Uses --> PyPDFLib
    DocParser -- Uses --> MarkdownLib
    DocParser -->|Text Content| SDG_Orchestrator
    
    SDG_Orchestrator -- Uses --> PromptBuilder
    PromptBuilder -->|System/User Prompts| SDG_Orchestrator

    SDG_Orchestrator -- Calls --> LLMInterface
    LLMInterface -- Selects & Calls --> LiteLLM
    LLMInterface -- Optionally Calls --> InternalProxyCall
    LLMInterface -- Optionally Calls --> CustomProxyCall
    
    LiteLLM -- Routes to --> OpenAI_API
    LiteLLM -- Routes to --> Groq_API
    LiteLLM -- Routes to --> Gemini_API
    LiteLLM -- Routes to --> Azure_OpenAI_API
    LiteLLM -- Routes to --> OtherLiteLLMLinkedAPIs
    
    InternalProxyCall -- Calls --> InternalLLMService
    CustomProxyCall -- Calls --> Gemini_API # Or other specified proxy endpoint

    OpenAI_API -->|LLM Response| LiteLLM
    Groq_API -->|LLM Response| LiteLLM
    Gemini_API -->|LLM Response| LiteLLM # or CustomProxyCall
    Azure_OpenAI_API -->|LLM Response| LiteLLM
    OtherLiteLLMLinkedAPIs -->|LLM Response| LiteLLM
    InternalLLMService -->|LLM Response| InternalProxyCall
    
    LiteLLM -->|Raw LLM Response| LLMInterface
    InternalProxyCall -->|Raw LLM Response| LLMInterface
    CustomProxyCall -->|Raw LLM Response| LLMInterface

    LLMInterface -->|Parsed Response (String/JSON)| ResponseParser
    ResponseParser -- Uses --> PandasLib
    ResponseParser -->|Structured Data (e.g., DataFrame)| SDG_Orchestrator
    
    SDG_Orchestrator -- generate_examples() uses --> FeedbackLoop
    FeedbackLoop -- User Input via console --> SDG_Orchestrator # For interactive refinement

    SDG_Orchestrator -- Input Validation uses --> TiktokenLib
    SDG_Orchestrator -->|Final Output (DataFrame/List/CSV)| UserOutput["User Output"]
```
