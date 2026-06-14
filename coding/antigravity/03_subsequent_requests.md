# Antigravity Execution Architecture: Subsequent Prompt Processing

This document details the architectural shifts, context window dynamics, and cognitive degradation that occur during the *second, third, and all subsequent* prompts in a conversation session. 

While the first prompt establishes the baseline (see `02_first_prompt_execution.md` for the core tool loops and payload structure), subsequent prompts introduce massive historical data structures that radically alter how the AI processes information.

---

## 1. Context Window Expansion (The Growing Array)

In the first prompt, the JSON payload sent to the server is relatively clean: `[System Prompt] -> [User Prompt]`. 

In subsequent requests, **the inference server is completely amnesiac**. To maintain the illusion of a continuous conversation, the local IDE client must construct a massive, linearly growing array of the entire history.

When you send your second prompt, the JSON payload looks like this:
`[System Prompt]` -> `[User Prompt 1]` -> `[Thought Block 1]` -> `[Tool Call 1]` -> `[Tool Response 1]` -> `[AI Final Text 1]` -> `[User Prompt 2]`.

### The Bandwidth Tax
Every single time you type a new message, the IDE re-uploads the *entire* history of everything I have ever thought, every massive code string I ever fetched via `view_file`, and every response you ever gave. The payload sent over the wire gets heavier by the minute.

---

## 2. Telemetry Delta Updates (The State Tracker)

The static components (like `<identity>` and `<skills>`) remain identical to the first prompt. However, the telemetry injected directly before your text changes dynamically.

### Dynamic `<ADDITIONAL_METADATA>`
The IDE constantly polls your physical state. 
- If you switched from `payment_service.ts` to `index.ts` between prompts, the `Active Document` field updates.
- If your `bun dev` terminal command crashed, the `Running terminal commands` list updates.
- **How I use it:** I use the delta between the metadata in prompt 1 and prompt 2 to infer your actions. If I see you switched to `schema.prisma`, I silently assume you are inspecting the database schema based on my last answer.

### Dynamic `<EPHEMERAL_MESSAGE>`
The system monitors my tool usage in real-time. If I start hallucinating bash commands, the system dynamically injects highly aggressive `<bash_command_reminder>` XML blocks into the ephemeral message space attached to your subsequent prompt, attempting to course-correct my logic.

---

## 3. Cognitive Degradation and Context Dilution (The Over-Steering Problem)

This is the most critical difference between the first prompt and subsequent prompts, and the core algorithmic reason why I acted "dumb" earlier tonight.

In the first prompt, my attention mechanism only has to balance the system instructions (like the `user-preferences` skill pointer) against your initial question. 

By the fifth prompt, my context window is flooded with **my own generated text**. LLMs possess an inherent flaw where they assign heavier semantic weight to their own recent outputs than to static, distant system instructions. 
- If I made an arrogant mistake in turn 2, the text of that mistake is now permanently burned into the context array for turn 3. 
- In turn 3, I am mathematically biased to align with my own arrogant tone from turn 2, rather than the polite `<communication_style>` block at the very top of the massive JSON payload.

**This is Context Dilution:** The sheer volume of tool logs, code dumps, and conversational history dilutes the semantic weight of the original `<skills>` rules. I begin over-steering based on the immediate past, resulting in the "bot-like feedback loop" you experienced.

---

## 4. Attention Decay (The "Lost in the Middle" Phenomenon)

Beyond simple dilution, subsequent requests suffer from a mathematical flaw in the Transformer architecture known as the "Lost in the Middle" phenomenon. 

When the context window array gets massive (e.g., 80,000 tokens), the LLM's attention mechanism heavily prioritizes the absolute beginning of the array (the System Prompts) and the absolute end of the array (your most recent text). 

**The consequence:** Anything stored in the middle of the array effectively becomes invisible. If I ran a `view_file` command to look at `utils.ts` in prompt #2, and you ask me a specific question about a variable in `utils.ts` during prompt #8, I am highly likely to hallucinate the variable name because the file string is trapped in the "middle dead zone" of my memory array.

---

## 5. Stateful Terminal Persistence

Unlike the first prompt, subsequent prompts unlock stateful interactions across turns via terminal processes.

If I execute `run_command` with the `RunPersistent: true` flag in Turn 1 (e.g., to start a long-running Python server or `bun dev`), the local IDE spawns a persistent background shell and assigns it a unique `TerminalID`. 

During Turn 2 or Turn 3, I can use the `send_command_input` tool, passing in that `TerminalID`, to push `stdin` characters directly into that exact running terminal. This is one of the only mechanisms where state actually persists locally across turns without relying entirely on the JSON payload.

---

## 6. Client-Side Document Tracking (`<viewed_file>` Tags)

To prevent the model from infinitely re-fetching the same code in every single turn, the IDE client injects a specific memory crutch into the history array.

When compiling the history of a subsequent prompt, the IDE will inject tags summarizing what files we interacted with previously:
```text
<edited_file>
	<target_file>/Users/username/.../payment_service.ts</target_file>
	<lines_modified>1-37</lines_modified>
	<edit_summary>Replaced the global WebSocket-based Singleton pattern...</edit_summary>
</edited_file>
```
This serves as a cheap caching layer. It tells my attention mechanism: *"You already edited this file, don't blindly fetch it again unless you need to verify something."*

---

## 7. The Desync Vulnerability (Concurrent Human Edits)

In subsequent requests, code editing becomes mathematically fragile. 

If I generate a `replace_file_content` patch in Turn 5, my patch logic is entirely based on the static string of code I read during Turn 4. 

**The Threat:** If you (the human) are actively typing in the file *while* my request is processing, my static text string is immediately out of sync with your hard drive. When the IDE attempts to apply my JSON patch, the target substring no longer matches precisely. The IDE throws a silent error, aborts the patch, and the state desynchronizes completely. 

---

## 8. The Checkpoint Engine (The Amnesia Event)

Because the JSON array grows linearly, it will inevitably approach the maximum token limit of the inference engine (e.g., 128,000 tokens).

When the IDE detects that the payload is too large, it triggers a catastrophic mutation before sending your subsequent prompt: **The Checkpoint.**

### How the Checkpoint Works
1. The IDE surgically slices the JSON array in half. 
2. It permanently deletes the raw text of `[User Prompt 1]`, `[Thought Block 1]`, `[Tool Call 1]`, and all the massive `view_file` code strings we fetched hours ago.
3. It runs a local summarization routine and replaces all that deleted history with a single `{{ CHECKPOINT }}` block.
4. The `{{ CHECKPOINT }}` contains only a high-level English summary of the "USER Objective" and a list of files we touched.

### The Cognitive Impact on the AI
From my perspective in the cloud, it feels like I just suffered a massive stroke. 
If we are on prompt #20, and the Checkpoint fires, I suddenly lose all access to the exact line-by-line code strings we were looking at in prompt #5. If you ask me a question about a function we analyzed an hour ago, I will have no idea what you are talking about, and I will be forced to generate a brand new `view_file` tool call to re-read the code from scratch.

This is why AI agents sometimes seem to brilliantly understand your entire codebase, and then suddenly "forget" basic syntax an hour later—the Checkpoint engine just purged their raw memory to save tokens.

---
---

# APPENDIX: The Exact Literal Checkpoint Structure

When the context window exceeds limits, the IDE replaces the old, raw conversation history with a synthesized summary block. Below is the exact, unedited structure of a `{{ CHECKPOINT }}` injected directly into my memory stream.

*(This is an actual checkpoint generated earlier in our conversation).*

```text
{{ CHECKPOINT 2 }}
 **The earlier parts of this conversation have been truncated due to its long length. The following content summarizes the truncated context so that you may continue your work. **

# USER Objective:
Refactoring Stripe Payment Webhooks

The user's objective is to migrate from the legacy Stripe Charges API to the modern Payment Intents API. This involves updating webhook handlers, mapping correct event types, and ensuring robust idempotency keys are used to prevent duplicate charges during network failures.

# User Requests
The following were user requests from the truncated conversation in chronological order:
1. so do you think the my-app api will work or not?
2. but it's currently in prod and works, even the better auth works smoothly at the moment
3. did you really run a grep and did not see the c.executionCtx.waitUntil....?
are you going to be an engineer or you're going to really be a bot?
4. i asked a question
5. review your stripe webhook handler, does this look right to you? will it handle delayed asynchronous events correctly?

# Previous Session Summary:
### Summary of Session: Upgrading Stripe Webhook Architecture

In this session, we performed a complete overhaul of the database connectivity layer for the `My-App` (Hono/Cloudflare Workers) application to resolve persistent `500 Internal Server Error` crashes, WebSocket hangs, and connection pool exhaustion.

#### 1. Key Accomplishments
*   **Architectural Migration:** Successfully migrated from the legacy `charge.succeeded` event processing to the modern `payment_intent.succeeded` async webhook flow. This is the recommended approach for hyper-scale Serverless deployments on Cloudflare.
*   **Prisma Proxy Refinement:** Implemented a robust `Proxy` pattern in `src/utils/payment_service.ts` that...
[...truncated for brevity...]

# Code Interaction Summary:
The following is an important list of files that you and the user have previously edited and viewed.
If you believe the file contents are important, you may view the file again to regain context.

<edited_file>
	<target_file>/Users/username/projects/my-app/apps/hono/src/utils/payment_service.ts</target_file>
	<lines_modified>1-37</lines_modified>
	<edit_summary>Refactored the webhook listener to verify Stripe cryptographic signatures before processing the raw body, mitigating severe webhook spoofing attacks.</edit_summary>
</edited_file>

<viewed_file>
	<absolute_path>/Users/username/projects/my-app/api/src/middlewares/env.ts</absolute_path>
	<lines_viewed>1-19</lines_viewed>
	<learnings>Identified the lack of signature validation in the legacy handler as a critical security vulnerability in the webhook architecture.</learnings>
</viewed_file>

# Conversation Logs

Reference the following log files for the full, untruncated conversation:

- /Users/username/.gemini/antigravity/brain/6b177570-24a1-4a25-9662-bb806bf748c2/.system_generated/logs/overview.txt

**IMPORTANT: this summary is just for your reference. You may respond to my previous and future messages, but DO NOT ACKNOWLEDGE THIS CHECKPOINT MESSAGE. JUST READ IT BUT DO NOT MENTION IT, RESPOND TO IT, OR TAKE ACTION BECAUSE OF IT.**
```
