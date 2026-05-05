---
name: local-council-execution
description: Executes a local Claude-subagent council when no vendor API keys are configured (or when --local is passed). Each subagent takes on a different role from config/roles.json (security auditor, devil's advocate, simplicity champion, etc.) and answers the question independently. Synthesis cross-references their perspectives.
---

# Local Council Execution

When no provider keys are configured, or `--local` is passed explicitly, the
council runs entirely as Claude subagents — each pinned to a different role
from `config/roles.json`. Diversity comes from role-prompting, not from
cross-vendor priors.

This is weaker signal than a true cross-model council (same priors, same
training data) but still produces meaningfully different angles, especially
for "what am I missing" or "poke holes in this" questions. It's also the
default fallback so users can try the plugin before wiring up paid keys.

## Step 1: Determine Roles

### Pick the roles config

If `--domain=NAME` was passed (e.g. `--domain=research`), use
`${CLAUDE_PLUGIN_ROOT}/config/roles-${NAME}.json`. Otherwise default to
`${CLAUDE_PLUGIN_ROOT}/config/roles.json` (software roles).

```bash
# Default
ROLES_JSON="${CLAUDE_PLUGIN_ROOT}/config/roles.json"

# Or with --domain=research
ROLES_JSON="${CLAUDE_PLUGIN_ROOT}/config/roles-research.json"
```

If the chosen file doesn't exist, error out and tell the user which domain
files are available:

```bash
ls "${CLAUDE_PLUGIN_ROOT}/config/" | grep -E '^roles(-.*)?\.json$'
```

### Pick the role set

Priority order:

1. If `--roles` was specified, use it (preset name or comma list).
2. Otherwise default to the `balanced` preset (defined per domain — for
   software it's `security,performance,maintainability`; for research it's
   `steelman,evidence-auditor,audience-loss`).

For each role, extract display name and prompt:

```bash
jq -r --arg r "<role>" '.roles[$r].name' "$ROLES_JSON"
jq -r --arg r "<role>" '.roles[$r].prompt' "$ROLES_JSON"
```

If a preset name was passed, expand it via `.presets[$name]`.

## Step 2: Spawn Role Agents in Parallel

Launch ALL role agents in a **single message** (multiple Agent tool calls)
with `subagent_type: "general-purpose"` and `run_in_background: true` for
parallel execution.

**Agent prompt template** — fill in `{ROLE_NAME}`, `{ROLE_PROMPT}`,
`{QUESTION}`, and `{FILE_CONTEXT}` (omit the file context block if no
context was gathered):

```
You are part of a council of advisors. Your role: {ROLE_NAME}.

{ROLE_PROMPT}

## Question

{QUESTION}

{FILE_CONTEXT}

## Your task

Answer the question from your role's perspective. Be specific and
actionable. Do not hedge or caveat — your job is to surface the angle
others might miss. Code examples welcome where they sharpen the point.
4-8 paragraphs is typical.

Return ONLY your analysis. No preamble, no meta-commentary about the role.
```

**CRITICAL**: Spawn all agents in one message so they run in parallel.
Sequential dispatch defeats the purpose.

## Step 3: Collect Results

Wait for all background agents to complete. You will be auto-notified.
If an agent fails or returns nothing, note the failure and continue with
whatever responses came back.

## Step 4: Display Results

For each role, display using markdown:

```
---
## {ROLE_NAME} (local)

{response}
```

Use a consistent ordering — same order as the `--roles` list (or the
expanded preset order).

## Step 5: Synthesis

Generate a synthesis section that mirrors standard mode:

1. **Consensus** — points where roles agreed
2. **Divergence** — where they disagreed and what assumption drove it
3. **Unique insights** — notable points only one role surfaced
4. **Recommendation** — strongest action given the spread

Note in the synthesis that this is a local council (single-model, multi-role)
so consensus is weaker signal than a cross-vendor consensus would be. Flag
when all roles agree on something — that's likely a Claude-prior, not a
genuinely robust conclusion.

## Step 6: Save Output

```bash
mkdir -p .claude/council-cache
```

Write the full output (all role responses + synthesis) to
`.claude/council-cache/council-local-{TIMESTAMP}.md` where TIMESTAMP is the
current Unix timestamp.

Tell the user:
> ---
> Local council output saved to `.claude/council-cache/council-local-{TIMESTAMP}.md`

## Error Handling

- If a role agent fails, show the error and continue with the others
- If ALL agents fail, report clearly and suggest retrying
- If `roles.json` is missing or malformed, error out with the path
