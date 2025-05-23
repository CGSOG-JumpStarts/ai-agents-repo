# VisionAgent

## Agents / Agentic Workflows Name
- **VisionAgentV2 (Orchestrator)**: The main conversational agent that coordinates the workflow between the Planner and Coder agents. It manages user interaction, delegates tasks, and returns the final results.
- **VisionAgentPlannerV2 (Planner)**: Responsible for analyzing the user's vision task, media, and generating a multi-step plan. It utilizes sub-tools for visual understanding, tool recommendation, and can incorporate Human-in-the-Loop (HIL) for decision-making.
- **VisionAgentCoderV2 (Coder)**: Takes the plan from the Planner and generates executable Python code to solve the vision task. It also includes capabilities for generating test cases, executing them, and debugging the code based on test results.
- **Chat Application (Interface Agent)**: (e.g., `examples/chat/app.py`) Provides the user interface for interacting with the VisionAgentV2 system, handling media uploads, displaying messages, and rendering results.

## Agent / Workflow Description
VisionAgent is an AI-powered system designed to assist users in solving computer vision tasks by generating Python code. The workflow operates as follows:

1.  The user interacts with a **Chat Application**, providing a natural language prompt describing the vision task and optionally uploading media (images or videos).
2.  The Chat Application sends the user's request to the **VisionAgentV2 (Orchestrator)**.
3.  The Orchestrator first delegates the task to the **VisionAgentPlannerV2 (Planner)**.
4.  The Planner analyzes the request. It uses its internal Large Language Model (LMM) and specialized planning tools:
    *   `vqa`: For visual question answering about the provided media.
    *   `suggestion`: To generate high-level approaches or strategies.
    *   `get_tool_for_task`: To recommend specific vision tools from the VisionAgent library suitable for sub-tasks identified in the plan. This may involve:
        *   A `ToolRecommender` (Sim object) using tool docstring embeddings.
        *   A `ToolTesterLMM` to generate test code for candidate tools.
        *   A `CodeInterpreter` to run these tests.
        *   A `ToolChooserLMM` to evaluate test results and select the best tool.
        *   Optionally, engage Human-in-the-Loop (HIL) for tool selection or clarification, presenting choices back to the user via the Chat Application.
5.  The Planner formulates a multi-step plan, detailing the vision tools to be used and the logic to connect them. This plan is returned to the Orchestrator.
6.  The Orchestrator then passes this plan, along with the original task and media, to the **VisionAgentCoderV2 (Coder)**.
7.  The Coder uses its LMM to write Python code based on the plan. This code will call functions from the `vision_agent.tools` library (e.g., object detectors, OCR models, image utilities).
8.  The Coder also employs a `TesterLMM` to generate test cases for the written code.
9.  A `CodeInterpreter` executes the generated code and its test cases within a sandboxed environment.
10. If tests fail, a `DebuggerLMM` within the Coder attempts to identify and fix the bugs in the generated code. This write-test-debug cycle can iterate a few times.
11. The final, tested Python code, along with any visual outputs (like annotated images or videos produced by the code), is passed back to the Orchestrator.
12. The Orchestrator sends the results to the Chat Application, which displays the generated code and any media outputs to the user. Intermediate progress and thoughts from the agents can also be streamed to the user interface.

## Domain / Industry
-   **Computer Vision**: Core domain, providing tools and agents for various visual tasks.
-   **AI-assisted Software Development**: Focuses on generating Python code for vision applications.
-   **General Purpose Automation**: Can be applied to any industry requiring automated visual analysis, such as:
    -   Retail (e.g., product recognition, inventory management)
    -   Manufacturing (e.g., defect detection, quality control)
    -   Security and Surveillance
    -   Healthcare (e.g., medical image analysis, though specialized models would be needed)
    -   Content Moderation
    -   Autonomous Systems

## Tools / Functions Used By Agents

**VisionAgentV2 (Orchestrator):**
-   Manages conversation flow with the user.
-   Delegates tasks to Planner and Coder sub-agents.
-   Uses an LMM (configurable, e.g., Anthropic Claude Sonnet) for high-level understanding, decision-making, and response generation.
-   `update_callback`: To send intermediate messages and progress updates to the UI.

**VisionAgentPlannerV2 (Planner):**
-   **Planner LMM, Summarizer LMM, Critic LMM**: (Configurable, e.g., Anthropic Claude Sonnet) For understanding tasks, breaking them down, generating plans, summarizing information, and critiquing plans.
-   **Core Planning Tools (`vision_agent.tools.planner_tools`):**
    -   `vqa(prompt, medias)`: Visual Question Answering about media. (Uses a configured VQA LMM like Gemini).
    -   `suggestion(prompt, medias)`: Generates high-level strategic suggestions. (Uses a configured Suggester LMM like OpenAI o1).
    -   `get_tool_for_task(task, images, exclude_tools)`: Recommends vision tools. Involves:
        -   `Sim` (Tool Recommender): Semantic search over tool documentation.
        -   Tool Tester LMM: Generates code to test candidate tools.
        -   Code Interpreter: Executes tool test code.
        -   Tool Chooser LMM: Evaluates tool outputs and selects the best one.
        -   `get_tool_for_task_human_reviewer`: (If HIL enabled) Presents choices to the user.
    -   `judge_od_results(prompt, image, detections)`: LMM-based evaluation of object detection results.
-   **Media Utilities (from `vision_agent.tools.tools` called during planning):**
    -   `load_image(image_path)`
    -   `extract_frames_and_timestamps(video_uri, fps)`
-   **Code Interpreter (`CodeInterpreterFactory.new_instance()`):** To execute Python code snippets, especially for tool testing during the planning phase.

**VisionAgentCoderV2 (Coder):**
-   **Coder LMM, Tester LMM, Debugger LMM**: (Configurable, e.g., Anthropic Claude Sonnet) For writing Python code, generating test cases, and fixing bugs.
-   **Generated Code Utilizes Vision Tools Library (`vision_agent.tools.tools`):**
    -   **Object Detection**: `owlv2_object_detection`, `florence2_object_detection`, `countgd_object_detection`, `agentic_object_detection`, `glee_object_detection`, `custom_object_detection`.
    -   **Instance Segmentation**: `sam2` (Segment Anything Model 2), `owlv2_sam2_instance_segmentation`, `florence2_sam2_instance_segmentation`, etc.
    -   **Video Processing**: `*_video_tracking` (e.g., `countgd_sam2_video_tracking`), `agentic_activity_recognition`.
    -   **OCR/Document Understanding**: `ocr`, `florence2_ocr`, `claude35_text_extraction`, `agentic_document_extraction`, `document_qa`.
    -   **VQA**: `qwen25_vl_images_vqa`, `qwen25_vl_video_vqa`, `qwen2_vl_images_vqa`, `qwen2_vl_video_vqa`.
    -   **Image Classification**: `vit_image_classification`, `vit_nsfw_classification`, `siglip_classification`.
    -   **Image Generation/Editing**: `gemini_image_generation`.
    -   **Depth/Pose Estimation**: `depth_anything_v2`, `generate_pose_image`.
    -   **Other Vision Primitives**: `template_match`, `minimum_distance`.
-   **General Utility Functions (called by generated code, from `vision_agent.tools.tools`):**
    -   `load_image(image_path)`
    -   `extract_frames_and_timestamps(video_uri, fps)`
    -   `save_image(image, file_path)`
    -   `save_video(frames, output_video_path, fps)`
    -   `save_json(data, file_path)`
    -   `overlay_bounding_boxes(medias, bboxes)`
    -   `overlay_segmentation_masks(medias, masks)`
    -   `overlay_heat_map(image, heat_map)`
-   **Code Interpreter (`CodeInterpreterFactory.new_instance()`):** To execute the generated vision code and its test cases in a sandboxed Python environment (likely Jupyter client based).

**Chat Application specific tools (from `examples/chat/app.py`):**
-   WebSocket communication for real-time updates.
-   FastAPI for backend server.
-   File handling for media uploads (images, videos).
-   Polygon drawing and annotation tools in the UI (`PolygonDrawer.tsx`).
-   Result visualization components (`GroupedVisualizer.tsx`, `ImageVisualizer.tsx`, `VideoVisualizer.tsx`).

**External Services / APIs (as per tool implementations):**
-   **LandingAI API**: Primary backend for many vision tools (e.g., CountGD, Florence2, SAM2, OwlV2, Agentic OD, GLEE, custom models).
-   **LLM Provider APIs**: Anthropic, Google (Gemini), OpenAI, etc., depending on the LMMs configured in `vision_agent.configs.Config` for different agent roles (Planner, Coder, VQA, Suggester).
-   **Ollama**: Can be configured as an LMM backend.
-   **Azure OpenAI**: Can be configured as an LMM backend.

## Architecture Design

```mermaid
graph TD
    UserInterface[User Interface (Web App / CLI)] -->|User Prompt, Media| VisionAgentV2[VisionAgentV2 Orchestrator]

    subgraph VisionAgentV2 Orchestrator
        direction LR
        Planner[VisionAgentPlannerV2]
        Coder[VisionAgentCoderV2]
    end

    VisionAgentV2 -- "1. Plan Task" --> Planner
    Planner -- "2. Generate Plan" --> PlannerLMM[Planner LMM Engine]
    PlannerLMM -- "Uses" --> PlanningTools[Planning Helper Tools (VQA, Suggestion, Tool Selector)]
    PlanningTools -- "Accesses" --> VisionToolDocs[Vision Tools Docstrings & Knowledge]
    PlanningTools -- "Human-in-the-Loop (Optional)" --> UserInterface
    Planner --> |"Returns Plan"| VisionAgentV2

    VisionAgentV2 -- "3. Code Generation Task + Plan" --> Coder
    Coder -- "4. Generate Code & Tests" --> CoderLMMs[Coder, Tester, Debugger LMMs]
    CoderLMMs -- "Writes code that uses" --> VisionToolsLib[Vision Tools Library (tools.py)]
    CoderLMMs -- "Code & Tests" --> CodeExecutionEnv[Code Execution Environment (CodeInterpreter)]
    CodeExecutionEnv -- "Executes & Gets Results/Errors" --> CoderLMMs
    Coder --> |"Returns Final Code & Output"| VisionAgentV2

    VisionAgentV2 -- "5. Display Results" --> UserInterface

    %% External Services and Dependencies
    PlannerLMM --> LLM_APIs[LLM Provider APIs (Anthropic, Google, OpenAI, etc.)]
    CoderLMMs --> LLM_APIs
    PlanningTools -- "Uses (for VQA, Suggestion, Tool Choice)" --> LLM_APIs
    VisionToolsLib -- "Calls for CV tasks" --> CV_Backends[CV Model Backends (LandingAI API, other specific model APIs)]

    classDef agent fill:#DDA0DD,stroke:#333,stroke-width:2px;
    classDef module fill:#ADD8E6,stroke:#333,stroke-width:2px;
    classDef lmm fill:#90EE90,stroke:#333,stroke-width:2px;
    classDef tool fill:#FFB6C1,stroke:#333,stroke-width:2px;
    classDef external fill:#F0E68C,stroke:#333,stroke-width:2px;

    class VisionAgentV2,Planner,Coder agent;
    class PlannerLMM,CoderLMMs,PlanningTools,VisionToolDocs,VisionToolsLib,CodeExecutionEnv module;
    class LLM_APIs,CV_Backends,UserInterface external;
```

The architecture shows VisionAgentV2 as a central orchestrator. It first employs the Planner (VisionAgentPlannerV2) which uses an LMM and specialized planning tools (like VQA and tool recommenders) to create a step-by-step plan. This plan can optionally involve human feedback. The plan is then passed to the Coder (VisionAgentCoderV2), which uses LMMs to generate Python code, test it using a CodeInterpreter, and debug it iteratively. The generated code calls upon a library of vision tools, many of which interact with backend services like the LandingAI API. The final code and results are then relayed back to the user through the UI.
