# Canvas Study Brain — System Design Mockup

Conceptual deployment for one user on a TrueNAS SCALE home NAS. The browser reaches the app over the tailnet; the database and model are private services on the NAS.

```mermaid
flowchart LR
    subgraph Client[User devices]
        Browser[Browser UI\nSearch · Ask · Quiz · Review]
    end

    subgraph Tailnet[Tailscale tailnet]
        TS[Tailscale access\nTailnet identity / ACL]
    end

    subgraph NAS[TrueNAS SCALE]
        subgraph AppNet[Private app network]
            App[Study Brain app\nWeb UI + API + sync worker]
            Graph[(Neo4j Community\nGraph + full-text + vector search)]
            Model[Ollama\nChat model + embedding model]
        end

        subgraph Datasets[Persistent TrueNAS datasets]
            GraphData[(Neo4j data)]
            ModelData[(Ollama model files)]
            SourceFiles[(Original course files\nextracted text + manifests)]
            Backup[Snapshots / backups]
        end
    end

    subgraph External[External services]
        Canvas[Canvas API\nAuthenticated with read-only token]
        Public[Public assignment links\nNo Canvas credential forwarded]
        OptionalLLM[Optional OpenAI / Anthropic API\nOnly when explicitly selected]
    end

    Browser -->|HTTPS over tailnet| TS
    TS -->|UI/API only| App

    App -->|Cypher queries / graph writes| Graph
    App -->|Generate answers, summaries, quizzes| Model
    App -->|Create query and chunk embeddings| Model

    App -->|Course and assignment sync| Canvas
    App -->|Fetch allowed public content| Public
    Canvas -->|Course, assignment, file, page data| App
    Public -->|Public page/file content| App

    App -->|Optional remote inference| OptionalLLM
    App -->|Originals, extracted text, sync manifest| SourceFiles
    Graph --> GraphData
    Model --> ModelData
    GraphData -.-> Backup
    SourceFiles -.-> Backup

    classDef private fill:#e8f0fe,stroke:#3867a5,color:#172b4d;
    classDef external fill:#fff4e5,stroke:#bd7b00,color:#543600;
    class App,Graph,Model,GraphData,ModelData,SourceFiles private;
    class Canvas,Public,OptionalLLM external;
```

## Main request flows

1. **Sync:** the worker reads courses and assignments from Canvas, follows Canvas resources with the token, and fetches only public external links without forwarding that token. It preserves source URLs, Canvas IDs, content hashes, fetch status, and extraction metadata.
2. **Index:** extracted text is split into cited chunks. Ollama creates embeddings; the app writes source, chunk, concept, and relationship records to Neo4j. Concept edges retain evidence chunk IDs and confidence.
3. **Ask:** the app combines vector matches, full-text matches, and graph traversal to select cited context. Ollama answers from that context and returns source references to the UI.
4. **Quiz/review:** quiz questions point to evidence chunks and concepts. Review attempts and due dates are written back to Neo4j; scheduling stays deterministic in application code.

## Container and network boundaries

- Run Neo4j and Ollama with persistent TrueNAS datasets. The study app gets access to the source-files dataset; the browser does not access storage or databases directly.
- Publish only the study UI/API through Tailscale. Keep Neo4j Bolt/Browser and Ollama ports private to the app network.
- Store Canvas and optional provider credentials as server-side secrets. Do not expose them to browser code or forward the Canvas token to external-link hosts.
- The first deployment can use one Study Brain container for the UI, API, and sync worker. Split the worker out only if ingestion needs independent scheduling or resource limits.
- Optional OpenAI/Anthropic calls are disabled by default; local Ollama is the default for generation and embeddings.
