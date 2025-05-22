# Creative Writer Agent

## Agents / Agentic Workflows Name
- **Researcher Agent**: Finds information on the web using Bing Search
- **Product Agent**: Retrieves relevant product information using vector search
- **Writer Agent**: Creates articles using research and product information
- **Editor Agent**: Reviews articles and provides feedback
- **Orchestrator**: Coordinates the entire workflow between agents

## Agent / Workflow Description
This is a content creation agentic workflow for an outdoor retailer (Contoso). The system orchestrates multiple specialized agents to create blog articles:

1. The **Orchestrator** coordinates the entire workflow, passing information between agents and handling the feedback loop
2. The **Researcher Agent** uses Bing Search to find relevant information based on user queries
3. The **Product Agent** finds relevant products using vector search and embeddings
4. The **Writer Agent** creates an article using the research and product information
5. The **Editor Agent** reviews the article and decides whether to accept it or provide feedback
6. If feedback is provided, the Orchestrator loops back to the Researcher and Writer agents to improve the article

## Domain / Industry
Retail

## Tools / Functions Used By Agents

### Researcher Agent:
- `find_information`: Web search for general information
- `find_entities`: Search for people, places, or things
- `find_news`: Search for news articles
- Uses Azure OpenAI and Bing Search API

### Product Agent:
- `generate_embeddings`: Creates vector embeddings for queries
- `retrieve_products`: Vector search for products
- `find_products`: Combines query generation, embedding, and retrieval
- Uses Azure OpenAI and Azure AI Search

### Writer Agent:
- `write`: Generates article content using research and product data
- `process`: Processes the writer's response
- Uses Azure OpenAI

### Editor Agent:
- `edit`: Reviews article and decides on feedback
- Uses Azure OpenAI

### Additional Tools:
- Content safety evaluation
- Article quality evaluation (relevance, fluency, coherence, groundedness)
- Image content safety evaluation

## Architecture Design

```mermaid
graph TD
    User[User] -->|Research query, product query,\nwriting assignment| Orchestrator
    
    subgraph Orchestrator[Orchestrator]
        Create[create function]
    end
    
    subgraph Agents[AI Agents]
        Researcher[Researcher Agent]
        Product[Product Agent]
        Writer[Writer Agent]
        Editor[Editor Agent]
    end
    
    subgraph ExternalServices[External Services]
        AzureOpenAI[Azure OpenAI]
        BingSearch[Bing Search]
        AzureSearch[Azure AI Search]
    end
    
    Orchestrator -->|1. Request research| Researcher
    Researcher -->|Web searches| BingSearch
    Researcher -->|Process results| AzureOpenAI
    BingSearch -->|Web results| Researcher
    Researcher -->|Research results| Orchestrator
    
    Orchestrator -->|2. Request product info| Product
    Product -->|Generate embeddings| AzureOpenAI
    Product -->|Vector search| AzureSearch
    AzureSearch -->|Product data| Product
    Product -->|Product results| Orchestrator
    
    Orchestrator -->|3. Generate article| Writer
    Writer -->|Create content| AzureOpenAI
    Writer -->|Article draft| Orchestrator
    
    Orchestrator -->|4. Review article| Editor
    Editor -->|Evaluate content| AzureOpenAI
    Editor -->|Decision & feedback| Orchestrator
    
    Orchestrator -->|5. If feedback accepted,\nloop back to improve| Researcher
    Orchestrator -->|Final article| User
    
    AzureOpenAI -->|LLM capabilities| Agents
```

The architecture shows a multi-agent system with specialized roles working together through the orchestrator to create high-quality articles that incorporate both web research and product information, with an editorial feedback loop to ensure quality.
