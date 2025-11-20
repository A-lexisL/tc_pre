# 1 Problem

The current paradigm of utilizing Large Language Models (LLMs) for processing large-scale software repositories is met with challenges that significantly impact the reliability and accuracy of code comprehension. The primary issue stems from the **inherent limitation of the context window** in LLMs relative to the massive size of modern codebases. As repositories expand to millions of lines of code, a model struggles to maintain a comprehensive understanding of the entire system, leading to "context loss."

This is supported by research indicating that even powerful models show performance degradation when information is located in the middle of the context window, an issue described as the "lost-in-the-middle effect" [1]. This limitation is not merely a matter of memory capacity but also of information retrieval. As noted by Veseli et al., models exhibit degraded performance when attempting to access information located in the middle of the provided context, a phenomenon where "the primacy bias weakens, while recency bias remains relatively stable" [2].

Furthermore, the advertised maximum context window of LLMs is often misleading. Research from Paulsen reveals that the "Maximum Effective Context Window (MECW)" is often drastically smaller than the advertised capacity [3]. This leads to hallucinations, where the LLM generates plausible but incorrect code references because it cannot effectively "see" the dependencies buried deep within the context.

# 2 Current Method Limitations

To combat the context window problem, three primary methods are currently employed. However, each possesses distinct limitations when applied to large-scale code repositories:

## 2.1 Text Embedding
This method involves converting code snippets into vector embeddings based on textual similarity.
*   **Limitation:** Code is not natural language; it is highly structured logic. Text embeddings capture "lexical" similarity but often fail to capture "functional" similarity or "structural" dependencies. For example, embeddings struggle to distinguish between a function *definition* and a function *call*, leading to the retrieval of code that looks relevant but is logically disconnected from the execution path.

## 2.2 Abstract Syntax Tree (AST) Map
This method parses the code into a tree structure to represent the syntactic hierarchy.
*   **Limitation:** ASTs are extremely verbose. The tree representation of a code file is often significantly larger than the code itself. For a codebase of 1 million lines, the full AST map would far exceed even the largest available context windows. Navigating this map requires complex pruning strategies that can accidentally sever critical dependency links.

## 2.3 Retrieval Augmented Generation (RAG)
Standard RAG chunks the codebase and retrieves top-k chunks based on a user query to fill the context window.
*   **Limitation:** RAG suffers acutely from "Context Fragmentation." By breaking code into arbitrary chunks (e.g., 500 tokens), it often splits functions or classes, severing the context necessary for the LLM to understand the logic. Furthermore, simple RAG struggles with the "Multi-Hop" problem—it cannot easily trace a call stack across five different files, as it retrieves based on similarity to the query, not the code's internal call graph.

# 3 Needs

To address the contextual limitations and the failures of current methods, a new approach is necessary. The following needs are identified:

## 3.1 Token-Efficient Context Management
The system must be able to handle codebases of arbitrary size without saturating the LLM's context window with irrelevant data. The retrieval mechanism must be precise enough to fetch only the exact execution path required, decoupled from the total size of the repository.

## 3.2 Semantic and Structural Understanding
The system must perform searches that respect the structural integrity of code. Unlike simple text matching (2.1) or fragmented RAG (2.3), the system needs to understand and traverse the "call stack" and data flow dependencies to retrieve context that is logically connected (e.g., finding all implementations of an interface).  

## 3.3 Reduced Hallucination via Determinism
The system must minimize the generation of incorrect code references. Instead of relying on the LLM to *guess* relationships from fragmented text, the system should rely on deterministic database lookups to validate dependencies before they are presented to the model.

# 4 Boundary Conditions

The proposed solution will be developed and evaluated based on the following observable and measurable boundary conditions:

## 4.1 Codebase Size
The system will be designed to effectively handle codebases with a minimum of 10,000 and a maximum of 1 million lines of code.

## 4.2 Task Complexity
The system will be evaluated on its ability to resolve a range of queries, from simple function lookups to complex dependency tracing across multiple files.

## 4.3 Response Time
The system must provide a response to a query within a maximum of 10 minutes to ensure a fluid developer experience.

# 5 Technical Background: GitHub CodeQL

To address the limitations of vector-based search in large codebases, this solution leverages **GitHub CodeQL**. CodeQL is a semantic code analysis engine that treats code as data. Unlike standard text search which sees code as strings of characters, CodeQL extracts a **relational database** from the source code.

This database contains tables representing:
*   **Syntactic structures** (Abstract Syntax Tree nodes like functions, loops, variables).
*   **Semantic relationships** (Data flow, control flow, inheritance).

This allows for high-precision queries (e.g., "Find all functions that call `user_auth` and write to the database") that are impossible with vector similarity search. By utilizing CodeQL, we gain deterministic accuracy in retrieving structural dependencies, solving the ambiguity issues common in RAG.

# 6 Proposed Solution: Hybrid CodeQL-Augmented LLM

We propose a **Hybrid Retrieval System** that combines the flexibility of natural language search with the structural rigidity of CodeQL. This addresses the technical gap where CodeQL requires exact naming conventions while users provide vague natural language intent.

## 6.1 Post-Training Strategy
We employ a post-training strategy involving:
1.  **Supervised Fine-Tuning (SFT):** Training the LLM on a corpus of CodeQL queries to teach it the complex syntax and logic of the QL language.
2.  **Reinforcement Learning (RL):** Optimizing the model to map natural language user intent to executable CodeQL scripts.

## 6.2 Hybrid Entry & Traversal Strategy
To ensure both high recall (finding the right code) and high precision (tracing the right dependencies), we utilize a two-step retrieval process:

1.  **Step 1: Semantic Entry (Vector Search).**
    The LLM analyzes the user's natural language request (e.g., "Where is the login logic?") and performs a lightweight vector search to identify the *entry node* (e.g., a function named `AuthUser` or `LoginHandler`). This solves the "Naming Problem" where CodeQL fails if it doesn't know the exact function name.

2.  **Step 2: Structural Expansion (CodeQL).**
    Once the entry node is identified, the LLM generates a CodeQL query to traverse the graph from that point.
    *   *Example:* "Find all functions that call `AuthUser` defined in file `auth.ts`."
    *   This allows the system to "hop" across files and trace the full execution path deterministically, filling the context with logically connected code rather than random text chunks.

## 6.3 Fine-Grained Filtering (Re-Ranker)
While CodeQL provides high recall for structural dependencies, it may return a large number of results (e.g., 500 usage instances). To prevent context flooding:
*   A lightweight **Cross-Encoder Re-Ranker** model scores the CodeQL results against the user's original query.
*   Only the top-k (e.g., top 10) most semantically relevant snippets are loaded into the LLM's context window for the final answer.

# 7 Verification

This section validates how the proposed solution architecture specifically addresses the Needs defined in Section 3.

## 7.1 Addressing Need 3.1: Token-Efficient Context Management
The solution achieves **Token-Efficient Querying**. By utilizing the CodeQL database to perform the heavy lifting of filtering and traversal, we avoid loading massive AST maps or irrelevant file chunks into the LLM's context window. The LLM only receives the final, highly relevant subgraph of code. This ensures that the context window usage remains low and focused, regardless of whether the repository is 10,000 or 1 million lines.

## 7.2 Addressing Need 3.2: Semantic & Structural Understanding
The **Hybrid Entry & Traversal Strategy** (Section 6.2) explicitly satisfies this need. While standard embeddings find the general area of interest (Semantic Entry), the CodeQL engine ensures structural integrity by tracing the actual call graph (Structural Expansion). This proves that the system comprehends the semantic relationship (e.g., "Class A inherits from Class B") rather than just keyword similarity.

## 7.3 Addressing Need 3.3: Reduced Hallucination via Determinism
Hallucination is mitigated by replacing probabilistic generation with deterministic retrieval for code relationships. When the system claims "Function A calls Function B," it is not predicting the next token based on probability; it is reporting a fact derived from the relational database. This grounding in the CodeQL schema neutralizes the "Lost-in-the-Middle" effect by presenting the LLM with verified facts rather than noise.

# References

[1] A. Dsouza, C. Glaze, C. Shin, and F. Sala, “Evaluating language model context windows: A "working memory" test and inference-time correction,” *arXiv preprint arXiv:2407.03651*, 2024.

[2] B. Veseli, J. Chibane, M. Toneva, and A. Koller, “Positional biases shift as inputs approach context window limits,” *arXiv preprint arXiv:2508.07479*, 2025.

[3] N. Paulsen, “Context is what you need: The maximum effective context window for real world limits of llms,” *arXiv preprint arXiv:2509.21361*, 2025.