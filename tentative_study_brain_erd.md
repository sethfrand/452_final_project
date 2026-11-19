# Canvas Study Brain — Conceptual ERD

This is a logical model, not a requirement to use relational storage. It keeps Canvas identifiers and source provenance so the same domain can map to MongoDB documents or graph nodes and edges.

```mermaid
erDiagram
    USER ||--o{ COURSE_ENROLLMENT : has
    COURSE ||--o{ COURSE_ENROLLMENT : includes
    COURSE ||--o{ MODULE : contains
    COURSE ||--o{ ASSIGNMENT : contains
    MODULE o|--o{ ASSIGNMENT : groups
    ASSIGNMENT ||--o{ CONTENT_LINK : references
    SOURCE ||--o{ CONTENT_LINK : target
    SOURCE ||--o{ CHUNK : split_into
    CHUNK ||--o{ CHUNK_CONCEPT : tagged_with
    CONCEPT ||--o{ CHUNK_CONCEPT : grounded_in
    CONCEPT ||--o{ CONCEPT_RELATION : relation_from
    CONCEPT ||--o{ CONCEPT_RELATION : relation_to
    QUIZ ||--o{ QUESTION : contains
    CONCEPT ||--o{ QUESTION : tested_by
    QUESTION ||--o{ QUESTION_EVIDENCE : supported_by
    CHUNK ||--o{ QUESTION_EVIDENCE : evidence_for
    USER ||--o{ REVIEW_STATE : tracks
    QUESTION ||--o{ REVIEW_STATE : scheduled_as
    REVIEW_STATE ||--o{ REVIEW_ATTEMPT : history

    USER {
        string user_id PK
        string display_name
        datetime created_at
    }
    COURSE {
        string course_id PK
        string canvas_course_id UK
        string name
        string term
        string workflow_state
        datetime synced_at
    }
    COURSE_ENROLLMENT {
        string enrollment_id PK
        string user_id FK
        string course_id FK
        string role
        string enrollment_state
    }
    MODULE {
        string module_id PK
        string canvas_module_id UK
        string course_id FK
        string name
        int position
    }
    ASSIGNMENT {
        string assignment_id PK
        string canvas_assignment_id UK
        string course_id FK
        string module_id FK
        string name
        datetime due_at
        string workflow_state
        string description
    }
    CONTENT_LINK {
        string link_id PK
        string assignment_id FK
        string source_id FK
        string link_text
        string link_kind
        string access_status
    }
    SOURCE {
        string source_id PK
        string canvas_file_id UK
        string canvas_page_id UK
        string source_kind
        string title
        string url
        string mime_type
        string content_hash
        datetime fetched_at
    }
    CHUNK {
        string chunk_id PK
        string source_id FK
        int sequence
        string text
        vector embedding
        string embedding_model
    }
    CONCEPT {
        string concept_id PK
        string canonical_name UK
        string description
        string subject_area
    }
    CHUNK_CONCEPT {
        string chunk_id PK, FK
        string concept_id PK, FK
        float confidence
        string evidence_text
    }
    CONCEPT_RELATION {
        string relation_id PK
        string from_concept_id FK
        string to_concept_id FK
        string relation_type
        float confidence
        string evidence_chunk_id FK
    }
    QUIZ {
        string quiz_id PK
        string course_id FK
        string title
        string generation_model
        datetime created_at
    }
    QUESTION {
        string question_id PK
        string quiz_id FK
        string concept_id FK
        string question_type
        string prompt
        string answer
        string explanation
    }
    QUESTION_EVIDENCE {
        string question_id PK, FK
        string chunk_id PK, FK
        string citation_label
    }
    REVIEW_STATE {
        string review_state_id PK
        string user_id FK
        string question_id FK
        datetime due_at
        int interval_days
        int repetition_count
        string last_rating
    }
    REVIEW_ATTEMPT {
        string attempt_id PK
        string review_state_id FK
        datetime answered_at
        string user_response
        string rating
        boolean correct
    }
```

## Storage mappings

- **Graph database:** make `Course`, `Module`, `Assignment`, `Source`, `Chunk`, `Concept`, `Quiz`, `Question`, and `User` nodes. Turn the ERD relationships into typed edges. Store confidence and evidence references on concept edges; keep vector and full-text indexes on `Chunk` text.
- **MongoDB:** use separate collections for courses, assignments, sources, chunks, concepts, quizzes, and review state. Embed small, stable arrays such as module summaries, but reference chunks, sources, and review attempts by ID to avoid growing course documents indefinitely.
- **Common identifiers:** preserve Canvas IDs as unique external keys; store source URL, content hash, sync time, and extraction status to support safe re-syncs and citations.

## Modeling notes

- A source can be a Canvas file, Canvas page, assignment description, or public external page. `CONTENT_LINK` records where it was discovered and whether it could be fetched.
- Concepts and concept-to-concept relationships should retain evidence chunk IDs and confidence. Treat model-extracted relationships as suggestions until grounded in source text.
- Keep spaced-review scheduling deterministic in application code. `REVIEW_STATE` stores the next due time; `REVIEW_ATTEMPT` preserves the answer history.
- The `vector` field is conceptual. Choose the concrete vector representation and index based on the selected database and installed extensions.
