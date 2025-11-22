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
The system must perform searches that respect the structural integrity of code. Unlike simple text matching or fragmented RAG, the system needs to understand and traverse the "call stack" and data flow dependencies to retrieve context that is logically connected (e.g., finding all implementations of an interface).  

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


# 5 Technical Background: Structural Analysis Tools

To address the limitations of vector search, this solution leverages **ast-grep** and **GitHub CodeQL**. Both rely on the Abstract Syntax Tree (AST) to treat code as data rather than plain text.

## 5.1 The Abstract Syntax Tree (AST)
The AST represents code as a hierarchical tree of syntactic nodes (e.g., `FunctionDeclaration`, `Identifier`) rather than a linear string of characters. This structure allows tools to distinguish actual logic from unrelated text.
*   **Example:** An AST parser distinguishes the variable `user_id` inside a logic block from the text "user_id" inside a comment, preventing irrelevant matches.

## 5.2 ast-grep: Structural Pattern Matching
**ast-grep** is a high-performance search tool that matches patterns against the AST using "meta-variables" (wildcards) effectively acting as a structural regex. It is optimized for speed, enabling rapid scanning of massive repositories[4].
*   **Example:** The pattern `try { $$$ } catch ($$$) { }` instantaneously locates all empty catch blocks across a project, ignoring formatting or internal content.

## 5.3 GitHub CodeQL: Relational Code Analysis
**CodeQL** treats the codebase as a relational database[5]. During the build, it extracts the AST, Control Flow, and Data Flow into queryable tables. It excels at deep analysis, linking logically connected elements across files.
*   **Example:** A CodeQL query can trace "Taint Tracking" paths to identify if an unsanitized HTTP request parameter flows into a specific SQL execution statement.

# 6 Proposed Solution: Multi-Stage Structural Retrieval

We propose a "Coarse-to-Fine" retrieval pipeline. Instead of relying on a single retrieval method, the system employs a three-stage process: **Structural Candidate Generation**, **Semantic Filtering**, and **Deterministic Context Expansion**.

## 6.1 Step 1: Structural Candidate Generation (ast-grep)
The first phase aims to identify potential entry points in the codebase quickly.
*   **Mechanism:** Instead of simple text search, the LLM generates **permissive ast-grep patterns**. These patterns are designed to be "fuzzy" regarding specific naming but strict regarding structure.
*   **Example:** If a user asks about "handling retry logic," the LLM might generate a pattern like `try { $$$ } catch ($$$) { $$$ retry $$$ }`.
*   **Advantage:** This leverages `ast-grep`'s speed to scan millions of lines of code in milliseconds, returning a high-recall list of candidate snippets that structurally match the intent, filtering out irrelevant text matches (like comments containing the word "retry").

## 6.1 Step 2: Semantic Relevance Filtering (Cross-Encoder)
The `ast-grep` step may yield a large number of results (e.g., 200 matches), many of which may be syntactically similar but semantically irrelevant.
*   **Mechanism:** The raw candidates are passed through a **Cross-Encoder Re-Ranker**. This model takes the user's original natural language query and the code snippet as a pair, outputting a relevance score.
*   **Outcome:** The system filters the results down to the top-$k$ (e.g., top 5) "Anchor Nodes." These represent the most likely locations of the logic the user is interested in. This step bridges the gap between *syntactic structure* and *user intent*.

## 6.1 Step 3: Deterministic Context Expansion (CodeQL)
Once the "Anchor Nodes" are identified, the system must understand their context within the larger system (references, definitions, and data flow).
*   **Mechanism:** The LLM analyzes the top-$k$ Anchor Nodes and generates a targeted **CodeQL query**.
*   **Action:** The CodeQL engine executes a graph traversal starting from these anchor nodes.
    *   *Definition Lookup:* "Where is the class of this object defined?"
    *   *Reference Tracing:* "Who calls this function?"
    *   *Data Flow:* "Where does the variable passed into this function originate?"
*   **Result:** This step deterministically retrieves the connected subgraph of code. Unlike RAG, which might retrieve disconnected chunks, this ensures the final context provided to the user contains the complete, executable logic path.

## 6.2 Tool-Use Alignment Strategy
To bridge the gap between natural language intent and rigid code analysis syntax, we employ a dual-pronged strategy to "equip" the LLM with proficiency in `ast-grep` and CodeQL.

### Post-Training Strategy
To enable this pipeline, the LLM undergoes a specialized post-training regimen:
1.  **Tool-Use SFT (Supervised Fine-Tuning):** The model is fine-tuned on a corpus of structural search patterns. It learns to map natural language queries (e.g., "Find user authentication logic") to:
    *   Permissive **ast-grep** patterns for broad gathering.
    *   Precise **CodeQL** queries for deep traversal.
2.  **Reinforcement Learning:** Optimization rewards are given when the generated queries successfully retrieve executable code paths that answer the user's prompt.

### Run-time Prompting

**1. Unified Documentation:**
We have developed a specialized documentation for both `ast-grep` and CodeQL. During run-time, this file is fed into the LLM's system context, providing it with an immediate, hallucination-free reference for API signatures, standard predicates, and configuration syntax.

**2. System Prompting:**
> 1.  **Decompose:** Break the natural language query into logical requirements.
> 2.  **Consult:** Refer to the provided documentation.  
> 3.  **Construct:** Build atomic sub-rules for each requirement and combine them into a valid query.
> 4.  **Refine:** If the rule is too specific, remove non-essential constraints to improve recall.


# 7 Verification

## 7.1 Addressing Need 3.1: Token-Efficient Context Management
The solution achieves **Token-Efficient Querying** by restricting the LLM's role to "Query Generation" rather than "Content Scanning." The retrieval process requires only two targeted LLM invocations:
1.  **Invocation 1:** Generating the `ast-grep` pattern to identify potential anchors.
2.  **Invocation 2:** Generating the `CodeQL` script to expand context from those anchors.
**Result:** The massive overhead of scanning and traversing the code is offloaded to the external tools (`ast-grep` and CodeQL engine), ensuring the context window is never flooded with raw code regardless of repository size.

## 7.2 Addressing Need 3.2: Semantic and Structural Understanding
The solution satisfies this need by exclusively employing **Structure-Aware Tools** for retrieval. Unlike vector embeddings, both `ast-grep` and CodeQL natively understand the Abstract Syntax Tree (AST). `ast-grep` filters results based on precise syntactic patterns (e.g., class definitions), while CodeQL traverses the semantic graph (e.g., variable data flow).
**Result:** The system fundamentally comprehends the code's architecture, ensuring retrieved snippets are logically connected rather than just textually similar.

## 7.3 Addressing Need 3.3: Reduced Hallucination via Determinism
Hallucination is mitigated by replacing probabilistic generation with **Deterministic Retrieval** for code relationships. When the system asserts a dependency (e.g., "Function A calls Function B"), it is reporting a verifiable fact derived from the CodeQL relational database, not predicting the next token.
**Result:** This grounding prevents the model from inventing non-existent imports or function calls, effectively neutralizing the "Lost-in-the-Middle" effect by presenting the LLM with verified facts.


# References

[1] A. Dsouza, C. Glaze, C. Shin, and F. Sala, “Evaluating language model context windows: A "working memory" test and inference-time correction,” *arXiv preprint arXiv:2407.03651*, 2024.

[2] B. Veseli, J. Chibane, M. Toneva, and A. Koller, “Positional biases shift as inputs approach context window limits,” *arXiv preprint arXiv:2508.07479*, 2025.

[3] N. Paulsen, “Context is what you need: The maximum effective context window for real world limits of llms,” *arXiv preprint arXiv:2509.21361*, 2025.  

[4] ast-grep, “Prompting,” ast-grep documentation, 2025. [Online]. Available: https://ast-grep.github.io/advanced/prompting.html.

[5] GitHub, “CodeQL documentation,” GitHub, 2025. [Online]. Available: https://codeql.github.com/.