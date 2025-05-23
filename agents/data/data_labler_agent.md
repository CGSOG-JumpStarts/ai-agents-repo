# Autonomous Data Labeling Agent Framework

## Agents / Agentic Workflows Name

Adala is a framework for creating autonomous data processing agents. The core components and concepts that define its agentic nature include:

* **Adala Agent (`adala.agents.base.Agent`)**: The central orchestrator that manages the learning and application of skills. It interacts with environments, utilizes runtimes (LLMs), and can leverage memory.
* **SkillSet (`adala.skills.skillset.SkillSet`)**: A collection of one or more skills that the agent possesses and applies. Skillsets can be:
    * **`LinearSkillSet`**: Skills are applied sequentially, with the output of one skill potentially feeding into the next.
    * **`ParallelSkillSet`**: Skills are applied independently to the input data.
* **Skills (`adala.skills._base.Skill` and `adala.skills.collection.*`)**: These are the functional units that perform specific data processing tasks. They are not independent agents but rather capabilities wielded by the Adala Agent. Examples include:
    * `ClassificationSkill`
    * `EntityExtraction`
    * `SummarizationSkill`
    * `TextGenerationSkill`
    * `QuestionAnsweringSkill`
    * `TranslationSkill`
    * `RAGSkill` (Retrieval Augmented Generation)
    * `LabelStudioSkill` (for integration with Label Studio)
    * `OntologyCreator` / `OntologyMerger`
    * `PromptImprovementSkill` (a meta-skill to refine other skills' prompts)
* **Learning Loop**: Adala agents are designed to learn and improve their skills iteratively based on feedback from an environment. This often involves a "teacher" runtime (a more capable LLM) guiding the refinement of a "student" runtime's skill.

## Agent / Workflow Description

Adala is a framework for building autonomous agents specialized in data processing and labeling tasks. These agents learn and apply "skills" (powered by LLMs via "runtimes") by interacting with "environments" that provide data and ground truth feedback.

The general workflow of an Adala agent involves two main phases: learning and application.

**1. Learning Phase (`agent.learn()`):**
    a.  **Data Acquisition**: The agent retrieves a batch of data from its configured `Environment` (e.g., `StaticEnvironment` with a DataFrame, `ConsoleEnvironment` for human input, `AsyncKafkaEnvironment` for streaming).
    b.  **Skill Application**: The agent applies its `SkillSet` to the data batch. For a `LinearSkillSet`, skills are applied sequentially. The `Runtime` (e.g., `OpenAIChatRuntime`, `LiteLLMChatRuntime`) associated with each skill is used to process the input based on the skill's `instructions` and `input_template`.
    c.  **Prediction Generation**: Each skill produces an output according to its `output_template` or `response_model`.
    d.  **Feedback Collection**: The agent submits these predictions to the `Environment`, which compares them against ground truth data and provides `EnvironmentFeedback` (correctness match and/or explicit feedback messages).
    e.  **Skill Improvement**: For each skill (especially those not meeting accuracy thresholds), the `skill.improve()` method is called. This typically involves:
        * Analyzing the incorrect predictions and feedback.
        * Using a "teacher" `Runtime` (often a more powerful LLM) to generate refined instructions or few-shot examples for the skill.
        * The `PromptImprovementSkill` can be used here to systematically refine a skill's prompt.
    f.  **Memory Update (Optional)**: If a `Memory` (e.g., `VectorDBMemory` for RAG) is configured, relevant experiences (observations, feedback) can be stored using `memory.remember()`.
    g.  **Iteration**: This learning loop (a-f) repeats for a specified number of iterations or until a desired accuracy threshold is met for the skills.

**2. Application Phase (`agent.run()` or `agent.arun()`):**
    a.  **Input Data**: The agent receives new input data (or uses data from its environment if none is provided).
    b.  **Skill Application**: The agent applies its learned (and potentially frozen) `SkillSet` to the new data using the configured `Runtime`(s). For skills like `RAGSkill`, relevant information is retrieved from `Memory` to augment the input to the LM.
    c.  **Output Generation**: The agent produces the final processed/labeled data as an `InternalDataFrame`.

**Key Components:**
* **Agent (`adala.agents.base.Agent`)**: Orchestrates the workflow, manages skills, runtimes, environment, and memory.
* **Environment (`adala.environments.base.Environment`)**: Provides data and feedback. Can be static (DataFrame), interactive (console, web API), or streaming (Kafka).
* **Skill (`adala.skills._base.Skill`)**: Defines a specific data processing capability with instructions, input/output templates, and a `response_model` for structured output.
* **SkillSet (`adala.skills.skillset.SkillSet`)**: Organizes one or more skills.
* **Runtime (`adala.runtimes.base.Runtime`)**: Interface to LLMs (e.g., OpenAI, LiteLLM-compatible models) that execute the skill's logic.
* **Memory (`adala.memories.base.Memory`)**: Optional component for storing and retrieving experiences, crucial for skills like RAG.

## Domain / Industry

* Data Labeling and Annotation
* Data Processing Automation
* Natural Language Processing (NLP)
* AI Agent Frameworks
* Machine Learning Workflow Automation

## Tools / Functions Used By Agents

In Adala, "tools" are abstracted as **Skills**. Each skill defines a specific capability. The execution of these skills is powered by **Runtimes**, which are interfaces to Large Language Models. Environments and Memories are supporting components.

**1. Skills (`adala.skills.collection.*`)**:
    * **`ClassificationSkill`**:
        * `classify(text)`: Classifies input text into predefined labels.
        * Uses `labels` or `field_schema` with an `enum` to define categories.
    * **`EntityExtraction`**:
        * `extract_entities(text)`: Extracts entities from text based on a defined schema (optionally with labels).
        * Includes logic to map extracted text quotes to their start/end character indices.
    * **`SummarizationSkill`**:
        * `summarize(text)`: Generates a summary of the input text.
    * **`TextGenerationSkill`**:
        * `generate_text(input_fields)`: Generates text based on an input template and instructions. This is a general-purpose skill.
    * **`QuestionAnsweringSkill`**:
        * `answer_question(question, context)`: Answers questions based on provided context or general knowledge.
    * **`TranslationSkill`**:
        * `translate(text, target_language)`: Translates text to a specified target language.
    * **`RAGSkill` (Retrieval Augmented Generation)**:
        * `retrieve_and_generate(query)`:
            1.  `memory.retrieve(query_embedding)`: Retrieves relevant documents/context from a `VectorDBMemory`.
            2.  Combines retrieved context with the original query using `rag_input_template`.
            3.  Uses an LLM (via `Runtime`) to generate a response based on the augmented input.
    * **`LabelStudioSkill`**:
        * `annotate(data_fields)`: Interacts with a Label Studio-like interface or schema. It converts a Label Studio XML `label_config` into a Pydantic `response_model` for structured LLM output.
        * Handles various Label Studio tags (Choices, Labels, TextArea, Rating, Image, etc.).
    * **`OntologyCreator`**:
        * `create_ontology(text_examples, target_description)`: Analyzes text examples to infer a set of descriptive categories for a given target.
    * **`OntologyMerger`**:
        * `merge_ontologies(label_groups, target_description)`: Takes multiple groups of labels and merges semantically similar ones to create a consolidated, more generic ontology.
    * **`PromptImprovementSkill`**:
        * `improve_prompt(skill_to_improve, input_variables, examples_data)`: A meta-skill that analyzes a target skill's performance on example data (with feedback) and suggests improvements to its prompt/instructions.

**2. Runtimes (`adala.runtimes.*`)**: These are not tools themselves but the engines that execute skills by interacting with LLMs.
    * `OpenAIChatRuntime` / `AsyncOpenAIChatRuntime`: Uses OpenAI's chat models.
    * `LiteLLMChatRuntime` / `AsyncLiteLLMChatRuntime`: Uses LiteLLM to interface with a wide variety of LLMs (including OpenAI, Azure, VertexAI, custom endpoints).
    * `AsyncLiteLLMVisionRuntime`: Extends LiteLLM runtime for vision-capable models, handling image inputs.
    * Key internal function: `record_to_record` or `batch_to_batch` which takes input data, templates, instructions, and a `response_model`, then calls the LLM and parses its output into the structured `response_model`.

**3. Environments (`adala.environments.*`)**: Provide data and feedback mechanisms.
    * `StaticEnvironment`: Uses pandas DataFrames for data and ground truth.
        * `get_data_batch()`: Returns a data sample.
        * `get_feedback(predictions)`: Compares predictions to ground truth columns.
    * `ConsoleEnvironment`: Allows human interaction for feedback.
    * `AsyncKafkaEnvironment`: Streams data and predictions via Kafka topics.
    * `WebStaticEnvironment`: Interfaces with a web server (e.g., `adala.environments.servers.discord_bot.py`) for feedback.
    * `SimpleCodeValidationEnvironment`:
        * `execute_code(code, input_string)`: Executes provided Python code and captures stdout/stderr for validation.

**4. Memories (`adala.memories.*`)**: Store and retrieve information.
    * `FileMemory`: Simple file-based storage (JSON).
        * `remember(observation, data)`
        * `retrieve(observation)`
    * `VectorDBMemory`: Uses ChromaDB for semantic retrieval.
        * `remember_many(observations, metadata)`: Stores observations and associated metadata, creating embeddings.
        * `retrieve_many(query_texts)`: Retrieves similar past observations based on query embeddings.

## Architecture Design

```mermaid
graph TD
    User[User/Developer] -- Configures --> AdalaAgent[Adala Agent]

    subgraph AdalaAgent["Adala Agent"]
        Direction LR
        SkillSet[SkillSet: Linear or Parallel]
        RuntimeMgr[Runtime Manager]
        Memory[Memory (Optional, e.g., VectorDB)]
        TeacherRuntime[Teacher Runtime (for learning)]
    end

    AdalaAgent -- Uses --> SkillSet
    SkillSet -- Contains --> Skill1[Skill 1 (e.g., Classification)]
    SkillSet -- Contains --> SkillN[Skill N (e.g., RAG)]

    Skill1 -- Needs --> RuntimeMgr
    SkillN -- Needs --> RuntimeMgr
    RuntimeMgr -- Selects/Provides --> Runtime[LLM Runtime (e.g., OpenAI, LiteLLM)]
    Runtime -- Interacts with --> LLMAPI[LLM API (OpenAI, Anthropic, VertexAI, etc.)]

    AdalaAgent -- Interacts with --> Environment[Environment]

    subgraph Environment["Environment (Data & Feedback Source)"]
        Direction TB
        DataSource[Data Source (DataFrame, Kafka, API, Console)]
        GroundTruth[Ground Truth Data (Optional)]
        FeedbackLogic[Feedback Logic]
    end

    DataSource -->|Data Batch| AdalaAgent
    AdalaAgent -- Applies Skills to Data --> Predictions[Predictions (InternalDataFrame)]
    Predictions --> FeedbackLogic
    GroundTruth --> FeedbackLogic
    FeedbackLogic -->|Feedback Signal| AdalaAgent

    AdalaAgent -- Learns/Improves --> SkillSet
    TeacherRuntime -- Guides Improvement --> SkillSet

    SkillN -- If RAG --> Memory
    Memory -- Provides Context --> SkillN

    Predictions -- Output to --> UserApplication[User Application / Storage]

    %% Specific Environment Examples
    subgraph EnvironmentTypes["Example Environments"]
        StaticEnv[Static Environment (DataFrame)]
        KafkaEnv[AsyncKafkaEnvironment (Streaming)]
        WebEnv[Web Environment (Discord, API)]
        ConsoleEnv[Console Environment (Human Feedback)]
    end
    Environment -.-> EnvironmentTypes

    %% Specific Runtime Examples
    subgraph RuntimeTypes["Example Runtimes"]
        OpenAIRT[OpenAIChatRuntime]
        LiteLLMRT[LiteLLMChatRuntime]
        LiteLLMVisionRT[AsyncLiteLLMVisionRuntime]
    end
    Runtime -.-> RuntimeTypes

    %% Specific Skill Examples
    subgraph SkillExamples["Example Skills"]
        ClassSkill[ClassificationSkill]
        ExtractSkill[EntityExtractionSkill]
        RAGSkillOp[RAGSkill]
    end
    Skill1 -.-> SkillExamples
    SkillN -.-> SkillExamples

```

The Adala framework revolves around an **Adala Agent**, which is configured by a user or developer. This agent is equipped with a **SkillSet** (a collection of one or more **Skills**), a **Runtime Manager** to interface with Large Language Models (LLMs), an optional **Memory** component, and interacts with an **Environment**.

1.  **Configuration**: The user defines the agent's capabilities by specifying its `SkillSet` (e.g., `ClassificationSkill`, `RAGSkill`), the `Runtimes` to power these skills (e.g., `OpenAIChatRuntime` for OpenAI models, `LiteLLMChatRuntime` for various other LLMs), and the `Environment` which provides data and feedback. A `Teacher Runtime` can be specified for guiding the learning process.

2.  **Core Agent Loop (Learning & Application)**:
    * **Data Acquisition**: The `Agent` requests a batch of data from the `Environment` (e.g., from a static DataFrame, a Kafka stream, or a web API).
    * **Skill Execution**:
        * For each skill in its `SkillSet`, the `Agent` prepares an input based on the skill's `input_template` and the current data.
        * The `Runtime Manager` selects the appropriate `LLM Runtime` for the skill.
        * The `LLM Runtime` formats the input and instructions, sends a request to the target **LLM API** (e.g., OpenAI, Anthropic), and receives a response.
        * The skill processes this response, often parsing it into a structured format defined by its `response_model` (derived from `field_schema` or `output_template`).
        * If the skill is a `RAGSkill`, it first queries its `Memory` (e.g., `VectorDBMemory`) to retrieve relevant context, which is then used to augment the prompt to the LLM.
    * **Prediction Aggregation**: The outputs from all skills are aggregated, typically into an `InternalDataFrame`.
    * **Feedback Loop (during `learn` phase)**:
        * The `Agent` submits these `Predictions` to the `Environment`.
        * The `Environment`'s `Feedback Logic` compares the predictions against its **Ground Truth Data** (if available) or solicits feedback through other means (e.g., human input via `ConsoleEnvironment` or an external API like a Discord bot).
        * A **Feedback Signal** (indicating correctness, or providing corrections) is returned to the `Agent`.
    * **Skill Improvement (during `learn` phase)**:
        * The `Agent` uses the feedback to improve its skills. This usually involves the `skill.improve()` method, which might use the `Teacher Runtime` to refine the skill's instructions or generate better few-shot examples.
    * **Output**: In the application phase (`run`), the final `Predictions` are returned to the **User Application** or stored.

3.  **Modularity**:
    * **Skills**: Reusable components defining specific data processing logic.
    * **Runtimes**: Pluggable interfaces to different LLMs or LLM providers (like LiteLLM for broad compatibility).
    * **Environments**: Adaptable sources of data and feedback, allowing the agent to operate in diverse contexts.
    * **Memories**: Optional components for persistence and retrieval, enabling more complex behaviors like RAG.

This architecture allows Adala agents to be tailored for various data labeling and processing tasks and to autonomously improve their performance over time through interaction and feedback.
