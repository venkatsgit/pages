---
layout: default
title: "The vLLM Story"
---

# The vLLM Story: Why do we need it?

Have you ever wondered why ChatGPT feels fast, while running an AI model on your own A100 GPU can be very slow? 

Running Large Language Models (LLMs) is **slow and costs a lot of money**. Before vLLM, a single high-end GPU could only handle one or two requests per second. To serve many users, you needed too many GPUs and too much money.

vLLM was built to fix this. It is the "engine" that makes AI models fast, easy to use, and cheaper to run.

### The Journey of a Prompt: From Text to Thought
When you send a message to an AI, your words go on a long journey inside the GPU:

1. **The Arrival:** Your words are cut into small pieces called "tokens."
2. **The Processing:** The GPU does a lot of math to understand your meaning.
3. **The Generation:** The model predicts the very next word, then "loops" back to do it again for the next word.

To understand how vLLM makes things fast (using ideas like PagedAttention and Speculative Decoding), we first need to see how the AI works. Let's look inside the **Transformer**.

---

# The "Cocktail Party" Idea of Attention

To understand Transformers, imagine a party with two guests: **Alex** and **Venkat**.

| Component | Alex (Input 1) | Venkat (Input 2) |
| :--- | :--- | :--- |
| **Query (Q)** | "How do I build an agent?" | "What is the basic idea of LLMs?" |
| **Key (K)** | "AI Engineer" | "Product Engineer (MLOps/vLLM)" |
| **Value (V)** | "Deep knowledge of AI theory" | "Practical knowledge of MLOps & vLLM" |

### How they talk:

1. **Attention Score:** Alex's Query matches Venkat's Key. They realize they have something to talk about.
2. **Softmax (Scaling):** They decide to focus 80% of their talk on each other and 20% on the background noise.
3. **Output:** Alex leaves the talk having "learned" Venkat's knowledge. His internal knowledge is now updated with practical vLLM info.

---

# The Journey of a Word: From Dictionary to Context

How does the model know if "Bank" means a place for money or the edge of a river? It happens in three steps:

### 1. Static Embedding (The "Identity")

Every word starts as a fixed set of numbers.

* **The "Bank" Problem:** At this step, the word "Bank" looks the same in every sentence. It has no meaning from the words around it yet.

### 2. Positional Embedding (The "Location")

The AI looks at all words at the same time, so it needs to know the order of the words.

* **The Fix:** We add "position info" to the word. This tells the model that "Bank" is the 2nd word in "The bank of the river," not the 10th.

### 3. Contextual Embedding (The "Meaning from Context")

This happens during the Attention step.

* **The Change:** In "The **bank** of the **river**," the word **Bank** (Query) looks at the word **river** (Key).
* **The Result:** Because "river" is close and important, "Bank" takes the "water and nature" meaning from the context.
* **Final State:** The word "Bank" is now updated to mean "land next to water."

---

# The Structure of a Talk (Heads & Dimensions)

Words have "depth" and many different meanings.

### 1. Dimensions (*d*<sub>model</sub>)

Every word is a list of numbers. The number of dimensions shows how "detailed" the model's understanding is:

* **BERT-Base:** 768 dimensions.
* **Llama-3 (8B):** 4,096 dimensions.
* **GPT-3:** 12,288 dimensions.

### 2. Multi-Head Attention: The Group Discussion

Instead of one talk, the model has many **Heads** working at the same time.

* **Example:** Alex and Venkat are in a room with 32 different microphones.
    * **Head 1** listens for grammar.
    * **Head 2** listens for technical words.
    * **Head 3** listens for feelings.
* **Final Output:** All heads combine their results to make one very detailed set of numbers.

---

# The Math of "Matching": **Q × K^T × V**

How do we go from numbers to "meaning"? It is a three-step process.

### Step 1: The Matching Check (Q × K^T)

The **Query (Q)** of one word is compared with the **Key (K)** of all other words.

* **Result:** A score that shows how well they match.

### Step 2: The Decision (Softmax)

We turn those scores into percentages that add up to 100%.

* **Result:** "I will spend 80% of my focus on Venkat and 20% on others."

### Step 3: The Learning (Attention @ V)

We multiply those percentages by the **Value (V)** of the guests.

* **Result:** Alex is now "updated" with Venkat's specific knowledge.

---

# A Simple 3D Example: "Bank of the River"

Let's see how the math works. Imagine the word **Bank** is trying to decide if it means "Money" or "Nature." 

**Note:** We use the word **Money** here as a "wrong" example. Even though it is not in our prompt, we want to see if the AI can tell the difference between a "Money Bank" and a "River Bank."

### The Setup (3D Dimensions):

* Dim 1: **Finance/Money**
* Dim 2: **Water/Nature**
* Dim 3: **Physical Land**

### The Word Vectors:

1. **Bank (Query):** Q = [0.1, 0.9, 0.5] — *(It is looking for nature and land)*
2. **River (Key):** K = [0.0, 1.0, 0.4] — *(It says "I am 100% water and 40% land")*
3. **Money (Key):** K = [1.0, 0.0, 0.0] — *(It says "I am 100% money")*

### The Calculation (Q × K^T):

* **Bank vs. River:** (0.1×0) + (0.9×1.0) + (0.5×0.4) = **1.1**
* **Bank vs. Money:** (0.1×1.0) + (0.9×0) + (0.5×0) = **0.1**

### The Result:

Because **1.1** is much higher than **0.1**, the model gives "River" a high weight (nearly 90%). "Bank" now officially means a river bank!

---

# The Full Assembly Line: Predicting "To Chill Out"

When you give the prompt: *"The bank of the river is the perfect place"*, the model follows this flow:

### 1. The GPT "Decoder-Only" Structure

*Diagram: Transformer block flow — Input → Attention → Add & Norm → Feed Forward → Add & Norm → Linear → Softmax → Output.*

<pre class="mermaid">
graph TD
    subgraph "Transformer Block (Repeated many times)"
        A[Input Words + Position] --> B["Attention (The Meeting)"]
        B --> C[Add &amp; Norm]
        C --> D["Feed Forward (Thinking)"]
        D --> E[Add &amp; Norm]
    end
    
    E --> F[Linear Layer]
    F --> G["Softmax (Probabilities)"]
    G --> H["Output Word: 'to'"]
</pre>

### 2. Step-by-Step Breakdown:

#### **Architecture Stage 1: Input & Position**
The words "The", "bank", "of", "the", "river", "is", "the", "perfect", "place" enter. Alex and Venkat are at the party and know where they are sitting.

#### **Architecture Stage 2: Masked Attention**

In GPT, the attention is "Masked." This means "Bank" can look at "The", but it **cannot** look at future words (like "place").

* **The Meeting:** "Bank" looks at "River" and confirms: *"I am in a nature context."*

#### **Architecture Stage 3: Thinking**
The word **Bank** enters its "private office." It uses its knowledge to conclude: *"Since I am a river bank, I should be about peace or resting."*

#### **Architecture Stage 4: Prediction**

The model focuses on the very last word: **"place"**.

1. **Linear Layer:** Maps the word to all possible words (e.g., 50,000 words).
2. **Softmax:** Calculates the chance for each word.
3. **"to"** (85%)
4. **Result:** It outputs **"to"**.

#### **Architecture Stage 5: The Loop**
The model now takes *"The bank of the river is the perfect place **to**"* and feeds it back to the start. It repeats this until it says **"chill"**, then **"out"**.

---

# The Memory Hack: Why we need the KV Cache

In **Architecture Stage 5 (The Loop)**, we saw that the model repeats the whole process for every new word. This is very slow.

### The "Dory" Problem (Forgetting Everything)

Imagine if the model acted like Dory from the movie *Finding Nemo*:

1. **Step 1:** Model reads "The... place," calculates everything, and says **"to"**.
2. **Step 2:** Model **forgets everything**. It re-reads the whole history—"The", "bank", "of", "the", "river", "is", "the", "perfect", "place"—PLUS the new word "to," and calculates everything again from zero to say **"chill"**.
3. **Step 3:** Model **forgets again**. It re-reads everything and says **"out"**.

As the story gets longer, the model gets much, much slower.

### The KV Cache Solution: "Sticky Notes"

The **KV Cache** gives the model a memory.

1. **First Pass (Prefill):** The model calculates the **Keys (K)** and **Values (V)** for every word in your message once.
2. **The Cache:** It saves those K and V notes in the GPU memory (the KV Cache).
3. **The Shortcut (Decoding):** When it predicts the next word after **"to"**:
    * It **only** does the math for the new word **"to"**.
    * It "grabs" the saved math for the earlier words from its "sticky notes" (the cache).

**The Benefit:** We turn a huge, repeating task into a simple "look-up" task. This makes real-time chatting possible.

---

# PagedAttention: The "Tetris" of GPU Memory

If the KV Cache is our "Sticky Note" system, we have a problem: **GPU Memory is small and expensive.** 

### The Old Way: The "Reserved Bookshelf" Problem

In older systems, the memory for a conversation had to be one single, long piece.

* Imagine if, for **Alex** and **Venkat** to talk, you had to buy a **huge 10-shelf bookshelf** the moment they said their first word.
* **The Waste:** You don't know if they will talk for 1 minute or 1 hour, so you reserve the *most* space. If they only talk for a minute, 9 shelves stay empty, but no one else can use them.
* **Wasted Space:** Eventually, your room is full of giant, mostly-empty bookshelves. A new guest arrives, but there is no space on the floor for another giant bookshelf, even if the room has "empty air."

### The vLLM Way: PagedAttention (The "Floating Shelves")

PagedAttention breaks the memory into small "pages" (like how your computer's RAM works).

* **The Technical Bit:** vLLM splits the memory into **Logical Blocks**. Each block holds 16 tokens. These are mapped to **Physical Blocks** in the GPU via a **Block Table** (our "Table of Contents").
* **The Analogy:** Instead of a giant fixed bookshelf, we give Alex and Venkat **small individual floating shelves**. Each shelf holds 16 "sticky notes" (tokens).
* **The Benefit:** We only add a new shelf when the current one is 100% full. These shelves can be put anywhere there is space in the GPU—they do not have to be next to each other.
* **The Result:** This stops all memory waste. This is the "secret sauce" that gave vLLM its **24x speed improvement**.

### The Extra Win: Automatic Prefix Caching

Because our memory is now flexible, we can do **Prefix Caching**.

* Suppose 10 people all start their talk with the same 500-word story.
* Instead of every person having their own copy of "Sticky Notes," vLLM lets all 10 people **share the same floating shelves** for that story.
* This saves a lot of memory and makes the first answer instant for everyone.

### The Trade-off

* **For the System:** It is like a Librarian tracking a story where pages are on 20 different shelves. The bookkeeping is more complex.
* **For the User:** Almost **no downside**. It is just faster and cheaper.

**Now that memory is efficient, we need to keep the GPU busy. This is where the Scheduler takes over.**

---

# The "Party Host" Logic: Inference Phases and Batching

To keep the GPU working at 100%, we look at the two ways an AI model "talks." 

### 1. Prefill vs. Decoding: The Inference Phases

Every talk with a model has two stages:

* **Inference Phase 1: The Prefill (Compute-Bound)** — This is the "First Impression." When you hit enter, the model processes your entire message at once to create the **KV Cache**. It is a big burst of math for the GPU.
* **Inference Phase 2: Decoding (Memory-Bound)** — This is the "Back-and-Forth." After the intro, the model predicts words one by one. This is lighter on math but requires the GPU to quickly "read" all the saved memory (the Floating Shelves).

### 2. What is a Batch?
A **Batch** is a group of people the GPU talks to at the same time. Since the GPU is a super-calculator, it is faster to calculate the next word for 100 people at once than to do them one by one.

### 3. Static vs. Continuous Batching
* **Static Batching (The Old Way):** Like a waiter who won't serve any new guests until **everyone** at the table has finished their entire 5-course meal. If Alex finishes in 2 seconds, his seat stays empty and wasted until Venkat finishes his long essay.
* **Continuous Batching (The vLLM Way):** Like a busy sushi bar. The **Scheduler** (the Party Host) checks the stools after *every single word*. The moment Alex finishes and says "Goodbye" (the **EOS Token**), the Host fills his seat with the next person in line immediately.

*Diagram: Sushi bar — Queue feeds empty stools; when someone leaves, the next guest fills the seat.*

<pre class="mermaid">
graph LR
    subgraph "The Sushi Bar (GPU)"
        S1[Stool 1: Alex]
        S2[Stool 2: Venkat]
        S3[Stool 3: Empty]
    end
    
    Q[Queue: New Guests] -->|Immediate Entry| S3
    S1 -->|Done| Gone[Leaves]
    Gone -->|Immediate Fill| S1
</pre>

### 4. Who's the Boss? (The Scheduler)

The **Scheduler** is the brain of vLLM. It decides how many people to let in based on:

1. **The Seat Limit:** Usually up to **256** people.
2. **The Word Limit:** It won't process more than **2048** words total at one time to keep things fast.
3. **Memory:** It checks if there is enough space for new sticky notes.

### 5. Chunked Prefill: The "No-Starvation" Rule

What if a new guest has a massive 2,000-word intro? vLLM uses **Chunked Prefill**.

* **What it is:** Normally, a new guest takes over the whole GPU, making others wait. vLLM breaks their intro into small "chunks." It mixes these chunks with the words of people already at the table.
* **The Example:** Instead of ignoring all current tables to seat a group of 20, the waiter seats 2 people, checks his other tables, then seats 2 more.
* **The Benefit:** Current guests do not see a pause. The conversation stays smooth. This reduces delays by up to **2x**.
* **The Policy:** This is vLLM's "No-Starvation" rule: **Current guests come first.**

### The Trade-off
* **Overhead:** There is a tiny bit of extra work for the Scheduler to manage these chunks, but it is worth it for the smooth speed.

---

# Speculative Decoding: The "Junior Intern" Strategy

Even with these tricks, AI can be slow. A big model (like Llama-3 70B) is like a smart Senior Engineer—great but slow to think.

### The Problem: Loading billions of numbers

Every time the model predicts a word, it has to load billions of "weights" from memory. The GPU often waits for these numbers to load.

### The Solution: The Junior Intern (Draft Model)

We hire a **Junior Intern** (a tiny, fast model).

* **1. The Guess (Proposal):** The Intern quickly "drafts" the next 5 words. The Intern is fast but can make mistakes.
* **2. The Check (Scoring):** The Senior Engineer looks at those 5 words all at once.
* **3. The Decision (Verification):**
    * If the Intern is right, the Senior nods. We get those words very fast.
    * If the Intern is wrong, the Senior fixes it.

### The Result: Doubling the Speed
* **Speed:** We can cut the time per word from **30ms down to 16ms**.
* **Quality:** The big model always checks the work, so the quality stays the same.

### How to use it:

```bash
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Meta-Llama-3-70B \
    --speculative-model meta-llama/Meta-Llama-3-8B \
    --num-speculative-tokens 5 \
    --enable-prefix-caching
```

---

# Quantization: Making the Model Smaller

Large models take up a lot of space. **Quantization** is the process of making them smaller.

### The Problem: Heavy Backpacks

A big model is like a 140GB backpack. It is heavy and slow to move.

### The Solution: Smaller Numbers (FP8, GPTQ, AWQ)

Quantization rounds the numbers used by the model (like rounding 3.14 to 3).

* **The Example:** Like converting a **4K video** to **1080p**.
* **The Benefit:** The file is half the size and loads much faster.
* **The Result:** On high-end GPUs, this can **cut delays by 50%**.

### The Trade-off

If you make it too small (like 4-bit), the model might get a bit "dumber," like watching a movie in low quality.

---

# Guided Decoding: The "Strict Form" Rule

Sometimes you need the answer in a specific format (like JSON) so your code can read it.

* **The Problem: The Talkative Chef** — LLMs are wordy. If you ask for a price, they might tell you a long story first. Extra text breaks your software.
* **The Solution: The Fill-in-the-Blanks Form** — Imagine giving the model a **paper form with boxes**. The model is physically blocked from writing outside the boxes.
* **The Benefit:** Your code won't crash because the model forgot a bracket or added a polite sentence at the end.

### The Trade-off

Checking the rules before every word can make the generation **slightly slower**.

---

# Summary
We have learned how vLLM uses **PagedAttention**, **Continuous Batching**, and **Speculative Decoding** to make AI models fast and cheap to run. vLLM helps models talk at the speed of thought! 

Host your models with vLLM and enjoy fast AI chatting! 🚀
