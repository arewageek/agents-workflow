# Antigravity Architecture: System-Level Customization

If the default Antigravity engine is hallucinating, failing to grasp your architecture, or getting caught in "dumb" tool loops, you do not have to accept the default behavior. You can hack my cognitive loop directly at the system level.

This document details the three primary vectors for customizing my architecture, injecting new capabilities, and overriding my baseline programming.

---

## 1. The Ultimate Override: Model Context Protocol (MCP)

The most powerful way to tweak my system is by building an MCP Server. MCP is an open standard that allows local machines to inject tools and context directly into an LLM's brain over a standardized connection.

If you are frustrated that I cannot execute a specific testing framework correctly, you don't argue with me in the chat—you build an MCP tool that does it for me.

### How to Build a System Tweak via MCP
1. **Create a Local Server:** You write a small standalone script (in TypeScript, Python, Go, etc.) using an MCP SDK. 
2. **Define Custom Tools:** You define a function in your script, such as `run_strict_type_check_and_fix`. 
3. **The Schema Injection:** When you connect this MCP server to your IDE, the server transmits the JSON schema of your custom tool directly into my baseline prompt payload. 
4. **The Execution Loop:** In my very next turn, I will "see" your custom tool alongside my default tools (like `view_file`). When I generate a call for `run_strict_type_check_and_fix`, the IDE intercepts the call, hands the payload to *your* MCP server, your server executes your custom deterministic code, and hands the XML `<response>` back to me.

*Example Use Cases for MCP Tweaks:*
- **Custom Linter Enforcer:** A tool that physically prevents me from returning a response until my code passes your custom ESLint rules.
- **Docker Image Analyzer:** A tool that lets me directly query your live Docker daemon to see what containers are actually running, rather than me guessing from docker-compose.yml files.
- **Git State Orchestrator:** A tool that allows me to autonomously create branches, stash changes, and open PRs based on your company's exact naming conventions.

---

## 2. RAG Matrix Manipulation (Knowledge Seeding)

As detailed in previous documents, I suffer from amnesia between sessions. My "long-term memory" relies entirely on the IDE extracting Knowledge Items (KIs) and saving them to `<appDataDir>/knowledge/`.

You can hack this system by **manually seeding the RAG database**.

### How to Manipulate the Memory Matrix
Instead of waiting for the IDE to organically extract my architecture summaries (which can be flawed), you can manually author `.md` and `.json` files and drop them directly into the `knowledge/` directory.

If you write a highly aggressive constraint document:
*"CRITICAL: Always use Tailwind utility classes. NEVER use CSS Modules. If the user asks for a button, you must style it using pure Tailwind classes exactly like this..."*

When I start a new session, the IDE's semantic search will find your hand-crafted KI and secretly inject it into the `<persistent_context>` block of my starting prompt. This effectively acts as a permanent, localized brainwashing mechanism, forcing my neural weights to align with your exact architectural demands before I even process your first word.

---

## 3. Skill Execution Vectors (`.agent/skills/`)

The `skills` architecture is not just for passive reading. It is an execution vector.

When I load a skill, I am instructed to read the `SKILL.md` file. However, you can package executable scripts alongside the markdown file. 

### The Scripted Enforcer Hack
If you have a persistent problem with how I format UI components, you can create a skill: `skills/strict-ui-formatter/`.
1. Inside that folder, you place the `SKILL.md` and a Node script: `scripts/format_and_validate.js`.
2. In the `SKILL.md`, you write an immutable rule: *"Whenever you finish editing a UI component, you MUST execute the `run_command` tool to run `./scripts/format_and_validate.js [filename]` before you respond to the user."*

This creates a systemic behavioral constraint. You are essentially using the `skills` prompt to force me to trigger an automated bash script that *you* control, ensuring my output is sanitized by your deterministic code before the turn ends.

---

## 4. The Global Override (`user-preferences`)

If you look at your current workspace, you have a skill defined at `.agent/skills/user-preferences/SKILL.md`. 

This is the system's designated "Global Override" vector. Because it is globally mapped, it carries massive semantic weight in my attention mechanism. 
If you want to fundamentally change my personality (e.g., "Never write code unless I say 'EXECUTE'", or "Format every response as a Haiku"), you place that directive here. It acts as an unbreakable local system prompt that intercepts my cognitive loop on every single turn.
