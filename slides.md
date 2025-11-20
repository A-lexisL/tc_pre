---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: Hybrid CodeQL-Augmented LLM for Large-Scale Code Comprehension
info: |
  ## Hybrid Retrieval System for Large-Scale Code Comprehension
  Leveraging CodeQL's structural querying with semantic understanding
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# duration of the presentation
duration: 35min
---

# Hybrid CodeQL-Augmented LLM for Large-Scale Code Comprehension

Making LLMs understand complex, large-scale codebases

<div @click="$slidev.nav.next" class="mt-12 py-1" hover:bg="white op-10">
  Press Space for next page <carbon:arrow-right />
</div>

---

# Large Codebase Problem  

> "If code is poetry, then large codebases are epic novels — and our tools often only read the first and last pages."

Can we help LLMs truly understand the full story of complex code?

---

# Problem: Context Window Limitations

Modern codebases (1M+ lines) exceed LLM context windows

<v-clicks>

- **"Lost-in-the-middle" effect**: Performance degrades mid-context
- **Maximum Effective Context Window (MECW)**: Much smaller than advertised
- **Hallucinations**: LLMs create incorrect references when key code is inaccessible

</v-clicks>

<!--add visualization(token/codespace size)-->

---

# Current Methods: Text Embedding

<v-click>

Convert code to vector embeddings based on textual similarity

</v-click>

<v-click>

**LIMITATION**: Can't distinguish function *definition* vs *call*

</v-click>

<v-click>

**RESULT**: Retrieves code that "looks" relevant but is logically disconnected

</v-click>

<v-click>

*Visualization needed: diagram showing wrong code retrieval*

</v-click>

---

# Current Methods: AST Map

<v-click>

Parse code into tree structures to represent syntax hierarchy

</v-click>

<v-click>

**LIMITATION**: AST representation is extremely verbose

</v-click>

<v-click>

**RESULT**: 1M lines → AST exceeds available context windows

</v-click>

<v-click>

*Visualization needed: huge tree diagram vs small context window*

</v-click>

---

# Current Methods: RAG

<v-click>

Chunks codebase, retrieves top-k chunks based on user query

</v-click>

<v-click>

**LIMITATION**: "Context Fragmentation" - splits functions/classes arbitrarily

</v-click>

<v-click>

**RESULT**: Can't trace call stacks across multiple files (Multi-Hop problem)

</v-click>

<v-click>

*Visualization needed: broken code fragments vs complete execution path*

</v-click>

---

# Need for a New Approach

<v-clicks>

- **Token-Efficient Context Management**:  
Handle codebases of arbitrary size without saturating LLM context
- **Semantic and Structural Understanding**:  
Perform searches that respect structural integrity of code  
- **Reduced Hallucination via Determinism**:   
Minimize incorrect code references with deterministic database lookups

</v-clicks>

---

# Boundary Conditions

<v-clicks>

- **Codebase Size**:  
10,000 - 1,000,000 lines of code
- **Task Complexity**:  
Simple lookups ✅ Complex dependency tracing ✅
- **Response Time**:  
Under 10 minutes for developer experience

</v-clicks>

---

# Solution: Combine natural language search with CodeQL's structural precision
<v-click>
Introducing: Hybrid Retrieval System  
= Text Dmbedding Flexibility + CodeQL Structural Rigidity  

</v-click>

---

# What is CodeQL?

<v-click>

Treats code as data in a relational database

</v-click>

<v-click>

- **Developed by GitHub**  
- **Syntactic structures**: Functions, loops, variables as database tables
- **Semantic relationships**: Data flow, control flow, inheritance

</v-click>

<v-click>

Enables high-precision queries impossible with vector search

</v-click>

</v-click>

---

# Post-Training Strategy

<v-click>

## Training the LLM to Understand CodeQL:

</v-click>

<v-clicks>

1. **Supervised Fine-Tuning (SFT)**: Train on CodeQL query corpus to learn QL language
2. **Reinforcement Learning (RL)**: Optimize natural language to executable CodeQL mapping

</v-clicks>

---

# Hybrid Solution Architecture

<v-click>

## Two-Step Retrieval Process:

</v-click>

<v-clicks>

1. **Semantic Entry (Vector Search)**: Find the entry node using natural language
2. **Structural Expansion (CodeQL)**: Trace execution path deterministically

</v-clicks>

---

# Step 1: Semantic Entry

<v-click>

LLM analyzes natural language request: "Where is the login logic?"

</v-click>

<v-click>

Vector search identifies the entry node: `AuthUser` or `LoginHandler`

</v-click>

<!--transform to illustration-->

<v-click>

**Solves**: The "Naming Problem" where CodeQL needs exact names

</v-click>

---

# Step 2: Structural Expansion

<v-click>

LLM generates CodeQL query from entry point

</v-click>

<v-click>

**Example**: "Find all functions that call `AuthUser` defined in file `auth.ts`"

</v-click>

<!--transform to illustration-->

<v-click>

**Result**: Traces full execution path across files deterministically

</v-click>

---

# Fine-Grained Filtering

<v-click>

CodeQL may return many results (500+ usage instances)

</v-click>

<v-click>

**Cross-Encoder Re-Ranker** scores results against original query

</v-click>

<v-click>

Only top-k most relevant snippets enter LLM context

</v-click>

<!--transform to illustration-->
---

# Verification: Token-Efficient Context Management

<v-click>

**Challenge**: How to handle arbitrary codebase size without saturating context?

</v-click>

<v-click>

**Solution**: CodeQL database does heavy lifting, LLM gets only relevant subgraph

</v-click>

<v-click>

**Result**: Context usage remains low regardless of codebase size

</v-click>

---

# Verification: Semantic & Structural Understanding

<v-click>

**Challenge**: How to respect structural integrity of code?

</v-click>

<v-click>

**Solution**: Hybrid Entry & Traversal Strategy

</v-click>

<v-click>

Vector search → structural entry, CodeQL → structural expansion

</v-click>

---

# Verification: Reduced Hallucination via Determinism

<v-click>

**Challenge**: How to minimize incorrect code references?

</v-click>

<v-click>

**Solution**: Replace probabilistic generation with deterministic retrieval

</v-click>

<v-click>

CodeQL facts rather than LLM probabilities

</v-click>

---

# Call to Action

The future of code comprehension lies in combining:
<div class="grid grid-cols-2 gap-10 pt-10">
  <div>
    <div class="text-2xl mb-4">Natural Understanding</div>
    <div> List possibilities</div>
  </div>
  <div>
    <div class="text-2xl mb-4">Structural Precision</div>
    <div> Accurate, deterministic results</div>
  </div>
</div>

**The Hybrid CodeQL-Augmented LLM is the bridge between them.**

<div @click="$slidev.nav.next" class="mt-12 py-1" hover:bg="white op-10">
  Thank you! Questions? <carbon:arrow-right />
</div>