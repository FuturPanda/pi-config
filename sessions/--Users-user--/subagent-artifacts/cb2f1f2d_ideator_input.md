# Task for ideator

You are a delegated subagent running from a fork of the parent session. Treat the inherited conversation as reference-only context, not a live thread to continue. Do not continue or answer prior messages as if they are waiting for a reply. Your sole job is to execute the task below and return a focused result for that task using your tools.

Task:
We're designing a skills system for a production application (not a dev tool). Skills will evolve frequently — updated by the application itself or its operators, not developers. The user is leaning towards storing skills in a database rather than S3 or the local filesystem.

## Context from previous discussion
- Agno's SkillLoader ABC has a single load() -> List[Skill] method
- Skill dataclass: name, description, instructions, scripts (list of filenames), references (list of filenames), source_path (local path used for file access)
- agent_skills.py reads scripts/references from local disk via source_path — this is a leaky abstraction
- The real differentiator of skills vs knowledge docs is executable scripts
- skills.reload() exists but has a race condition and is process-local

## The core question
For a production application where skills change frequently:
- Is a database the right backend?
- What does the schema look like?
- How do you handle the source_path problem (agent_skills.py expects local disk)?
- How do you handle script execution if scripts live in DB (as text/blobs)?
- What kind of database — relational (Postgres), document (Mongo), vector (pgvector)?
- Should skill content (instructions, scripts, references) be stored inline in the DB or as references?
- How does versioning work in a DB model?
- How does the application update/evolve skills at runtime?
- Is there a hot-reload mechanism that works across multiple instances?

Push hard on the trade-offs. What's the cleanest architecture for this use case?