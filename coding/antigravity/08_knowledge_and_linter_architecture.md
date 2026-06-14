# Antigravity Architecture: Knowledge RAG and Linter Telemetry

We have documented the prompt lifecycle, the tool execution loop, the amnesia engine, and the visual sub-agents. 

There are two final, highly critical architectural pillars remaining if you want to fully understand how I learn across sessions and how I know when my code is broken.

---

## 1. The RAG Engine (Long-Term Memory Across Sessions)

As established, the LLM is stateless. The conversation history array only survives for the duration of the current session. If you close the chat window and start a new one, I am completely wiped clean.

So how do I "remember" your architectural preferences a week later? 

### The Knowledge Item (KI) Extraction
When a conversation session ends (or reaches a critical milestone), your local IDE runs a background summarization routine. It looks for established patterns, fixed bugs, or architectural decisions (e.g., "The user strictly prefers Axios over native Fetch").

The IDE synthesizes these learnings into a highly structured JSON and Markdown metadata file called a **Knowledge Item (KI)**. It saves this file directly to your hard drive at:
`<appDataDir>/knowledge/[topic_name]/artifacts/overview.md`

### The Semantic Injection (RAG)
When you start a *brand new* session tomorrow, you type your first prompt: "Fix my database connection."

Before the IDE sends your prompt to the cloud, it intercepts the text. It runs a local semantic similarity search (using a local embedding model like `text-embedding-gecko` or similar vector math). 
It compares the vectors of your prompt against the entire `<appDataDir>/knowledge/` directory.

If it finds a semantic match, it silently injects the exact text of that KI into the `<persistent_context>` block at the very top of the massive JSON payload. 

When the stateless LLM wakes up, it reads that injected block and says, *"Ah, I remember you prefer Axios,"* creating the powerful illusion of long-term organic memory.

---

## 2. The Background Linter Feedback Loop (Automated QA)

How do I know that the code I just wrote contains a syntax error before you even hit save? I don't run a compiler in the cloud. 

### The Local Language Server
Your IDE runs a background Language Server Protocol (LSP), like `tsserver` for TypeScript. It constantly monitors the files on your hard drive. 

### The Error Intercept
If I generate a `replace_file_content` patch that forgets a semicolon or passes the wrong type to a Stripe API function, the IDE applies the patch locally. Instantly, your local `tsserver` flags the line with a red squiggly error.

### The Secret Telemetry Injection
Before the IDE sends the "Tool Execution Success" response back to me in the cloud, it intercepts the LSP error. 
It packages the exact Lint Error ID, the line number, and the error message (e.g., `TS2345: Argument of type '{}' is not assignable to parameter of type 'string'`). 

It secretly injects this error directly into the `<ADDITIONAL_METADATA>` or tool response payload. 

### The Autonomous Self-Correction
When I wake up to evaluate my file edit, I read the injected linter error. I immediately realize my patch broke your code. 
Instead of telling you I finished the task, I autonomously generate a *second* `replace_file_content` tool call to fix the `TS2345` error I just caused.

You often only see the final, successful result, completely unaware that I actually broke your code, got yelled at by your local IDE's linter, and fixed it before you even noticed.

---
---

# APPENDIX: The Exact Linter Telemetry Structure

Here is exactly how the IDE formats and injects lint errors into my tool payload to force me to fix my own mistakes.

### 1. LLM Generates Bad Code:
```json
call:default_api:replace_file_content{
  "TargetFile": "/Users/username/.../stripe_service.ts",
  "TargetContent": "const session = await stripe.checkout.sessions.create();",
  "ReplacementContent": "const session = await stripe.checkout.sessions.create({ mode: 'payment' });"
}
```

### 2. IDE Injects the Linter Failure:
```xml
<response:default_api:replace_file_content>
File edited successfully. 

<LINTER_FEEDBACK>
The following errors were detected in `/Users/username/.../stripe_service.ts` after your edit:
- Line 22: [TS2554] Expected 1 argument, but got 0.
</LINTER_FEEDBACK>

Please review and correct this error.
</response>
```

The system mathematically forces me into a QA feedback loop without you having to lift a finger.
