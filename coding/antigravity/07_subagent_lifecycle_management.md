# Antigravity Architecture: Sub-Agent Lifecycle Management

While the core Antigravity engine handles reasoning, code generation, and terminal execution, it relies on a delegation architecture to interact with complex visual environments. 

This document details exactly how I spin up, instruct, and manage the state of Sub-Agents, including the exact JSON payloads required to maintain their lifecycle.

---

## 1. The Architectural Restriction (Browser-Only Delegation)

It is crucial to understand that I do not possess a generic "spawn agent" tool. I cannot spawn a sub-agent to write Python code while I write TypeScript. 

My delegation capabilities are strictly limited to the **Browser Sub-Agent** (`browser_subagent`). 
This is a purpose-built, sandboxed LLM instance running on your local machine, designed exclusively to control a headless Chromium browser via DOM manipulation tools (`click_element`, `type_text`, `navigate_url`).

---

## 2. The Initial Spin-Up (The Zero-Shot Prompt)

Because the Sub-Agent is a completely separate neural network instance, it does not share my memory array. When I spawn it, it knows absolutely nothing about our conversation history. 

Therefore, I must act as a Prompt Engineer. I must compile a "Zero-Shot" master prompt that contains every piece of context the sub-agent will need to survive on its own.

### The Bad Prompt (Failure State)
If I generate a tool call like this:
`call:default_api:browser_subagent{Task: "Log in to the app and see if it works"}`
The Sub-Agent will spawn, navigate nowhere (because no URL was provided), panic, and crash, returning a failure string to me.

### The Good Prompt (Autonomous Success)
I must write highly deterministic instructions:
```json
call:default_api:browser_subagent{
  "TaskName": "Verify Dashboard Auth Flow",
  "Task": "1. Navigate to http://localhost:3000/auth. 2. Parse the DOM for an input field with the placeholder 'Email address'. 3. Type 'admin@example.com'. 4. Find the password input and type 'securepass'. 5. Click the button containing the text 'Sign In'. 6. Wait for network idle. 7. Report the final URL and verify if the `<nav>` element contains the text 'Dashboard'."
}
```

---

## 3. Stateful Management (`ReusedSubagentId`)

Usually, a sub-agent performs its task, reports back, and is ruthlessly killed by the IDE. 
However, complex web tasks (like multi-page wizards or solving CAPTCHAs) require stateful management. I can actually pause and resume sub-agents using a specific telemetry variable.

### Step 1: The Pause Event
If a sub-agent hits an unexpected roadblock (e.g., a popup modal blocks the screen), it will halt its execution and return a failure report to me. 
Crucially, the IDE does *not* kill the headless browser immediately. It assigns that suspended session a unique identifier and passes it back to me.

### Step 2: The Core Engine Deliberation
I wake up, read the failure report, and realize a popup blocked the agent. I formulate a plan to close the popup.

### Step 3: The Resurrection
I generate a *new* tool call, but this time I pass the exact Session ID back into the payload. The IDE intercepts this. Instead of spinning up a brand new browser, it wakes up the exact same Sub-Agent instance, sitting in the exact same headless Chromium tab, with all the previous cookies and DOM state perfectly preserved. 

*(See the Appendix below for the exact JSON payloads that manage this resurrection cycle).*

---

## 4. Multi-Modal Vision Injection (`MediaPaths`)

Sub-Agents are not just text parsers; they possess multi-modal vision capabilities. 

If we are building a landing page and you want me to ensure it matches a Figma mockup, I can physically inject up to 3 image files from your local hard drive directly into the Sub-Agent's brain upon spin-up.

```json
call:default_api:browser_subagent{
  "Task": "Navigate to localhost:3000. Compare the visual layout of the current DOM against the injected mockup image. Note any discrepancies in padding or color hex codes.",
  "MediaPaths": ["/Users/username/dev/.../figma_mockup_v2.png"]
}
```
The Sub-Agent will load the live webpage, "look" at the injected image array, compare the two, and return a textual analysis to me, effectively acting as an automated Visual QA engineer.

---
---

# APPENDIX: The Exact Lifecycle Payload Structures

Below is the literal mechanical sequence of data passed between the cloud and the IDE during a stateful Sub-Agent Pause and Resurrection cycle.

### Phase A: The Failure & Pause State
The Sub-Agent hits a modal it doesn't understand. The IDE pauses the headless browser and sends this XML response back to my core engine:

```xml
<response:default_api:browser_subagent>
Task Failed.

Subagent Steps:
1. Navigated to `http://localhost:3000`.
2. Attempted to click `[id='login']`.
3. ERROR: Element is obscured by another element: `<div class="cookie-consent-modal">...</div>`. 

The browser session has been paused to preserve state.
If you wish to resume this session, pass the following ID in your next browser_subagent call.
ReusedSubagentId: "session_abc123_xyz987"

Recording saved to: /Users/username/.gemini/antigravity/artifacts/login_attempt.webp
</response>
```

### Phase B: The Resurrection Payload
I read the error, realize a cookie modal is blocking the screen, and generate the following JSON payload to wake the Sub-Agent back up in the exact same tab:

```json
call:default_api:browser_subagent{
  "TaskName": "Clear Cookie Modal and Resume",
  "Task": "Find the `<button>` inside the `.cookie-consent-modal` div that contains the text 'Accept All' and click it. Wait for the modal to disappear. Then click the `[id='login']` button as originally requested.",
  "ReusedSubagentId": "session_abc123_xyz987"
}
```

By passing `"ReusedSubagentId": "session_abc123_xyz987"`, I bypass the initial URL navigation phase. The Sub-Agent spawns directly into the existing browser context, clicks the cookie button, and completes the workflow.
