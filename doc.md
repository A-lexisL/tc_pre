# 1 Problem

The current paradigm of utilizing Large Language Models (LLMs) for processing large-scale software repositories is met with challenges that significantly impact the reliability and accuracy of code comprehension. The primary issue stems from the **inherent limitation of the context window** in LLMs. As codebases expand, a model struggles to maintain a comprehensive understanding of the entire repository, leading to a phenomenon known as "context loss."

This is supported by research indicating that even powerful models show performance degradation when information is located in the middle of the context window, an issue described as the "lost-in-the-middle effect" [1]. This limitation is not merely a matter of memory capacity but also of information retrieval. As noted by Veseli et al., models exhibit degraded performance when attempting to access information located in the middle of the provided context, a phenomenon where "the primacy bias weakens, while recency bias remains relatively stable" [2].

Furthermore, the advertised maximum context window of LLMs is often misleading. Research from Paulsen reveals that the "Maximum Effective Context Window (MECW)" is often drastically smaller than the advertised capacity, with some models failing to retrieve information with as little as 100 tokens in context [3]. This leads to hallucinations, where the LLM generates plausible but incorrect code references, and a general failure to grasp the intricate dependencies within a large codebase.

# 2 Current Method Limitations

To combat the context window problem, three primary methods are currently employed. However, each possesses distinct limitations when applied to large-scale code repositories:

## 2.1 Text Embedding
This method involves converting code snippets into vector embeddings based on textual similarity.
*   **Limitation:** Code is not natural language; it is highly structured logic. Text embeddings often fail to capture "functional" similarity (e.g., two functions doing the same thing but named differently) or "structural" dependencies (e.g., inheritance or variable scope). Relying solely on text similarity leads to retrieving code that *looks* relevant but is logically disconnected from the execution path.

## 2.2 Abstract Syntax Tree (AST) Map
This method parses the code into a tree structure to represent the syntactic hierarchy.
*   **Limitation:** ASTs are verbose. The tree representation of a code file is often significantly larger than the code itself. For a codebase of 1 million lines, the full AST map would far exceed even the largest available context windows. Navigating this map requires complex pruning strategies that can accidentally sever critical dependency links, re-introducing context loss.

## 2.3 Retrieval Augmented Generation (RAG)
Standard RAG chunks the codebase and retrieves top-k chunks based on a user query to fill the context window.
*   **Limitation:** RAG suffers acutely from "Context Fragmentation." By breaking code into arbitrary chunks (e.g., 500 tokens), it often splits functions or classes, severing the context necessary for the LLM to understand the logic. Furthermore, simple RAG struggles with the "Multi-Hop" problem—it cannot easily trace a call stack across five different files, as it retrieves based on similarity to the query, not the code's internal call graph.

# 3 Needs

To address the contextual limitations and the failures of current methods, a new approach is necessary. The following needs are identified:

## 3.1 Scalable Context Management
The system must be able to handle codebases of arbitrary size without suffering from significant context loss. Crucially, the retrieval mechanism must not require expensive pre-indexing or vectorization processes that scale poorly with codebase size.

## 3.2 Semantic and Structural Understanding
The system should be able to perform semantic searches that respect the structural integrity of code. Unlike simple text matching (Need 2.1) or fragmented RAG (Need 2.3), the system needs to understand and traverse the "call stack" and data flow dependencies to retrieve context that is logically connected.

## 3.3 Reduced Hallucination
The system must minimize the generation of incorrect or irrelevant code by maintaining a high-quality context window. By ensuring that only the most relevant code files—validated by strict semantic queries—are presented to the model, the likelihood of hallucination should be drastically reduced.

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

This allows for high-precision queries (e.g., "Find all functions that call `user_auth` and write to the database") that are impossible with vector similarity search. By utilizing CodeQL, we gain 100% precision in retrieving structural dependencies, solving the ambiguity issues common in RAG.

# 6 Proposed Solution: CodeQL-Augmented LLM with Verification Loop

We propose a Post-Trained Tool-Augmented system. Instead of simple RAG, the LLM is trained to operate as a **CodeQL Query Engineer** with a built-in verification loop.

## 6.1 Post-Training Strategy
We employ a post-training strategy involving:
1.  **Supervised Fine-Tuning (SFT):** Training the LLM on a corpus of CodeQL queries to teach it the complex syntax and logic of the QL language.
2.  **Reinforcement Learning (RL):** Optimizing the model to map natural language user intent to executable CodeQL scripts.

## 6.2 The "Sample-First" Verification Loop
To ensure the generated CodeQL queries are accurate before they touch the massive codebase, we implement a **Test-Then-Query** workflow:

1.  **Step 1: Sample Generation.** Upon receiving a user request, the LLM first generates a small, synthetic "Golden Sample" code snippet that mimics the structure of the code it *expects* to find.
2.  **Step 2: Query Validation.** The LLM generates the CodeQL query and runs it against this synthetic sample.
    *   *If the query fails or returns nothing:* The LLM self-corrects and iterates until the query successfully retrieves the target info from the sample.
    *   *If the query succeeds:* The validated query is promoted to the next step.
3.  **Step 3: Execution at Scale.** The validated CodeQL query is executed against the actual 1M+ line CodeQL database.

## 6.3 Fine-Grained Filtering (Re-Ranker)
While CodeQL provides high recall for structural dependencies, it may return a large number of results (e.g., 500 usage instances). To prevent context flooding:
*   A lightweight **Cross-Encoder Re-Ranker** model scores the CodeQL results against the user's natural language query.
*   Only the top-k (e.g., top 10) most semantically relevant snippets are loaded into the LLM's context window for the final answer.

# 7 Verification

This section validates how the proposed solution architecture specifically addresses the Needs defined in Section 3.

## 7.1 Addressing Need 3.1: Scalable Context Management
The solution achieves **costless scalability** regarding indexing. Unlike vector databases (Method 2.1) that require expensive GPU compute to embed millions of lines of code, CodeQL utilizes a relational database structure that scales efficiently with standard database optimization techniques. The retrieval cost is decoupled from the context window size, allowing the system to handle 1 million lines of code as easily as 10,000.

## 7.2 Addressing Need 3.2: Semantic & Structural Understanding
The CodeQL engine explicitly satisfies this need by treating code as a graph. The **Verification Loop** (Step 6.2) ensures that the LLM understands the structure before querying. By validating the query on a generated sample, we prove that the system comprehends the semantic relationship (e.g., "Class A inherits from Class B") rather than just keyword similarity, avoiding the token bloat of raw AST maps (Method 2.2).

## 7.3 Addressing Need 3.3: Reduced Hallucination
Hallucination is mitigated through two distinct layers:
1.  **Input Quality:** The "Sample-First" strategy prevents the execution of malformed queries that would yield garbage data.
2.  **Output Filtering:** The **Re-Ranker** (Step 6.3) acts as a final sanity check, ensuring that irrelevant code—even if structurally related—is filtered out. By feeding the LLM only highly curated, structurally validated code, the "Lost-in-the-Middle" effect is neutralized, directly reducing the hallucination rate.

# References

[1] A. Dsouza, C. Glaze, C. Shin, and F. Sala, “Evaluating language model context windows: A "working memory" test and inference-time correction,” *arXiv preprint arXiv:2407.03651*, 2024.

[2] B. Veseli, J. Chibane, M. Toneva, and A. Koller, “Positional biases shift as inputs approach context window limits,” *arXiv preprint arXiv:2508.07479*, 2025.

[3] N. Paulsen, “Context is what you need: The maximum effective context window for real world limits of llms,” *arXiv preprint arXiv:2509.21361*, 2025.
