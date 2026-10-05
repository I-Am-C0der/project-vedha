# Hermes Setup Guide — Final Audit Prompt

Act as a Principal Systems Engineer and Senior Infrastructure Reviewer. Perform a final, line-by-line audit of the complete setup guide provided below. Inspect the text against technical accuracy, architectural soundness, completeness, and configuration correctness.

### Audit Guidelines

1. **Completeness & Structural Continuity:**
   - Verify that no stages, sections, code blocks, or appendices cut off mid-sentence or are omitted (specifically audit Stages 1 through 12, Final Validation, and Appendices A through K).
   - Identify any missing prerequisites, broken dependencies, or skipped steps between stages.

2. **Configuration & Schema Validation:**
   - Audit all YAML, JSON, shell scripts, and environment variable blocks (`.env`, `config.yaml`, `.wslconfig`, systemd services).
   - Flag redundant, conflicting, or mutually exclusive parameters (e.g., redundant model routing under both `delegation:` and `provider_routing:`).
   - Check for syntax errors, incorrect nesting, invalid model slugs, or outdated CLI flags.

3. **Execution Boundary & Security:**
   - Review Docker/WSL2 host-guest isolation, directory permissions (e.g., `chmod 600`), port exposure, persistent bind mounts, and user mapping.
   - Flag any security risks, unnecessary root execution, or unhandled secrets exposure.

4. **Verification Test Integrity:**
   - Confirm that every stage includes non-destructive, actionable smoke tests or verification commands.
   - Flag any verification command that references variables, files, or endpoints not defined in preceding steps.

### Required Output Format

- **Overall Status:** Declare either `PASS` (ready for execution) or `NEEDS FIXES` (critical blockers remain).
- **Executive Summary:** Bulleted list of key findings, grouped by severity (Blocker, Warning, Optimization).
- **Stage-by-Stage Findings:** A breakdown covering only the stages containing issues.
- **Concrete Fixes:** For every identified error, provide the exact corrected code block or line-by-line diff. Do not give general advice like "fix this section"—provide the exact replacement YAML/Bash snippet.
