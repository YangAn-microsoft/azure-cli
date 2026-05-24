## Automated Python upgrade: {current_minor} → {new_minor}

This PR was opened by the **Python Upgrade Agent**. See
[`scripts/python_upgrade_agent/README.md`](../scripts/python_upgrade_agent/README.md)
for the full design.

**Embedded Python**: `{current_full}` → `{new_full}` (latest stable patch on python.org)

### Files updated ({n_edits})
{edits_list}

### Locations the agent did NOT update — please review

These are matches for the previous minor that the agent was uncertain about.
Tick each item once you've decided whether to bump it or leave as-is.

{skipped_list}

### Post-check: possible forgotten references

A deterministic sweep ran after the LLM's plan was applied in memory; the lines
below still match the previous minor and were **not** in the skipped list, so
they may have been overlooked. Please verify — these are most often historical
comments or test fixtures, but occasionally indicate a missed bump.

{forgotten_list}

### Agent metadata
- **Model**: `{model}`
- **Run**: {run_url}
- **Reference PRs used as few-shot examples**: {reference_prs}
- **Notes from the agent**: {notes}

---

*This is Phase 1 of the agent: mechanical bumps only. CI failures are handed off
to humans. See the plan document for the full phased roadmap.*
