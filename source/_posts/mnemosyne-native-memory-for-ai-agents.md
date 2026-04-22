---
title: 'Mnemosyne: Long-Horizon Memory for Hermes Agent'
date: 2025-11-20 05:00:00
tags:
  - ai
  - machine-learning
  - sqlite
  - open-source
thumbnail: https://imgur.com/t7hl1Fw.jpg
---

<h1 id="Hermes-Remembers-Yesterday-But-What-About-Last-Year"><a href="#Hermes-Remembers-Yesterday-But-What-About-Last-Year" class="headerlink" title="Hermes Remembers Yesterday. But What About Last Year?"></a>Hermes Remembers Yesterday. But What About Last Year?</h1>

I have been using <a href="https://github.com/AxDSan/hermes" target="_blank">Hermes Agent</a> for a while now, and it is genuinely one of the most capable AI agent frameworks out there. It handles multi-step tasks, tool calling, autonomous research, and session management better than most alternatives I have tried.

But there was one thing that kept nagging me: memory.

Hermes ships with a default memory system. It works fine for short sessions. It captures the current context, recent conversation turns, and maybe a summary of what happened an hour ago. But that is about it. If you have ever tried to reference a project decision from three months ago, or reminded it about a preference you established in January, you know the pain. The default memory is capped, short-horizon, and optimized for the immediate session rather than the long arc of your relationship with the agent.

That is not a knock on Hermes. It is a design trade-off. But it was not the trade-off I wanted.

<h1 id="So-I-Built-Mnemosyne"><a href="#So-I-Built-Mnemosyne" class="headerlink" title="So I Built Mnemosyne"></a>So I Built Mnemosyne</h1>

Mnemosyne is a native, zero-cloud memory plugin built specifically for Hermes Agent. It replaces the default memory backend with a SQLite-backed system that stores conversations, preferences, and knowledge locally, using native vector search and full-text search. No external databases. No API keys. No network calls. Just a local SQLite file that lives next to your agent.

The goal was simple: give Hermes a memory that spans months, not minutes. A memory that can recall a technical decision from last quarter, a preference you mentioned once in passing, or the context of a project you shelved and came back to six months later.

The architecture is generic enough that you could adapt it to other agent frameworks if you want, but Hermes is where it lives and where it is tested. The plugin registers itself through Hermes' MemoryProvider interface, so switching from the default backend to Mnemosyne is a single command: `hermes memory setup`.

<h1 id="The-BEAM-Architecture"><a href="#The-BEAM-Architecture" class="headerlink" title="The BEAM Architecture"></a>The BEAM Architecture</h1>

Memory is not a monolith. Treating it like one leads to either bloated context windows or over-aggressive pruning. Mnemosyne organizes memory into three tiers, which I call BEAM:

**B — Hot Working Memory.** This is what Hermes is actively using right now. Recent conversation turns, current task context, recently recalled facts. It is fast, ephemeral by design, and gets pruned aggressively to keep the working set small. Think of it as RAM for an agent.

**E — Episodic Memory.** When working memories age out, they do not get deleted. They get summarized and promoted to episodic memory. This tier holds the long arc of the agent's experience: past conversations, learned facts, established preferences, project history. It is the agent's autobiography. The `mnemosyne_sleep()` function triggers this consolidation, compressing old working memories into dense episodic summaries.

**A — Active Scratchpad.** A temporary workspace for the current task. If Hermes is researching a topic, drafting a document, or debugging code, the scratchpad holds intermediate state without polluting the permanent memory tiers. It is the equivalent of a notepad you tear off and throw away when done.

**M — Meta.** Not a storage tier per se, but the orchestration layer that decides what goes where, what gets recalled, and what gets forgotten. This includes importance scoring, temporal decay functions, and cross-session identity management.

<h1 id="Hybrid-Search-That-Actually-Works"><a href="#Hybrid-Search-That-Actually-Works" class="headerlink" title="Hybrid Search That Actually Works"></a>Hybrid Search That Actually Works</h1>

The hardest part of long-horizon memory is retrieval. You do not want exact-match search. You want semantic search. But pure vector search has blind spots. If you ask Hermes "the project we worked on last Tuesday," vector similarity alone will not help because "Tuesday" is a temporal anchor, not a semantic concept.

Mnemosyne uses a weighted hybrid scoring system:

- **50% vector similarity** via sqlite-vec. Each memory is embedded into a vector space, and queries are matched by cosine similarity. This catches conceptual relevance.
- **30% full-text search rank** via SQLite FTS5. This catches exact phrases, names, dates, and technical terms that vectors might miss.
- **20% importance score.** Memories the user has explicitly flagged, or memories that have been recalled frequently, get boosted. This mimics how human memory works: things you use often stick around.

The result is retrieval that feels intuitive. Ask Hermes about "that database migration issue" and it finds the relevant conversation even if the exact words "database migration" were never spoken.

<h1 id="Performance-Numbers"><a href="#Performance-Numbers" class="headerlink" title="Performance Numbers"></a>Performance Numbers</h1>

Because everything is local SQLite, the numbers are kind of ridiculous:

| Operation | Latency |
|---|---|
| Store a memory | ~0.3ms |
| Recall with hybrid search | ~0.8ms |
| Consolidation (mnemosyne_sleep) | ~5ms per 100 memories |
| Cold start (load existing DB) | ~10ms |

Compare that to cloud alternatives where a single recall operation can take 50-200ms, and the difference is not incremental. It is transformative. An agent that recalls in under a millisecond feels psychic. An agent that takes 200ms feels broken.

The entire system, including vector search, runs in under 50MB of RAM for typical usage. No Docker containers. No services to keep alive. Just a Python module and a SQLite file.

<h1 id="Privacy-By-Design"><a href="#Privacy-By-Design" class="headerlink" title="Privacy By Design"></a>Privacy By Design</h1>

I am increasingly uncomfortable with the default assumption that AI data should live in the cloud. Your conversations with an agent are personal. They contain project details, opinions, mistakes, and half-formed ideas. Sending all of that to a third-party vector database feels wrong.

Mnemosyne keeps everything on your machine. The SQLite file is just a file. You can back it up, version it, encrypt it, or delete it. There are no Terms of Service to read, no rate limits to worry about, no pricing tiers to calculate. Your agent's memory belongs to you.

<h1 id="How-To-Use-It"><a href="#How-To-Use-It" class="headerlink" title="How To Use It"></a>How To Use It</h1>

Installation is deliberately minimal:

```bash
git clone https://github.com/AxDSan/mnemosyne.git
cd mnemosyne
pip install -e .
python -m mnemosyne.install
```

For Hermes users, activation is one command:

```bash
hermes memory setup
# → Select "mnemosyne" and press Enter
```

For everyone else, the Python API is straightforward and framework-agnostic:

```python
from mnemosyne import Mnemosyne

mem = Mnemosyne("./my_agent_memory.db")
mem.remember("The user prefers Python over JavaScript for backend work")
memories = mem.recall("What does the user prefer for backends?")
```

The `recall()` method returns ranked results with relevance scores, timestamps, and memory types. You can filter by time range, importance threshold, or memory tier.

<h1 id="Lessons-and-Future-Work"><a href="#Lessons-and-Future-Work" class="headerlink" title="Lessons and Future Work"></a>Lessons and Future Work</h1>

Building Mnemosyne taught me that "simple" and "local" are underrated architectural constraints. Cloud defaults are not always correct defaults. For many use cases, a well-designed local system outperforms distributed infrastructure on every metric that matters: latency, cost, privacy, and reliability.

The project is still evolving. Current work includes:

- **Cross-session identity linking.** Recognizing when the same human is interacting through different sessions or devices.
- **Temporal reasoning.** Better handling of time-based queries like "what did we discuss last month?"
- **Memory visualization.** A local web interface for browsing, editing, and auditing an agent's memory.

If you are running Hermes and hitting the memory wall, give Mnemosyne a try. It is not a silver bullet, but it is a solid foundation that respects your data and your time.

The repository is at <a href="https://github.com/AxDSan/mnemosyne" target="_blank">github.com/AxDSan/mnemosyne</a>. Issues, pull requests, and harsh code reviews are all welcome.

Thanks for reading.
