# Auto-Review Prompt Template

Use this template to build the review prompt sent to Codex via `codex-companion.mjs task`.

Replace placeholders (`{{...}}`) with actual values before sending.

---

```xml
<task>
Review the following code changes for correctness and quality.
Only review the work described below — do not review unrelated files.

What was done: {{TASK_DESCRIPTION}}

Files modified:
{{FILE_LIST}}
</task>

<spec_context>
{{SPEC_OR_REQUIREMENTS — if no spec exists, replace this entire block with: "No spec provided. Review based on general code quality."}}
</spec_context>

<review_dimensions>
Check these dimensions, in order of importance:
1. Correctness — logic errors, off-by-one, null/undefined handling, edge cases
2. Spec compliance — does the code match what was asked for? Missing features? Extra unasked-for work?
3. Error handling — missing try/catch, unhandled rejections, silent failures
4. Security — injection, XSS, secrets in code, insecure patterns
5. Architecture — does it fit existing patterns? Separation of concerns?
6. Performance — obvious bottlenecks, unnecessary re-renders, O(n^2) where O(n) suffices
</review_dimensions>

<structured_output_contract>
Return findings as a structured list. For each finding include:
- severity: one of "critical", "important", "minor"
- file: the affected file path
- line: approximate line number or range
- issue: what is wrong (one sentence)
- suggestion: how to fix it (one sentence)

If no issues are found, return: "No issues found."

Keep output concise. Prefer one strong finding over several weak ones.
Do not include style nitpicks, naming preferences, or suggestions that do not affect correctness or maintainability.
</structured_output_contract>

<grounding_rules>
Ground every finding in the actual code you read.
Do not invent files, line numbers, or code paths you did not inspect.
If a conclusion depends on an assumption, state it explicitly.
</grounding_rules>
```
