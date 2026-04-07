# Agent Permissions System Analysis

This document provides a technical deep-dive into the permission system within `claude_code_source_code`. 

## 1. Permission Data Model

Before looking at execution logic, it's important to understand the core data types that represent permissions. The model is built around behaviors, rules, and a global context object.

### Behaviors
Every permission decision results in a `PermissionBehavior`:
- `'allow'`: The action proceeds automatically.
- `'deny'`: The action is immediately rejected.
- `'ask'`: The decision requires interactive human approval or an AI classifier fallback.

### Permission Rules (`PermissionRule` & `PermissionRuleValue`)
Users grant or restrict access by defining rules. A `PermissionRule` groups the behavior, where it came from (`source`), and what it applies to (`ruleValue`).

```typescript
export type PermissionRuleValue = {
  toolName: string
  ruleContent?: string
}

export type PermissionRule = {
  source: 'userSettings' | 'localSettings' | 'cliArg' | 'session' // ...
  ruleBehavior: 'allow' | 'deny' | 'ask'
  ruleValue: PermissionRuleValue
}
```

**Examples of Rule Values:**
- **Exact tool match**: `{ toolName: 'Bash' }`
- **Content-scoped match**: `{ toolName: 'Bash', ruleContent: 'prefix:npm test' }`
- **Agent specific match**: `{ toolName: 'Agent', ruleContent: 'Explore' }`
- **MCP external integration match**: `{ toolName: 'mcp__server1__*'} `

### The Context Object (`ToolPermissionContext`)
The active session runs with a compiled `ToolPermissionContext`. This object aggregates all the `PermissionRules` and tracks the broader mode the system is currently in.

```typescript
export type ToolPermissionContext = {
  readonly mode: PermissionMode // e.g., 'default', 'auto', 'dontAsk'
  readonly alwaysAllowRules: ToolPermissionRulesBySource
  readonly alwaysDenyRules: ToolPermissionRulesBySource
  readonly alwaysAskRules: ToolPermissionRulesBySource
  readonly shouldAvoidPermissionPrompts?: boolean
  // ...
}
```

---

## 2. Deep Dive: Global Permission Modes

Beyond individual rules, the system maintains a high-level **Permission Mode** (`PermissionMode`). When a tool wants to run and no specific allow/deny rule explicitly catches it, the permission framework delegates to the `mode` to decide what happens.

| Mode | Behavior Detail |
|------|-----------------|
| **`default` / `ask`** | Routine operations prompt the human via interactive CLI. The user types `y` or `n`. |
| **`auto`** | Uses an AI "YOLO classifier" model to silently review the intended action. Safe actions are auto-approved; risky ones are flagged. |
| **`dontAsk`** | "Strict automated mode". If an action isn't explicitly on the `alwaysAllowRules` list, it gets instantly denied without showing a UI prompt. |
| **`acceptEdits`** | File modifications and sandbox-safe code execution are permitted directly, acting as a shortcut over the YOLO classifier. |
| **`plan`** | Suspends destructive operations. Allows the agent to use read tools and draft plans, but postpones executing the finalized write operations until the overall plan is approved. |

---

## 3. Deep Dive: Tool-Level Self Checks (`checkPermissions`)

A tool can be permitted globally, but the specific input still needs to be validated. Each tool implements its own `checkPermissions()` method.

If the global rules matched an `allow` for the tool (e.g., `Bash`), the system still runs the tool's localized check. 

**Bash Tool Example:**
Even if Bash is allowed, running `sudo rm -rf` shouldn't be blindly approved. The `BashTool` parses the shell command and determines if the command implies mutation, crosses outside the active workspace directory, or impacts sensitive configurations.

```typescript
export const BashTool = buildTool({
  name: 'Bash',
  async checkPermissions(input, context): Promise<PermissionResult> {
    // Validates read vs write, verifies not mutating ~/.config paths, 
    // and explicitly rejects unauthorized destructive commands by changing the
    // result to { behavior: 'ask' } or 'deny'.
    return bashToolHasPermission(input, context);
  }
})
```

---

## 4. Deep Dive: The Auto Mode YOLO Classifier

When users want autonomy (mode: `auto`), the system introduces the **YOLO (You Only Look Once) Classifier**. Instead of pinging the human, `hasPermissionsToUseToolInner()` defers the decision to a lightweight, fast LLM call.

### How it operates:
1. **Safety Fast-pathing**: If a requested tool is specifically in a safe-list (like `FileReadTool`), or fits neatly into `acceptEdits` criteria, the classifier is bypassed to save tokens/time.
2. **Transcript generation**: For risky actions, the tool's intended action (`toAutoClassifierInput`) is packaged alongside recent conversation history.
3. **Execution & Evaluation**: The `classifyYoloAction()` hook invokes the model, receiving a confidence level and a final `shouldBlock` payload.

**Fallback Metric (`denialTrackingState`)**: If the external YOLO model continually denies the agent (e.g., the agent keeps repeatedly trying to download a sketchy payload), the system tracks a `consecutiveDenials` counter. After hitting the threshold, it breaks the cycle by blocking the agent's background loop or forcing the user to intervene interactively.

---

## 5. Deep Dive: Pre-Filtering and Unattended Agents

The permission system operates at the absolute earliest point possible to ensure safety:

### Setup Phase Filtering (`filterToolsByDenyRules`)
When the main loop initialized, the array of available `Tools` is strictly filtered. If the `alwaysDenyRules` specify `Bash`, the tool is permanently struck from the array. The LLM never even sees the tool declaration in its system prompt—making unauthorized use architecturally completely impossible.

### Headless Agent Hooks (`shouldAvoidPermissionPrompts = true`)
Sub-agents often run in the background without UI bindings. Since they can't throw an interactive `y/n` prompt onto the user's terminal:
1. The framework calls a third-party `PermissionRequest` hook system, allowing enterprise policies or custom handlers to resolve it.
2. If unresolved, `checkPermissions()` returns an `asyncAgent` rejection message and forcefully terminates the request without halting the primary thread.

---

## 6. Summary

The permission system in `claude_code` is designed as a strict, fail-closed layered architecture. It ensures safety by enforcing rules incrementally across several discrete layers:

1. **Load-time Pre-filtering**: Instantly strips explicitly blocked tools from the agent's available execution capabilities before the session even starts.
2. **Contextual Evaluation Engine**: Scans explicit `PermissionRule` definitions (allow/deny) and yields to global context parameters (`PermissionMode`).
3. **Tool-level Vetting Hooks**: Distributes responsibility to individual tools (e.g., `Bash`, `FileWrite`) allowing them to interpret intent and reject destructive parameters.
4. **Intelligent Fallback/Escalation**: Routes uncertain actions to an asynchronous AI `YOLO` oversight model (in `auto` mode) or falls back on fail-close headless hooks for background agents.

By progressively gating permissions through definitions, contextual execution verification, and intelligent safety limits, the codebase guarantees that unauthorized actions never unexpectedly mutate the underlying host environment.
