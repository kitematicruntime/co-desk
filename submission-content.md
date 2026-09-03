# Co-Desk — Submission content (WebMCP Challenge 2026)

Copy-paste into the Devpost fields and adapt as needed. Keep it concrete and
truthful — the judges explicitly reward specifics and penalize vagueness.

## Project name
Co-Desk — Human + Agent Support Operations

## Tagline (one line)
A support desk where the agent's call *freezes mid-execution* until the human on
the page approves it — one live queue, two operators, real veto power.

## Elevator pitch / What it is
Co-Desk is a WebMCP-native support desk built around a human approval gate. The
agent discovers the site's own operations as declared tools and works the real
queue: reads return instantly, but every consequential write — resolve, route,
escalate, batch triage — **parks inside `execute()`** and renders an approval card
on screen. The agent is genuinely blocked, mid-call, until a person clicks approve
or reject; the human's decision is what returns to the model. Reject it and the
agent is told the action was *not* applied, and asked to adjust.

That pending card is the whole point. A remote MCP server can ask for permission
in chat and hope; only a tool running inside the page the human is already looking
at can hold the call open and hand back a real verdict.

## What can people and agents do together that was hard or impossible before? (required question)
Delegating a consequential action without giving up the decision.

Before WebMCP an agent helping with a web workflow had to drive the UI by clicking
— slow, brittle, unverifiable — and any "are you sure?" step lived in the chat
window, far from the actual data. Co-Desk inverts that. The site declares its
workflow as tools with JSON Schemas and real effects, then splits them by
consequence:

- **Ungated reads** run immediately: `listOpenTickets`, `getTicket`,
  `searchTickets`, `summarizeQueue`, `findDuplicates`, `draftReply`.
- **Perception**: `getViewState` lets the agent see the page as the human sees it
  — active filter, which ticket ids are on screen, live counts, pending approval
  cards, and the text the human currently has selected.
- **Gated writes** stop and wait for a human: `updateStatus`, `assignToTeam`,
  `escalateTicket`, `resolveTicket`, and batch `triageQueue`, which renders its
  full plan — every ticket it intends to move — as one card to approve or refuse.

So a support team can hand the agent the entire routine backlog and still own
every irreversible decision, at the moment it happens, on the same screen showing
the data. The human is not notified after the fact and not asked in a side
channel: the agent is stopped, waiting, and visibly so.

## How did you implement WebMCP? (required question)
In `app.js`, `registerTools()` registers each operation against the page's
`modelContext` — probed on `document`, `navigator` and `window`, since builds
have exposed it on different holders — with `registerTool({ name, description,
inputSchema, execute })`. Every `execute` routes through one `invokeTool(name, input)`
chokepoint, which checks the tool's `gate` flag before running anything:

```js
async function invokeTool(name, input) {
  const tool = TOOLS.find((t) => t.name === name);
  const needsGate = tool.gate === true ||
    (typeof tool.gate === "function" && !!tool.gate(input));
  if (needsGate) {
    // Renders the approval card and does not settle until the human clicks.
    const ok = await requestApproval(
      tool.name, input, tool.gateWhy(input), tool.plan?.(input));
    if (!ok) {
      return "ERROR: rejected by the human — " + name +
        " was NOT applied. Ask for guidance or adjust the proposal.";
    }
  }
  return tool.run(input);
}

// requestApproval parks the call in a Promise the UI later settles:
function requestApproval(toolName, input, why, planLines) {
  return new Promise((resolve) => {
    pendingApprovals.set("A" + (++approvalSeq),
      { tool: toolName, input, why, planLines, resolve });
    renderApprovals();               // the card the human clicks
  });
}
```

Clicking approve or reject calls `decideApproval(id, ok)`, which settles that
Promise and lets `execute()` finally return the real result to the agent. Marking
a tool gated is one line at its definition — `gate: true` — so the security model
is declarative and reviewable in one place, not scattered through handlers.

Everything else is ordinary WebMCP: each tool carries a **description** the agent
reads to decide when to use it, an **inputSchema** (JSON Schema) it must satisfy,
and an **execute(input)** with real effects on the shared queue — persisted to
`localStorage`, mirrored across tabs with `BroadcastChannel`, and written to a
live activity feed that labels every entry `human` or `agent`. A header badge
flips to **"WebMCP connected — agent tools registered"** on successful
registration.

## Testing instructions (for judges)
**No browser flag? Skip to step 5 — the gate is fully demonstrable without one.**

1. Open https://kitematicruntime.github.io/co-desk/ in Chrome with WebMCP enabled:
   go to `chrome://flags/#enable-webmcp-testing`, set it to **Enabled**, and use
   the **Relaunch** button — reloading the tab alone does not apply the flag.
   (The API is exposed only in a secure context; the live URL is HTTPS.)
2. Confirm the header badge reads **"WebMCP connected — agent tools registered"**.
   To double-check in DevTools:
   `!!(document.modelContext || navigator.modelContext || window.modelContext)`.
3. **See the gate.** Ask the agent, with no hedging in the prompt:
   > *"Resolve ticket T102 — the password reset — with a one-line summary."*

   The agent calls `resolveTicket` and **stops**. An approval card appears above
   the queue showing the tool, the ticket, and why it is gated. Nothing has
   changed in the queue yet. Click **Reject** — the agent is told the action was
   not applied and will ask you how to proceed. Then repeat and click **Approve**
   to watch it complete.
4. **See a whole plan gated at once.** Ask:
   > *"Triage the entire queue and apply your routing."*

   `triageQueue` renders every ticket it intends to move as a single card. One
   click approves or refuses the batch.
5. **Without the flag**, click **▶ mock agent scenario** in the footer. A scripted
   agent walks the same code path — summarize, note, route, resolve, triage — and
   the same approval cards block it, waiting for your click. This is the identical
   `invokeTool` chokepoint the real agent uses, not a mockup of it.
6. Work the queue yourself at any time — create, route, escalate, resolve. The
   activity feed tags each action `human` or `agent`, so you can see the two
   operators interleave on one dataset.

## Built with / tech
- WebMCP (`document.modelContext.registerTool`) — the agent-facing tool layer
- A promise-parking approval gate — the agent's `execute()` is held open by a
  Promise the UI settles on click
- Plain HTML/CSS/JS — no build step, no dependencies, trivially deployable
- `localStorage` for durable queue state, `BroadcastChannel` for live cross-tab
  sync, so two windows really do show one shared queue

## Source repository & license
Public repo with an MIT LICENSE at the top (About section) so it is visible.
Repository: https://github.com/kitematicruntime/co-desk

## What's next / ambition
The gate ships today; what it needs next is durability and scale. Real
persistence and auth behind the queue. A signed, tamper-evident audit trail so
"a human approved this at 14:02" is provable after the fact, not just visible in
a feed. Per-tool scopes and spending limits an operator sets once instead of
approving every call. Policies learned from past decisions, so the gate stops
asking about the routine cases and saves the human's attention for the ones that
actually carry risk. Co-Desk is a pattern any shared-inbox or ops product can
adopt: give the agent the workload, keep the verdict.
