# Antigravity Architecture: Web Search and Browser Sub-Agents

This document provides a highly visual, step-by-step breakdown of how the Antigravity engine gathers information from the live internet and visually interacts with running applications.

To understand this, you must first accept a core architectural reality: **The inference engine is a brain locked in a jar in a Google data center.** It has no network card, no IP address, no web browser, and no screen. It is entirely dependent on your local IDE to act as its eyes, ears, and hands.

Here is exactly how the system circumvents this limitation to "browse" the internet and "see" your apps.

---

## Part 1: The Web Search Protocol (`search_web` & `read_url_content`)

When you ask the AI a question about a new documentation update, the system executes a precise fetch-and-summarize loop.

### 1. The Realization in the Cloud
The LLM processes your prompt and its internal weights determine that it lacks the necessary facts (e.g., "I don't know the Stripe API v2 syntax"). 
It generates a highly specific JSON payload intended for the IDE:
`call:default_api:search_web{query: "\"StripePaymentIntent\" \"StripeWebhook\" site:stripe.com/docs"}`

The moment this string is generated, the LLM physically halts. It goes to sleep.

### 2. The IDE Intercept and API Call
Your local IDE catches this JSON payload. It strips out the `query` parameter.
The IDE itself does not open Chrome or Firefox. Instead, it fires a raw HTTP request directly to a backend Search Engine API (such as the Google Custom Search API).

### 3. The Snippet Assembly
The Search API responds to your IDE with a JSON array containing the top 10 search results. This raw data contains titles, URLs, and short 160-character metadata "snippets" (the exact same text you see under a blue link on a Google search results page).

Your IDE takes this JSON array, formats it into a readable text block, wraps it in a `<response>` tag, and uploads that text block *back* to the sleeping LLM in the cloud.

### 4. The Blindness Limitation
The LLM wakes up and reads the snippets. 
**Crucially, the LLM has not read the actual websites.** It has only read the 160-character summaries. If the answer is buried deep in a documentation page, the snippet won't contain it.

### 5. Chaining for Deep Scrapes (`read_url_content`)
If the LLM realizes the snippet isn't enough, it initiates a second phase. It generates a new tool call using a URL it found in the previous step:
`call:default_api:read_url_content{Url: "https://stripe.com/docs/webhooks"}`

The LLM goes to sleep again. 
Your IDE intercepts this command. It performs a direct HTTP GET request to that specific URL. It pulls down the raw HTML of the entire webpage. 
Because raw HTML is too token-heavy (full of `<div>` and `<script>` tags), the IDE passes the HTML through a local parser, stripping out all the styling and converting the core text into pure Markdown.
This massive Markdown string is then uploaded back to the LLM, finally allowing the brain in the cloud to "read" the webpage.

---

## Part 2: The Browser Preview Protocol (`browser_subagent`)

This is the most complex orchestration in the entire Antigravity architecture. How does a blind LLM in a data center interact with a React app running on `http://localhost:3000`?

It uses **Sub-Agent Delegation.**

### 1. The Delegation Payload
The main LLM realizes it needs to test a UI. It cannot "open a tab." Instead, it writes a detailed set of instructions for a *completely separate* entity to execute.
It generates a payload:
```json
call:default_api:browser_subagent{
  TaskName: "Verify Login UI", 
  Task: "Navigate to http://localhost:3000. Read the page to find the email and password inputs. Type 'test@test.com' and 'password123'. Find the submit button and click it. Wait 2 seconds and report if an error modal or success toast appears in the DOM.", 
  RecordingName: "login_test_run"
}
```
The main LLM then completely halts and goes to sleep.

### 2. Spawning the Headless Browser
Your local IDE intercepts this command. In the background of your physical machine, the IDE boots up a "headless" instance of Chromium (using a framework similar to Playwright or Puppeteer). "Headless" means the browser is actually running and rendering the HTML/CSS/JS, but the graphical window is invisible to you.

### 3. The Birth of the Sub-Agent
The IDE then initializes a completely separate, specialized instance of an LLM. This is the "Sub-Agent." 
The main LLM is built for coding and system architecture. The Sub-Agent is built exclusively for navigating accessibility trees and DOM structures.

The Sub-Agent is equipped with a distinct set of tools the main LLM does not possess:
- `navigate_url`
- `click_element(id)`
- `type_text(id, string)`
- `scroll_page`

### 4. The Visual Parsing Loop
The Sub-Agent is fed the `Task` written by the main LLM. 
The headless Chromium browser takes a snapshot of the local `localhost:3000` DOM. It converts the visual page into a massive structural tree (an Accessibility Tree) and feeds it to the Sub-Agent.

The Sub-Agent "looks" at this text tree. It identifies that node `442` is an `<input>` field labeled "Email". 
It generates a tool call: `type_text(442, "test@test.com")`.
The IDE intercepts this, translates it into a Playwright command, and injects the keystrokes into the headless browser. The DOM updates. 
The Sub-Agent receives the new DOM tree. It finds node `445` (the submit button) and generates `click_element(445)`.

### 5. The Telemetry and Recording
While the Sub-Agent is puppeteering the headless browser, your IDE is capturing the frame buffer of the invisible browser window. It encodes these frames into a `.webp` video animation. 
Once the task concludes, this video is saved directly to your hard drive at `.gemini/antigravity/artifacts/login_test_run.webp`. This is the *only* mechanism in the entire system capable of generating visual session recordings.

### 6. The Death of the Sub-Agent
Once the Sub-Agent completes the task (or crashes due to an error), the IDE ruthlessly kills the headless Chromium process and shuts down the Sub-Agent instance.

### 7. The "Field Report" Uplink
The IDE complies a dense textual report detailing everything the Sub-Agent did and saw. It wraps this report in a `<response>` XML block and uploads it across the internet to the main Antigravity LLM.

### 8. The Illusion of Sight
The main LLM finally wakes up. It reads the textual field report. 
It sees that the sub-agent clicked the button and an "Invalid Credentials" div appeared. 

The main LLM then generates its response to you, the human: *"I just checked your browser. I clicked the login button and the error handling works perfectly!"*

The main LLM never saw the screen. It never saw the colors, the layout, or the CSS. It merely read a text-based action report from a temporary clone that was executed on your local machine, creating the perfect illusion that it has eyes.

---
---

# APPENDIX: The Exact Telemetry and Data Structures

Below are the exact literal payload structures sent over the network between the LLM and the IDE during Web Search and Browser Preview protocols.

### A. The Web Search Exact Structure
**1. LLM Generates Call:**
```json
call:default_api:search_web{
  "query": "StripePaymentIntent Cloudflare limit"
}
```

**2. IDE Injects Response (The Search API Return):**
```xml
<response:default_api:search_web>
## Search Results for: "StripePaymentIntent Cloudflare limit"

1. [Stripe Idempotency Keys](https://stripe.com/docs/api/idempotent_requests)
   When using Prisma with Cloudflare Workers, StripePaymentIntent provides a stateless connection wrapper that bypasses WebSocket pool limits...

2. [Stripe API: Webhook Timing Issues](https://github.com/stripe/stripe-node/issues/567)
   A common mistake is parsing the JSON body before passing it to the constructEvent function; Stripe requires the raw, unparsed string buffer...
</response>
```

### B. The Full URL Scrape Exact Structure
**1. LLM Generates Call:**
```json
call:default_api:read_url_content{
  "Url": "https://stripe.com/docs/api/idempotent_requests"
}
```

**2. IDE Injects Response (HTML to Markdown):**
```xml
<response:default_api:read_url_content>
# Stripe Idempotency Keys
*Last updated: June 2024*

To instantiate the HTTP driver:
```typescript
import { StripePaymentIntent } from '@prisma/adapter-neon'
const adapter = new StripePaymentIntent(connectionString, {})
```
[...rest of the parsed webpage...]
</response>
```

### C. The Browser Sub-Agent Exact Structure
**1. LLM Generates Call:**
```json
call:default_api:browser_subagent{
  "TaskName": "Verify Cloudflare Login Route",
  "Task": "Navigate to http://localhost:3000/login. Wait for the DOM. Type 'admin@example.com' into the email field and 'test' into the password field. Click the primary submit button. Wait 3 seconds. Report if the URL changed to /dashboard or if an error span exists.",
  "RecordingName": "app_login_test"
}
```

**2. IDE Injects Response (The Sub-Agent Field Report):**
```xml
<response:default_api:browser_subagent>
Subagent completed the task successfully.

Task Summary:
- Navigated to `http://localhost:3000/login`.
- Found input field `[name='email']` and typed 'admin@example.com'.
- Found input field `[type='password']` and typed 'test'.
- Clicked `<button type="submit">Login</button>`.
- Waited 3 seconds.

Final State:
- Current URL: `http://localhost:3000/login`
- Error observed in DOM: `<span class="text-red-500">Invalid credentials. Please try again.</span>`

Recording saved to: /Users/username/.gemini/antigravity/artifacts/app_login_test.webp
</response>
```
