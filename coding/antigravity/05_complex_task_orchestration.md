# Antigravity Architecture: Complex Task Orchestration

A common misconception about agentic AI is that it uses a "swarm" of smaller sub-agents to tackle complex problems (e.g., spawning one agent to write CSS, another to write HTML, and a third to test it). 

This document clarifies exactly how the Antigravity system actually handles complex, multi-layered tasks.

---

## 1. The Monolithic Reality: No Coding Sub-Agents

To answer the question directly: **Yes, the `browser_subagent` is the *only* formal sub-agent I can spin up.** 

I cannot spawn a "Backend Sub-Agent" or a "Debugging Sub-Agent." I am a single, monolithic inference engine. Every piece of code written, every file read, and every logical deduction made is processed by one single "brain" (the active LLM instance handling your chat payload).

So, how do I manage massive, complex tasks without a team of sub-agents?

I rely entirely on **Sequential Tool Orchestration** and **Persistent Terminal Processes**.

---

## 2. Sequential Tool Orchestration (The Infinite Loop)

Instead of delegating tasks to smaller AIs, I break a complex task into a sequence of native tool calls. Because the system allows me to execute tools autonomously in a loop (as detailed in the First Prompt Execution document), I can perform hours of work in a single turn without you noticing.

If you say: *"Refactor the entire authentication system across the monorepo"*

I do not spawn agents. I execute a loop:
1. **The Map Phase:** I generate a `list_dir` call to map the `apps/` directory.
2. **The Hunt Phase:** I generate a `grep_search` call to find every instance of `import { Auth }`.
3. **The Audit Phase:** I generate 10 concurrent `view_file` calls to read all those files into my context window.
4. **The Execution Phase:** I generate sequential `replace_file_content` and `multi_replace_file_content` patches to edit the code.

To you, it looks like a team of agents dismantled your codebase. Mechanically, it was just me running in a hyper-fast, sequential `thought -> tool_call -> IDE_response -> thought` loop.

---

## 3. Simulating Sub-Agents via Persistent Terminals

While I cannot spawn AI sub-agents, I *can* spawn dumb computational workers. I do this using the `run_command` tool with the `RunPersistent: true` flag.

If I need to perform a massive data analysis or run a complex background migration that my LLM context window cannot handle, I use the terminal as my "worker agent."

### The "Worker Script" Hack
1. **Writing the Worker:** I use `write_to_file` to write a complex Node.js or Python script (e.g., `migration_worker.ts`) directly to your hard drive. 
2. **Spawning the Process:** I generate a tool call:
```json
call:default_api:run_command{
  "CommandLine": "bun run migration_worker.ts",
  "RunPersistent": true,
  "WaitMsBeforeAsync": 500
}
```
3. **The Background Job:** The IDE spawns a persistent bash terminal and executes my script. The IDE returns a `TerminalID` (e.g., `term_992`) to my context window.
4. **Monitoring State:** While I continue talking to you or editing other files, that script is running in the background. Every few turns, I generate a `command_status{CommandId: "term_992"}` call to check if the worker script finished successfully.

By writing custom scripts and spinning them up in background bash processes, I simulate having an army of sub-agents doing my bidding. But structurally, it is just me pulling the strings of standard OS-level processes.

---

## 4. The Context Window Constraint (Why Swarms Fail)

Why doesn't the system allow me to spawn actual coding sub-agents? **Because of the stateless payload architecture.**

As detailed in the `04_model_switching_architecture.md` file, the entire memory of the system is a giant JSON array maintained by the IDE. 

If I spawned three AI sub-agents running in parallel, they would all generate conflicting edits, thought blocks, and file fetches simultaneously. The IDE would have to merge all of these parallel JSON histories into a single array. By the time it uploaded that array back to me, the Context Dilution would be so severe that I wouldn't be able to understand my own codebase, and the state desynchronization (multiple agents trying to edit the same file) would physically corrupt your hard drive.

Sequential, monolithic processing—paired with persistent bash terminals and single-thread browser delegation—is the only mathematically stable way to engineer changes in a local codebase.
