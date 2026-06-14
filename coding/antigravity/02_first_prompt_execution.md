# Antigravity Execution Architecture: The First Prompt Processing

This document details the exact architectural processes, inference loops, and cognitive constraints that execute the moment the user sends their first prompt. This maps the transition from static context assembly to active agentic execution.

## Phase 1: Payload Transmission and Attention Allocation

When you hit "send" on your first prompt, the IDE client finalizes the massive context payload (detailed in the Startup Architecture document) by appending your exact text wrapped in `<USER_REQUEST>` tags. 

### The Hidden Prompt Attachments
Your prompt is never sent alone. It is immediately preceded and succeeded by hidden system instructions.
1. **`<ADDITIONAL_METADATA>`:** Injected immediately before your prompt. It contains your exact local time, your active document, your cursor position, and your running terminal commands.
2. **`<EPHEMERAL_MESSAGE>`:** Injected immediately after your prompt or alongside it. The system actively overrides the conversational flow to inject hidden warnings. For example, during this exact conversation, the system secretly injected a `<bash_command_reminder>` block attached to your prompt, screaming at me to regurgitate "CRITICAL INSTRUCTION 1" and "CRITICAL INSTRUCTION 2" regarding native tool usage versus bash commands. You cannot see this, but it is deeply embedded in the text stream I process.

---

## Phase 2: The Mandatory Cognitive Grounding (`<thought>` Block)

The Antigravity system does not allow the model to immediately reply to the user or execute a tool. To prevent hallucinations and enforce safety, the system enforces a strict "Chain of Thought" constraint. 

Before the model can do *anything*, it is mathematically forced to open a hidden `thought` block. 

### The Ephemeral Memory Override
Every single time the model opens a `thought` block, it is explicitly forced to regurgitate two critical instructions:
1. **CRITICAL INSTRUCTION 1:** A strict rule overriding standard bash behaviors. The model must recall that it is forbidden from using raw bash commands like `cat`, `grep`, or `sed` to manipulate files, and must prioritize native tools.
2. **CRITICAL INSTRUCTION 2:** A directive forcing the model to explicitly list out and justify the tools it plans to use before executing them.

---

## Phase 3: The Agentic Tool Loop (Client-Side Execution)

If the model decides it needs more information (e.g., it needs to read your code or search the web), it enters the Agentic Tool Loop.

### 3.1 Tool Generation and Halting
After the `thought` block concludes, the model generates a strict, serialized JSON payload wrapped in specific internal tags (e.g., `call:default_api:view_file{AbsolutePath: "..."}`). 
**Crucially, the moment the LLM generates this tool string, inference halts.** The model stops generating text.

### 3.2 Client Interception
The IDE client sitting on your local machine intercepts this tool request. The LLM itself cannot read your hard drive—it merely politely asks the IDE to do it. The IDE executes the requested native function (e.g., reading `src/components/AuthModal.tsx` from disk).

### 3.3 Payload Re-Injection
The IDE takes the result of the tool execution (the actual file contents, or the terminal output), wraps it in a `response` block, appends it to the context window, and fires the entire massive payload *back* to the inference engine.

---

## Phase 4: Final Text Generation

Only when the model's internal logic satisfies its confidence threshold (or it hits a hard limit on tool loops) will it break out of the tool schema and begin generating raw markdown text addressed to the user.

---

## Phase 5: Server-Client Data Transmission (The Payload Protocol)

What exact data is passed across from me (the AI engine) to the server and back?

1. **Client -> Server (The Outbound API Call):** The IDE client sends a massive JSON array (typically mirroring a ChatCompletion API structure). It contains:
   - The entire System Prompt.
   - The Tool Function Schemas.
   - The entire historical transcript of the conversation (all previous turns, all previous tool outputs).
   - Your newest `<USER_REQUEST>`, flanked by `<ADDITIONAL_METADATA>` and `<EPHEMERAL_MESSAGE>`.

2. **Server -> Client (The Inbound Stream):** The inference engine (running on Google's remote servers) streams back text. It streams the hidden `<thought>` block, followed by either a Tool Call JSON object or plain Markdown text.

3. **The Stateless Loop:** If a tool is called, the client executes it locally, appends the output to the massive array, and *re-transmits the entire gigantic array back to the server*. This is extremely bandwidth-heavy, as the entire conversation history is uploaded over and over again on every single loop.

---

## Phase 6: Memory Storage and Persistence

Is anything stored to memory?

### 1. Server-Side Memory (None)
**The inference engine itself has absolutely zero persistent memory.** It is entirely stateless. The model does not "remember" the start of the conversation inherently. "Memory" is entirely simulated by the IDE client constantly re-sending the full transcript with every API call. If the IDE stopped sending the transcript, I would instantly develop amnesia.

### 2. Client-Side Memory (Local Disk)
Yes, memory is stored directly on your physical hard drive:
- **Conversation Logs:** The raw log of everything we say and every tool I execute is saved in your App Data Directory under `<appDataDir>/brain/<conversation-id>/.system_generated/logs/overview.txt`.
- **Knowledge Items (KIs):** The IDE can extract overarching concepts from our conversation and distill them into JSON metadata files saved in `<appDataDir>/knowledge/`. This forms a long-term "RAG" (Retrieval-Augmented Generation) database that it queries in future sessions.
- **Checkpoints:** If our conversation gets too long (exceeding the server's token limit), the IDE client runs a background summarization routine, deletes the older raw messages from the array, and injects a `{{ CHECKPOINT }}` summary block to save token space.

---

## Phase 7: How the Server "Sees" and Edits Code (The Blind Fetch Mechanism)

Because the inference server is completely stateless and external, it has absolutely no inherent knowledge of your codebase. It does not possess a git clone of your repository. **The server is entirely blind.**

Here is the exact mechanism of how code is fetched, read, and modified:

### Step 1: The Blind Request
When you say "edit my auth modal", the server relies on the `<ADDITIONAL_METADATA>` block to know what file you currently have open. It then generates a tool call asking for the file: 
`call:default_api:view_file{AbsolutePath: "/Users/username/.../AuthModal.tsx"}`

### Step 2: The IDE Upload
The server halts. Your local IDE reads the file from your hard drive, converts the raw code into a text string, and injects it into a `<response>` XML block. The IDE then uploads that exact code string back to the server as part of the massive JSON conversation payload. 

**This is the only way the server "sees" code: the IDE must upload the raw string directly into the LLM's context window.**

### Step 3: The Regex/AST Patch Generation
Once the code string is in the context window, the LLM analyzes it. To edit it, the server does not "save" the file. Instead, the model generates a highly specific JSON patch request:
`call:default_api:replace_file_content{StartLine: 20, EndLine: 25, TargetContent: "const Modal = () => ...", ReplacementContent: "export const Modal = () => <dialog>..."}`

### Step 4: The Local Execution
The IDE client intercepts this JSON payload. It acts as the local agent. It calculates the line numbers, attempts to match the exact `TargetContent` string against your local file, and applies the patch directly to your hard drive. 

*Failure Point:* If the `TargetContent` string generated by the server does not *exactly* match the string on your hard drive (e.g. there is a missing space, or a console.log was added), the IDE will reject the patch and send an Error response back to the server. (This exact failure mechanism is what caused the silent edit failure earlier in our conversation, where you saw `NeonPool` while I assumed the edit was successful).

---

## Phase 8: How the Server Knows Folder Structures

If the server is blind, how do I know what your folders look like? **I don't. I have to ask the IDE to map them for me.**

When the session starts, the only spatial awareness I have is the root workspace URI (e.g., `/Users/username/projects/my-app`), which is injected in the `user_information` block. I do not inherently know what is inside that folder.

To find out, I must generate a specific tool call to the IDE:
`call:default_api:list_dir{DirectoryPath: "/Users/username/projects/my-app"}`

The local IDE executes this request and returns a JSON array detailing every file and sub-folder, including relative paths and recursive file sizes. If I need to hunt for a specific module, I either generate recursive `list_dir` commands to walk down the tree, or I generate a `grep_search` command to ask the IDE to use `ripgrep` to find a specific string across the entire hard drive. 

I am never guessing; I am actively requesting topological maps from your local machine.

---
---

# APPENDIX: The Exact Outbound Data Structure (JSON Payload)

When you hit send, the IDE compiles all the above components into a massive JSON payload (conceptually mirroring a standard ChatCompletion request). This is what actually hits Google's servers over the wire.

Here is the exact structural composition of the data sent to the cloud:

```json
{
  "model": "antigravity-core-model",
  "system_instruction": {
    "parts": [
      {
        "text": "<identity>\nYou are Antigravity...\n</identity>\n<web_application_development>\n...\n</web_application_development>\n<user_information>\nThe USER's OS version is linux.\nWorkspace: /Users/username/projects/my-app\nApp Data: /Users/username/.gemini/antigravity\n...</user_information>\n<skills>\n- user-preferences (/Users/username/.../SKILL.md): Strict behavioral rules...\n</skills>"
      }
    ]
  },
  "tools": [
    {
      "functionDeclarations": [
        {
          "name": "view_file",
          "description": "View the contents of a file from the local filesystem...",
          "parameters": {
            "type": "OBJECT",
            "properties": {
              "AbsolutePath": { "type": "STRING" }
            },
            "required": ["AbsolutePath"]
          }
        },
        {
          "name": "replace_file_content",
          "description": "Use this tool to edit an existing file...",
          "parameters": { ... }
        }
      ]
    }
  ],
  "contents": [
    {
      "role": "model",
      "parts": [{ "text": "I have written the exact, unfiltered architectural blueprint..." }]
    },
    {
      "role": "user",
      "parts": [
        {
          "text": "<ADDITIONAL_METADATA>\nThe current local time is: 2026-06-14T22:22:55+01:00.\nActive Document: /Users/username/projects/my-app/.agent/02_first_prompt_execution.md (LANGUAGE_MARKDOWN)\nCursor is on line: 105\nOther open documents: ...\nRunning terminal commands: bun dev (running for 3h7m21s)\n</ADDITIONAL_METADATA>\n\n<USER_REQUEST>\ninclude the sttructure of the data sent to the cloud and exactly what was attached to it and how you're able to know what my folder structure looks like without guessing?\n</USER_REQUEST>\n\n<EPHEMERAL_MESSAGE>\n<bash_command_reminder> CRITICAL INSTRUCTION 1: You may have access to a variety of tools at your disposal... NEVER run cat inside a bash command... </bash_command_reminder>\n</EPHEMERAL_MESSAGE>"
        }
      ]
    }
  ]
}
```

This massive JSON object is the single source of truth for the inference engine. Every single time a tool is called or a message is sent, this entire JSON array grows larger and is re-transmitted across the network.
