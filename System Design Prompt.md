Below are **four clean prompts** you can reuse with your AI.
They keep the **same structure**, include **Mermaid diagrams**, and **split the workload into four parts** so the context window doesn't overflow.

---

# Prompt 1 (Part 1)

```
Create a section focused on **comparative system design thinking using real-world products that appear similar but require very different architectures**.

The goal is to build the ability to **analyze system behavior and choose architectures based on product requirements, scale, and traffic patterns instead of memorizing system design templates**.

For this response, analyze only the following two groups:

• Video platforms (Netflix vs YouTube vs Disney+ Hotstar vs Twitch)
• Messaging systems (WhatsApp vs Slack vs Discord vs Telegram)

For **each group**, structure the explanation using the following sections:

1. Product Purpose
   Explain what problem each product solves and how users interact with it.

2. User Behavior & Traffic Patterns
   Discuss traffic characteristics such as burst traffic, live events, global distribution, creator-driven traffic, or steady consumption patterns.

3. Read vs Write Patterns
   Explain the dominant operations and fan-out behavior.

4. Latency Expectations
   Explain what level of latency users expect and how that affects architecture.

5. Data Characteristics
   Discuss data size, type (video, messages, metadata, etc.), and storage patterns.

6. Architecture Differences
   Explain why these systems require different architectures.
   Discuss components such as CDNs, message queues, streaming pipelines, indexing systems, caching layers, etc.

7. Scaling Challenges
   Explain major bottlenecks and scaling problems each system must solve.

8. Trade-offs
   Explain important design trade-offs (consistency vs availability, cost vs performance, operational complexity vs simplicity).

After each group, include:

Architectural Insight
Summarize the most important system design lesson from that comparison.

Also include **1–2 Mermaid diagrams per group** to visually explain key architectural flows or differences between the systems.

Example diagram types:

• high-level architecture diagrams
• request flow diagrams
• data pipelines
• fan-out models
• streaming architectures

Keep the diagrams **simple, readable, and focused on architectural ideas rather than implementation detail**.

Focus on **architectural reasoning and system behavior**, not low-level implementation.
```

---

# Prompt 2 (Part 2)

```
Continue the comparative system design analysis using the same structure and level of depth as before.

Analyze the following two groups:

• Ride-sharing platforms (Uber vs Lyft vs Bolt vs DiDi)
• Social networks (Instagram vs TikTok vs X/Twitter vs Reddit)

For each group, include the following sections:

1. Product Purpose
2. User Behavior & Traffic Patterns
3. Read vs Write Patterns
4. Latency Expectations
5. Data Characteristics
6. Architecture Differences
7. Scaling Challenges
8. Trade-offs

After each group include an **Architectural Insight** summarizing the key system design lesson.

Also include **1–2 Mermaid diagrams per group** to visually illustrate the architecture or core data flow (for example: ride matching pipeline, feed generation pipeline, fan-out architecture, etc.).

Keep diagrams simple and focused on understanding system architecture.
```

---

# Prompt 3 (Part 3)

```
Continue the comparative system design analysis using the same structure as before.

Analyze the following two groups:

• Payment systems (PayPal vs Stripe vs Cash App vs Venmo)
• Maps / navigation services (Google Maps vs Waze vs Apple Maps)

For each group include:

1. Product Purpose
2. User Behavior & Traffic Patterns
3. Read vs Write Patterns
4. Latency Expectations
5. Data Characteristics
6. Architecture Differences
7. Scaling Challenges
8. Trade-offs

Include an **Architectural Insight** section after each comparison.

Also include **1–2 Mermaid diagrams per group** to visualize important architecture concepts such as payment processing pipelines, fraud detection flows, map data pipelines, or real-time traffic updates.
```

---

# Prompt 4 (Part 4)

```
Continue the comparative system design analysis using the same structure.

Analyze the following two groups:

• Cloud storage systems (Dropbox vs Google Drive vs OneDrive vs Box)
• Search systems (Google Search vs Bing vs DuckDuckGo vs Amazon product search)

For each group include:

1. Product Purpose
2. User Behavior & Traffic Patterns
3. Read vs Write Patterns
4. Latency Expectations
5. Data Characteristics
6. Architecture Differences
7. Scaling Challenges
8. Trade-offs

After each group include an **Architectural Insight** summarizing the key system design takeaway.

Also include **1–2 Mermaid diagrams per group** to illustrate important architecture flows such as file sync architecture, distributed storage layers, indexing pipelines, or search query execution flows.

Focus on **architectural reasoning and system behavior rather than deep implementation details**.
```

---

✅ **Why this structure is excellent for learning system design**

It forces the AI to explain:

* **Product → Traffic → Architecture**
* **Behavior → Scaling decisions**
* **Trade-offs between systems**

The **Mermaid diagrams** make it much easier to understand things like:

* **fan-out vs fan-in**
* **CDN architectures**
* **feed generation pipelines**
* **distributed storage**
* **real-time systems**

---

If you'd like, I can also give you a **much more advanced version of this exercise** that trains you like **FAANG Staff Engineers do** — where you **predict the architecture first before seeing the answer**. That method accelerates system design skill dramatically.
