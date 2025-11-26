---
theme: seriph
background: https://cover.sli.dev
title: Semantic Code Search
info: |
  ## Structural Code Analysis Pitch
  Presentation for funding/stakeholders.
transition: slide-left
mdc: true
---

# CodeGist System
## Solving "Context Loss" in Large-Scale Code Repositories

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
  </span>
</div>

---
layout: center
class: text-center
---

# The Reality of AI Coding

> "Why does my Copilot fail when I ask about a bug buried deep in a million-line repo?"

<div class="mt-8 text-gray-500">
The promise of Infinite Context vs. The Reality of Retrieval
</div>

---

# The Problem: Context Loss

Large Language Models (LLMs) suffer from the **"Lost-in-the-Middle"** effect.

<div class="grid grid-cols-2 gap-4 mt-8">

<div>

* **Context Window Limit:** Even "large" windows cannot fit massive repositories.
* **Degradation:** Performance drops significantly when data is in the middle of the prompt \[1\].
* **Illusion:** Advertised context size $\neq$ Effective context size \[3\].

</div>

<div class="flex items-center justify-center h-full">
<img src="./image1.png" class="h-60 object-contain mx-auto" alt="U-Shaped Performance Curve">
</div>

</div>

---
transition: fade-out
layout: section
---

# Why Current Methods Fail

Three dominant approaches, three critical flaws.

---
layout: two-cols
---

# Text Embeddings

Treating code like natural language.

<v-click>

### The Flaw: "Lexical" vs "Functional"
* Embeddings capture text similarity, not logic.
* Cannot distinguish a *function definition* from a *function call*.
* **Result:** Retrieves irrelevant code that "looks" similar.

</v-click>

::right::

<div class="flex items-center justify-center h-full ml-4">
<img src="./mismatch.png" class="h-60 object-contain mx-auto" alt="Logic Mismatch">
</div>

---
layout: two-cols
---

# AST Maps

Feeding the raw syntax tree to the LLM.

<v-click>

### The Flaw: Verbosity
* ASTs are massive (often 10x code size).
* A 1M line codebase = Billions of tokens.
* **Result:** Instantly overflows the context window.

</v-click>

::right::

<div class="flex items-center justify-center h-full ml-4">
<img src="./explosion.png" class="h-60 object-contain mx-auto" alt="Data Explosion">
</div>

---
layout: two-cols
---

# Agentic RAG

<v-click>

### The Flaw
* Inefficiency
  * LLM explores codebase using tools like grep and ls.
  * Sequential keyword searches check files one by one.
* Result: Slow, brittle.

</v-click>

::right::

<div class="flex items-center justify-center h-full ml-4">
<img src="./final.png" class="h-60 object-contain mx-auto" alt="The Severed Link">
</div>


---
layout: center
class: text-center
---

# What We Need

To solve this, we need a paradigm shift.

<div class="grid grid-cols-3 gap-4 mt-10 text-left">
    <div class="p-4 border border-gray-200 rounded">
        <carbon:meter class="text-3xl text-blue-500 mb-2"/>
        <h3 class="font-bold">Token Efficiency</h3>
        <p class="text-sm">Decouple retrieval cost from repo size.</p>
    </div>
    <div class="p-4 border border-gray-200 rounded">
        <carbon:ibm-watson-knowledge-studio class="text-3xl text-green-500 mb-2"/>
        <h3 class="font-bold">Structural Awareness</h3>
        <p class="text-sm">Understand call graphs, not just text matches.</p>
    </div>
    <div class="p-4 border border-gray-200 rounded">
        <carbon:chemistry class="text-3xl text-purple-500 mb-2"/>
        <h3 class="font-bold">Determinism</h3>
        <p class="text-sm">Zero hallucinations on dependencies.</p>
    </div>
</div>



---

# Boundary Conditions & Scalability

We are building for the real world, not just a demo.

<div class="mt-4 mb-4">
  <ul class="space-y-2">
    <li><b>Target Scale:</b> < 1,000,000+ Lines of Code.</li>
    <li><b>Latency:</b> < 10 minutes.</li>
    <li><b>Complexity:</b> Multi-file tracing.</li>
  </ul>
</div>

<div class="flex justify-center mt-2">
  <img src="./size.png" class="h-60 object-contain" />
</div>

---
transition: slide-up
layout: section
---

# The Solution
**Structural Candidate Generation** $\to$ **Semantic Filtering**  $\to$  **Deterministic Context Expansion**

---

# The Technical Core

We leverage two industrial-grade engines to treat code as **Data**, not Text.

<div class="grid grid-cols-2 gap-12 mt-10">

<div class="text-center">
  <h2 class="text-green-600">ast-grep</h2>
  <p class="italic">"Structural Regex"</p>
  <ul class="text-left mt-4 text-sm">
    <li>Pattern matching on AST nodes.</li>
    <li>Insane speed (Milliseconds).</li>
    <li>Filters noise (comments vs code).</li>
  </ul>
</div>

<div class="text-center">
  <h2 class="text-blue-600">GitHub CodeQL</h2>
  <p class="italic">"Relational Database"</p>
  <ul class="text-left mt-4 text-sm">
    <li>Treats code as a database.</li>
    <li>Deep analysis (Taint tracking, Data flow).</li>
    <li>Connects logic across files.</li>
  </ul>
</div>

</div>

---
transition: slide-up
layout: section
---

# The CodeGist System

---

# The Pipeline: Step 1
## Structural Candidate Generation(**First LLM Call**)


*   **Mechanism:** LLM generates *permissive* `ast-grep` patterns.
*   **Input:** "Find retry logic."
*   **Pattern:** `try { $$$ } catch ($$$) { $$$ retry $$$ }`

<!-- VISUALIZATION DESCRIPTION:
An animation of a funnel.
Top of funnel: "1 Million Lines of Code".
Action: A filter labeled "ast-grep" slides across.
Output: "200 Candidate Snippets".
-->
<div class="h-40 mt-8  flex items-center justify-center">
  <img src="./step1.png">
</div>

---

# The Pipeline: Step 2
## Semantic Relevance Filtering

*   **Mechanism:** Cross-Encoder Re-Ranker.
*   **Action:** Compare User Query $\leftrightarrow$ Candidate Snippets.
*   **Output:** Top-k most relevant one.
*   **Significance:** Bridges the gap between *Syntactic Structure* and *User Intent*.

<!-- VISUALIZATION DESCRIPTION:
A magnifying glass focusing on the "200 Candidates".
The glass highlights 5 specific blocks in bright green ("Anchors") and fades the rest to grey.
-->
<div class="h-40 mt-8 bg-gray-100 rounded-lg dark:bg-gray-800 flex items-center justify-center">
  <img src="./step2.png">
</div>

---

# The Pipeline: Step 3
## Deterministic Context Expansion(**Second LLM call**)


*   **Mechanism:** LLM generates a **CodeQL** query based on ast-grep results(anchor).
*   **Result:** A connected subgraph of executable logic. 

<div class="h-40 mt-8 flex items-center justify-center">
  <img src="./step3.png">
</div>

---

# The Gap: Limited Knowledge of CodeQL & ast-grep.

<img src="./limits.png" class="h-100 mx-auto" />

---
layout: center
class: text-center
---

# Enabling the Model

How we bridge the gap between general intelligence and specific tool mastery.
```mermaid
graph LR
    classDef raw fill:#f3f4f6,stroke:#9ca3af,stroke-width:2px,stroke-dasharray: 5 5;
    classDef blackbox fill:#000,stroke:#333,stroke-width:0px,color:#fff;
    classDef expert fill:#dcfce7,stroke:#16a34a,stroke-width:2px;
    classDef input fill:#e0e7ff,stroke:#4f46e5,stroke-width:2px;

    %% Method 1: Post-Training
    Base(Raw Base Model):::raw --> Train[Post-Training]:::blackbox
    Train --> Model(Fine-Tuned<br/>Expert):::expert

    %% Method 2: Run-Time Prompting
    Prompt(Context &<br/>Prompting):::input --> Run[Run-Time<br/>Inference]:::blackbox
    Model --> Run
    Run --> Result(Final Tool<br/>Execution):::expert
    
    linkStyle default stroke-width:2px,fill:none;
```

---

# Tool-Use Supervised Fine-Tuning

<div class="h-[85%] grid place-content-center">

```mermaid {scale: 0.9}
graph LR
    classDef raw fill:#f3f4f6,stroke:#9ca3af,stroke-width:2px,stroke-dasharray: 5 5;
    classDef blackbox fill:#000,stroke:#333,stroke-width:0px,color:#fff;
    classDef expert fill:#dcfce7,stroke:#16a34a,stroke-width:2px;
    classDef data fill:#e0e7ff,stroke:#4f46e5,stroke-width:2px;

    Base(Base LLM):::raw
    
    subgraph Training_Data
        direction TB
        NL["Quest: 'Find Auth Logic'"]:::data
        Target["Answer: ast-grep / CodeQL"]:::data
        NL <-.-> Target
    end

    SFT[Supervised<br/>Fine-Tuning]:::blackbox
    Agent(Tool-Use<br/>Expert):::expert

    Base --> SFT
    Training_Data --> SFT
    SFT --> Agent

    linkStyle default stroke-width:2px,fill:none;
```
</div>

---

# Reinforcement Learning

<div class="h-[85%] grid place-content-center">

```mermaid {scale: 0.9}
graph LR
    classDef raw fill:#f3f4f6,stroke:#9ca3af,stroke-width:2px,stroke-dasharray: 5 5;
    classDef blackbox fill:#000,stroke:#333,stroke-width:0px,color:#fff;
    classDef expert fill:#dcfce7,stroke:#16a34a,stroke-width:2px;
    classDef action fill:#fff7ed,stroke:#ea580c,stroke-width:2px;

    Model(LLM):::raw
    
    subgraph Interaction
        Gen[Gen: ast-grep/CodeQL]:::action
        Env{Execution?}:::blackbox
    end

    Reward((Reward)):::expert
    Penalty((Penalty)):::expert

    Model --> Gen
    Gen --> Env
    
    Env -- "Success" --> Reward
    Env -- "Fail" --> Penalty
    
    Reward -.->|Update Weights| Model
    Penalty -.->|Optimization| Model

    linkStyle default stroke-width:2px,fill:none;
```
</div>

<!--
<div class="grid grid-cols-2 gap-8 mt-8">

<div v-click>
  <div class="flex items-center mb-2">
    <carbon:education class="text-3xl text-blue-500 mr-2" />
    <h3 class="text-xl font-bold">1. Tool-Use SFT</h3>
  </div>
  <div class="opacity-80 text-sm">
    <b>Supervised Fine-Tuning</b>
  </div>
  <p class="mt-4 text-sm leading-relaxed">
    The "Classroom" phase. We teach the model the <i>language</i> of the tools.
  </p>
  <ul class="mt-2 text-sm list-disc pl-4 space-y-2">
    <li><b>Input:</b> "Find user authentication logic"</li>
    <li><b>Target:</b> Valid <code>ast-grep</code> patterns & <code>CodeQL</code> syntax.</li>
    <li><b>Goal:</b> Fluency in tool usage.</li>
  </ul>
</div>

<div v-click>
  <div class="flex items-center mb-2">
    <carbon:trophy class="text-3xl text-yellow-500 mr-2" />
    <h3 class="text-xl font-bold">2. Reinforcement Learning</h3>
  </div>
  <div class="opacity-80 text-sm">
    <b>Reward-Based Optimization</b>
  </div>
  <p class="mt-4 text-sm leading-relaxed">
    The "Practice" phase. We optimize for <i>results</i>.
  </p>
  <ul class="mt-2 text-sm list-disc pl-4 space-y-2">
    <li><b>Reward (+1):</b> If the query returns a verifiable, executable code path.</li>
    <li><b>Penalty (-1):</b> If the query is syntactically correct but returns nothing.</li>
  </ul>
</div>

</div>
-->

---
layout: two-cols
---

# Run-time Context
* Provide unified documentation for `ast-grep` and `CodeQL`.  
* Inject system prompt:  



::right::

<div class="p-4 mt-10 ml-4 border border-gray-500 rounded bg-gray-900 text-white text-xs font-mono" v-click>
System: You are a Structural Analysis Agent.

Tool:
- ast-grep
- CodeQL

Task:
1. Consult docs.
2. Decompose user query.
3. Construct into deterministic query.
</div>

---

# Verification: Why This Wins

<div class="grid grid-cols-3 gap-8 mt-12">

<div v-click>
<h3 class="text-blue-500">Token Efficiency</h3>
<p class="text-xl">We don't scan code. We generate queries.</p>
<p class="text-base text-gray-500 mt-2">LLM only sees the final result, not the whole repo.</p>
</div>

<div v-click>
<h3 class="text-green-500">Structure Aware</h3>
<p class="text-xl">We take advantage of the AST.</p>
<p class="text-base text-gray-500 mt-2">We see calling stack and function chain.</p>
</div>

<div v-click>
<h3 class="text-purple-500">Zero Hallucination</h3>
<p class="text-xl">Deterministic Retrieval.</p>
<p class="text-base text-gray-500 mt-2">Dependencies are proven by ast-grep/CodeQL, not predicted.</p>
</div>

</div>

---
layout: center
class: text-center
---

# Conclusion

Shifting the heavy lifting of search from the LLM to ast-grep/CodeQL

```mermaid
graph LR
A["context waste 
and semantic guessing"]
B["token efficiency
and structural correctness"]
A-->B
```

<div class="mt-12">
  <h2 class="text-xl font-bold">Please Invest in CodeGist system.</h2>
</div>
