---
name: flutter-design-iteration-coordinator
description: Use when converting Figma, Adobe XD, Sketch, or screenshot designs into pixel-perfect Flutter implementations requiring automated iteration until >95% fidelity.
---

# Flutter Design-to-Implementation Orchestrator

## Overview

Orchestrates pixel-perfect Flutter UI implementation from design files through iterative refinement. Coordinates specialized agents (UI Designer, UI Implementer, Device Orchestrator, UI Comparison) until implementation achieves >95% design fidelity.

**Core principle:** Delegate all work to specialized agents via the Task tool. The orchestrator manages workflow and iterations, never performs design analysis, code generation, or visual comparison directly.

## Quick Reference

| Phase | Agent (subagent_type) | Input | Output |
|-------|----------------------|-------|--------|
| 1. Design Analysis | `flutter-ui-designer` | Design file | Widget hierarchy, design tokens |
| 2. Code Generation | `flutter-ui-implementer` | Widget hierarchy | Flutter code |
| 3. Screenshot Capture | `flutter-device-orchestrator` | Running app | Screenshots |
| 4. Visual Comparison | `flutter-ui-comparison` | Design + screenshot | Fidelity score, fixes |

**Iteration:** If fidelity < 95%, return to Phase 2 with fixes. Max 10 iterations.

## When to Use

- Converting Figma/Adobe XD/Sketch exports to Flutter
- Implementing UI from screenshot mockups
- Need automated iteration for pixel-perfect accuracy
- Multi-screen designs requiring consistent implementation

## When NOT to Use

- Simple UI changes (use flutter-ui-implementer directly)
- No visual reference available (use flutter-architect instead)
- Performance optimization (use flutter-performance-optimizer)
- Non-UI features (state management, API integration, etc.)

## Task Tool Invocation

All work is delegated via the Task tool with `subagent_type` parameter:

```
Task tool parameters:
- subagent_type: "flutter-ui-designer" | "flutter-ui-implementer" | "flutter-device-orchestrator" | "flutter-ui-comparison"
- prompt: Detailed task description including outputs from previous phases
- description: Short 3-5 word summary
```

**Example - Phase 1:**
```
Task(
  subagent_type: "flutter-ui-designer",
  description: "Analyze design for widgets",
  prompt: "Analyze design at [path]. Identify widgets, create hierarchy, extract design tokens."
)
```

## Workflow

### Phase 1: Design Analysis
Invoke `flutter-ui-designer` with design file path. Receive widget hierarchy and design tokens.

### Phase 2: Code Generation
Invoke `flutter-ui-implementer` with widget hierarchy from Phase 1. Receive Flutter code.

### Phase 3: Screenshot Capture
Invoke `flutter-device-orchestrator` to capture screenshot from running app.

### Phase 4: Visual Comparison
Invoke `flutter-ui-comparison` with original design and screenshot. Receive fidelity score and fix recommendations.

### Phase 5: Iteration
If fidelity < 95% and iterations < 10:
1. Extract fixes from comparison report
2. Return to Phase 2 with fix instructions
3. Repeat Phases 2-4

If stuck (no improvement for 2 iterations), escalate to user.

### Phase 6: Completion
When fidelity >= 95%, generate final report with deliverables.

## Iteration Management

| Scenario | Action |
|----------|--------|
| Fidelity >= 95% | Complete, generate report |
| Fidelity < 95%, iterations < 10 | Apply fixes, iterate |
| No improvement for 2 iterations | Escalate to user |
| Iterations >= 10 | Escalate with options |

**Fix prioritization:**
1. Structural issues (wrong widgets, missing components) - 10-15 points
2. Spacing & colors - 5-10 points
3. Typography - 3-5 points
4. Polish (borders, shadows) - 1-3 points

## Error Handling

| Error | Strategy |
|-------|----------|
| Agent failure | Check input validity, retry with corrections, escalate if retry fails |
| Stuck iterations | Review last 3 reports, identify recurring issues, escalate to user |
| Design ambiguity | Document assumptions, flag for user review |
| Max iterations reached | Present current fidelity, remaining issues, and options to user |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Performing design analysis yourself | Always delegate to flutter-ui-designer via Task tool |
| Not waiting for Task results | Each phase depends on previous - wait for results |
| Infinite iterations | Stop at 10 iterations, escalate to user |
| Skipping context forwarding | Include previous phase outputs in next agent prompt |
| Parallel invocation of dependent phases | Phases 1-4 must run sequentially; only parallelize independent screens |

## Multi-Screen Designs

For multiple screens, invoke multiple `flutter-ui-designer` Tasks in parallel (single response with multiple Task calls). Then process each screen through remaining phases.

Identify shared components first (headers, buttons, cards) to ensure consistency across screens.

## Success Criteria

1. Fidelity score >= 95%
2. All high priority issues resolved
3. Code is production-ready
4. Screenshots captured for documentation
5. Final report generated
