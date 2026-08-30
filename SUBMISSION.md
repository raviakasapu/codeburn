# Submission Statement

## Proposed title

Add Bahulam Code provider

## Summary

This pull request adds Bahulam Code as a first-class CodeBurn provider and includes fixture coverage for discovery, parsing, cost semantics, model attribution, subagent attribution, tool capture, project attribution, and provider cache invalidation.

Disclosure: I maintain Bahulam Code. This PR adds CodeBurn support for a tool I publish.

### Core behavior

1. **Cost semantics**: Distinguishes "cost reported as $0" (metered free call) from "cost absent" (no cost field). Uses `isReportedCost` presence check matching the cline-cli pattern. Reported zero-cost calls preserve `costUSD: 0` with `costIsEstimated: false`. Absent cost triggers `calculateCost` with `costIsEstimated: true`. Negative costs (invalid) are treated as absent.

2. **Multi-model attribution**: When `usage.models` contains multiple entries, the parser emits one `ParsedProviderCall` per model entry with that model's own token counts, cache stats, and cost. Model rows share a single turn id so CodeBurn keeps them as one user turn.

3. **Subagent attribution**: Bahulam stores subagent usage inside `complete.usage.models[]` rather than in sidecar transcripts. Non-root model roles such as `plan` and `explore` are emitted as `subagentTypes`, so CodeBurn can explain subagent spend without adding a second cost source.

4. **Tool events**: `tool_call`/`tool_request` events are accumulated between `complete` events. Tool names and bash commands (via `extractBashCommands`) are attached to the subsequent `complete` yield.

5. **CWD / project attribution**: `workingDirectory` and `projectPath` are set from the first `cwd` seen in the session. Source project remains the directory slug for backward compatibility.

### New files

6. **`tests/providers/bahulam.test.ts`**: Full fixture test suite covering discovery, parsing, cost semantics, multi-model, subagent attribution, cwd attribution, probeRoots, tool extraction, deduplication, corrupt-file resilience, and session_info model resolution.

### Documentation

7. **`docs/providers/README.md`**: Bahulam row added to eager provider index (alphabetical, before Claude).
8. **`README.md`**: Bahulam Code link added to supported-tools section using `assets/providers/bahulam.png`.
9. **`SUBMISSION.md`**: Updated with current PR context.

## User-visible behavior

- `codeburn --provider bahulam` and all report commands can read Bahulam Code sessions from disk.
- Reported Bahulam costs are preserved when present, including `$0` calls.
- Calls with multiple model rows are attributed per model while remaining one user turn.
- Subagent roles in model rows populate CodeBurn's subagent breakdown without double-counting total cost.
- Tool and shell-command usage from Bahulam events appears in CodeBurn's tool/activity views.
- `codeburn doctor` can show the resolved Bahulam projects root.

## Validation

- `npx vitest run tests/providers/bahulam.test.ts tests/provider-env-declarations.test.ts tests/env-isolation-declarations.test.ts tests/provider-registry.test.ts`: **54/54 passed**
- `npx tsc --noEmit`: passed
- `git diff --check`: passed
- Real-session smoke test: `npm run dev -- today --provider bahulam` completed and detected Bahulam usage: 1 current-day session, 11 calls, `$0.128` total, and a `plan` subagent row in Skills & Agents.
- Real-session smoke test: `npm run dev -- models --provider bahulam` completed and showed Bahulam Code model aggregation for DeepSeek v4 Pro and MiMo v2.5.

## Deliberate non-changes

- Bahulam remains an eager provider because it has no heavyweight optional dependencies.
- README includes the Bahulam Code provider logo asset.

## Reviewer focus

The highest-value review is the provider parser contract: reported-cost presence semantics, multi-model turn grouping, and tool attribution from `tool_call` / `tool_request` events.
