---
title: Idea Exhaustor
description: Feedforward Foundry
published: true
date: 2025-10-03T19:08:04.429Z
tags: 
editor: markdown
dateCreated: 2025-10-03T19:08:04.429Z
---

# The Idea Exhaustor: A Scientific Approach to Exploring Solution Spaces

## Overview

The Idea Exhaustor is a system that estimates when an AI model + prompt combination has exhausted its ability to generate meaningfully distinct ideas. By borrowing concepts from **ecological biodiversity assessment**, it can predict the total "idea space" available to a given configuration and track progress toward exhausting that space. Instead of having to guess when you are done using an LLM for research, we can estimate how many more ideas the system might be able to create.

### Theory
#### The Ecological Inspiration: Species Richness Estimation

Imagine you're a field biologist studying butterflies in a forest. You catch 100 butterflies and identify 30 different species. The fundamental question: **How many total species exist in this forest that you haven't caught yet?**

This is the **species richness estimation problem**. You've observed a sample, but the true population remains hidden. Ecologists developed clever statistical methods to estimate total richness from incomplete sampling.

#### The Chao1 Estimator

Chao (1984) developed an elegant estimator based on a simple insight: **the frequency of rare species tells us about unobserved species**.

The estimator works like this:

- **u** = number of unique species you've observed
- **f₁** = "singletons" – species you've seen exactly once
- **f₂** = "doubletons" – species you've seen exactly twice

**The Chao1 formula:**

```
T̂ = u + (f₁²) / (2 × f₂)
```

When f₂ = 0 (no doubletons), a bias-corrected version is used:

```
T̂ = u + f₁(f₁ - 1) / 2
```

**The intuition:** If you're seeing many singletons, you're likely missing many unobserved species. The ratio between singletons and doubletons captures your sampling intensity – high f₁ relative to f₂ suggests shallow coverage.

#### From Butterflies to Ideas
An LLM + prompt is an **idea generation function** f(prompt) → ideas. Just as a forest has finite species diversity, this function has a finite capacity to generate distinct ideas before it begins repeating itself in subtle ways.

While recent research has confirmed that AI systems can generate high-quality ideas, they struggle with diversity (Dell'Acqua et al., 2023; Meincke et al., 2025). Ideas tend to cluster too tightly, limiting variance in the ideation process which reduces the chances for a breakout idea. As Kornish and Ulrich (2011) demonstrated in their empirical analysis of large idea samples, the lessons from ecology can be applied to ideeation as well and Meincke, Mollick, and Terwiesch (2024) applied the logic to LLM-based exploration for the first time.

This motivates the Idea Exhaustor's approach: **track idea clusters over time to estimate when you've explored the accessible idea space**.

## How the Estimator Works

### Step 1: Clustering Ideas by Similarity

Each generated idea is:
1. **Embedded** into a high-dimensional vector using OpenAI's `text-embedding-3-small`
2. **Compared** to existing cluster representatives using cosine similarity
3. **Assigned** to an existing cluster if similarity ≥ 0.7, or forms a new cluster

This clustering creates our "species" – groups of conceptually similar ideas. Of course, other choices for embedding models and thresholds are possible.

### Step 2: Tracking the Frequency Spectrum

The system maintains a count for each cluster:
- Cluster A: 5 ideas (seen 5 times)
- Cluster B: 1 idea (singleton)
- Cluster C: 2 ideas (doubleton)
- Cluster D: 1 idea (singleton)
- ...

From these counts, we compute:
- **N** = total ideas generated
- **u** = number of unique clusters observed
- **f₁** = clusters with count = 1
- **f₂** = clusters with count = 2

### Step 3: Applying Chao1

With these statistics, we estimate the total cluster richness:

```python
def chao1_estimator(u: int, f1: int, f2: int) -> float:
    if f2 == 0:
        return u + f1 * (f1 - 1) / 2.0
    return u + (f1 * f1) / (2.0 * f2)
```

This gives us **T̂** – the estimated total number of distinct idea clusters in the complete idea space.

### Step 4: Computing Exhaustion

**Exhaustion percentage** = (u / T̂) × 100

This tells us: "We've observed **u** clusters out of an estimated **T̂** total clusters, so we're **X%** done."

When exhaustion exceeds the user's threshold (default 95%), the system stops generating – it has effectively "exhausted" the idea space for this configuration.

## Implementation Guide

### Architecture Overview

The system has three main components:

1. **Idea Generation Loop**: Calls LLM in batches to generate ideas
2. **Clustering Engine**: Groups similar ideas using embeddings
3. **Exhaustion Calculator**: Applies Chao1 to estimate progress

### Core Implementation

#### 1. Clustering with Representatives

```python
clusters: list[dict] = []  # Each: {key, rep_emb, count}
cluster_sim_threshold = 0.7

def assign_cluster(emb: List[float]) -> str:
    """Assign idea embedding to nearest cluster or create new one."""
    best_sim = -1.0
    best_cluster = None
    
    # Find most similar existing cluster representative
    for cl in clusters:
        sim = cosine(emb, cl["rep_emb"])
        if sim >= cluster_sim_threshold and sim > best_sim:
            best_sim = sim
            best_cluster = cl
    
    if best_cluster is not None:
        best_cluster["count"] += 1
        return best_cluster["key"]
    
    # No similar cluster found – spawn new one
    key = f"cluster_{cluster_id_counter}"
    cluster_id_counter += 1
    clusters.append({"key": key, "rep_emb": emb, "count": 1})
    return key
```

We compare against the **first idea's embedding** (representative), not a centroid. This is faster and works well for tight clusters but other implementations are possible. See Cox et al. (2021) for a broader discussion.

#### 2. Exhaustion Calculation

```python
def compute_exhaustion():
    # Build frequency spectrum
    instance_counts = [cl["count"] for cl in clusters]
    freq_counter = Counter(instance_counts)
    
    u = len(instance_counts)  # observed clusters
    f1 = freq_counter.get(1, 0)  # singletons
    f2 = freq_counter.get(2, 0)  # doubletons
    
    # Apply Chao1
    chao1T = chao1_estimator(u, f1, f2)
    
    # Compute exhaustion
    exhaustion_pct = (u / chao1T) * 100 if chao1T else 0.0
    
    return exhaustion_pct, {
        "N": len(all_ideas),
        "u": u,
        "f1": f1,
        "f2": f2,
        "chao1T": chao1T,
    }
```

#### 3. Retain History

The system keeps the current ideation history to encourage novelty:

```python
def build_prompt(query: str, batch_size: int, prior: list[str]) -> str:
    """Build prompt that requests ideas distinct from prior ones."""
    if not prior:
        # First batch: simple request
        return (
            f"Generate {batch_size} ideas for: {query}\n\n"
            "Return as newline-separated list. "
            "Each idea: 40-80 words, format 'number. title: description'"
        )
    
    # Later batches: show what we have, ask for distinct ideas
    prior_block = "\n".join(f"- {idea}" for idea in prior)
    return (
        f"We already generated these ideas for '{query}':\n"
        f"{prior_block}\n\n"
        f"Generate {batch_size} **additional, distinct** ideas.\n"
        "Return as newline-separated list..."
    )
```

This helps the LLM avoid repetition by explicitly showing what's already been generated.

#### 4. Cosine Similarity

```python
def cosine(a, b):
    """Compute cosine similarity between two embedding vectors."""
    a = np.asarray(a)
    b = np.asarray(b)
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))
```

#### 5. Main Generation Loop

```python
async def generate_with_exhaustion(query: str, batch_size: int, 
                                   threshold: float, max_batches: int):
    """Generate ideas until exhaustion threshold is reached."""
    all_ideas = []
    all_embeddings = []
    
    for batch_no in range(1, max_batches + 1):
        # Build context-aware prompt
        prompt = build_prompt(query, batch_size, all_ideas)
        
        # Call LLM
        response = await llm.generate(prompt)
        new_ideas = parse_ideas(response)
        
        # Process each new idea
        for idea in new_ideas:
            emb = await get_embedding(idea)
            assign_cluster(emb)
            
            all_ideas.append(idea)
            all_embeddings.append(emb)
            
            # Check exhaustion
            exhaustion_pct, details = compute_exhaustion()
            
            # Stop if threshold exceeded (minimum 10 ideas required)
            if exhaustion_pct / 100 > threshold and details["N"] >= 10:
                return all_ideas, details
    
    # Reached max batches
    return all_ideas, compute_exhaustion()[1]
```

### Configuration Parameters

- **batch_size** (default: 25): Ideas generated per LLM call
- **similarity_threshold** (default: 0.95 = 95%): Exhaustion % to stop at
- **max_batches** (default: 20): Safety limit on iterations
- **cluster_sim_threshold** (fixed: 0.7): Minimum similarity to join cluster

**Tuning guidance (depends on embedding model):**
- **Lower similarity threshold** (70%): Faster, fewer ideas, higher diversity per idea
- **Higher similarity threshold** (98%): Slower, more ideas, diminishing returns
- **Sweet spot**: 90-95% balances coverage and efficiency

## Interpreting the Statistics

When the system reports:

```
N=150, u=42, f1=18, f2=8, T̂=62.2, Exhaustion=67.5%
```

This means:
- **N=150**: Generated 150 total ideas
- **u=42**: Identified 42 distinct idea clusters
- **f1=18**: 18 clusters have only one idea (rare themes)
- **f2=8**: 8 clusters have exactly two ideas
- **T̂=62.2**: Estimated ~62 total clusters exist
- **Exhaustion=67.5%**: Explored about 2/3 of the idea space

The high f1 count (18 singletons) suggests there's still room to explore – many themes appeared only once. As generation continues, f1 should decrease (rare themes get revisited) and exhaustion should increase.

## Example Session

```
Query: "New features for a task management app"

Batch 1: N=25, u=22, f1=20, f2=2, T̂=122, Exhaustion=18%
→ High diversity, low coverage. Keep going.

Batch 5: N=125, u=68, f1=32, f2=15, T̂=102, Exhaustion=67%
→ Still finding new clusters, but slowing down.

Batch 8: N=200, u=87, f1=24, f2=18, T̂=95, Exhaustion=92%
→ Few singletons remain. Approaching completion.

Batch 9: N=225, u=91, f1=18, f2=20, T̂=94, Exhaustion=97%
→ Threshold exceeded. Stopping.
```

The system generated 225 ideas across 91 conceptual clusters, exhausting an estimated 97% of the idea space for this LLM + prompt configuration.

## Conclusion

By treating idea generation as an ecological sampling problem, the Idea Exhaustor provides a principled way to measure creative exhaustion and capture the diminishing returns of continued ideation. This allows users to optimize their brainstorming process – knowing when they've genuinely exhausted a prompt's potential versus just scratching the surface.

## References
```markdown
Chao, A. (1984). Nonparametric Estimation of the Number of Classes in a Population. *Scandinavian Journal of Statistics*, 11(4), 265-270. https://www.jstor.org/stable/4615964

Cox, S. R., Wang, Y., Abdul, A., von der Weth, C., & Lim, B. Y. (2021). Directed Diversity: Leveraging Language Embedding Distances for Collective Creativity in Crowd Ideation. In *Proceedings of the 2021 CHI Conference on Human Factors in Computing Systems* (Article 393, pp. 1–35). Association for Computing Machinery. https://doi.org/10.1145/3411764.3445708

Dell'Acqua, F., McFowland, E., Mollick, E. R., Lifshitz-Assaf, H., Kellogg, K., Rajendran, S., Krayer, L., Candelon, F., & Lakhani, K. R. (2023). Navigating the Jagged Technological Frontier: Field Experimental Evidence of the Effects of AI on Knowledge Worker Productivity and Quality. Harvard Business School Working Paper No. 24-013.

Kornish, L. J., & Ulrich, K. T. (2011). Opportunity Spaces in Innovation: Empirical Analysis of Large Samples of Ideas. *Management Science*, 57(1), 107-128. https://www.jstor.org/stable/41060704

Meincke, L., Mollick, E., & Terwiesch, C. (2024). Prompting Diverse Ideas: Increasing AI Idea Variance. The Wharton School Working Paper. https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4708466

Meincke, L., Nave, G., & Terwiesch, C. (2025). ChatGPT decreases idea diversity in brainstorming. *Nature Human Behaviour*, 9, 1107–1109. https://doi.org/10.1038/s41562-025-01878-0
```