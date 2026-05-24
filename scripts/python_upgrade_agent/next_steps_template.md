<!--
Template embedded into the PR body by scripts/python_upgrade_agent/agent.py.
Placeholders use the {{name}} form so braces in code blocks do not clash with
Python's str.format. Substitutions are done via plain str.replace.

Available placeholders:
  {{current_minor}}   e.g. "3.13"
  {{new_minor}}       e.g. "3.14"
  {{current_full}}    e.g. "3.13.13"
  {{new_full}}        e.g. "3.14.5"
-->

## Next steps for the developer

This draft PR completes the mechanical bump described in Step 4 of the
Python-onboarding workflow. The items below are the remaining manual steps
needed to ship Python {{new_minor}} support. References point to the 3.14
roll-out (issue [#32869](https://github.com/Azure/azure-cli/issues/32869))
as a worked example.

### 1. Verify the prerequisite workstreams have started

These should already be in motion before the agent was dispatched. Confirm
each one before moving the PR out of draft.

- [ ] **Tracking issue filed** in `Azure/azure-cli` titled `Support Python {{new_minor}}`,
      with a checklist linked from this PR. (References: 3.14
      [#32869](https://github.com/Azure/azure-cli/issues/32869), 3.13
      [#29640](https://github.com/Azure/azure-cli/issues/29640).)
- [ ] **Quality & Productivity squad notified** to add Python {{new_minor}}
      support to `azdev` ([Azure/azure-cli-dev-tools](https://github.com/Azure/azure-cli-dev-tools))
      and `aaz-dev` ([Azure/aaz-dev-tools](https://github.com/Azure/aaz-dev-tools)).
      These tools do not run in the `azure-cli` pipeline, so a missing bump
      will not show as a red leg here — but command authors cannot develop
      locally under {{new_minor}} until both are released.
- [ ] **`knack` released with Python {{new_minor}} support and pinned in azure-cli.**
      Land a PR in [microsoft/knack](https://github.com/microsoft/knack) that
      declares {{new_minor}} in classifiers (and fixes any test-suite fallout
      under {{new_minor}}), cut a release via the ADO pipeline
      ([azclitools / release / definitionId=8](https://dev.azure.com/azclitools/release/_release?definitionId=8&view=mine&_a=releases)),
      then bump the `knack` pin in this repo. Reference: 3.14
      [#33377](https://github.com/Azure/azure-cli/pull/33377).

### 2. Triage CI failures on this PR and land fix PRs

The first CI run on this draft PR is your TODO list. Categorise each failure
into one of the buckets below and address them in roughly this order. **Each
fix should be its own PR**, reviewed and merged independently of this umbrella
draft. Rebase this PR on `dev` after each merge.

#### Category A — Bump a third-party dependency

Symptom: import error, missing `cp{{new_minor}}` wheel, or runtime error
inside a dependency.

Action: find a dependency release that ships {{new_minor}} wheels and bump
the pin in `src/azure-cli/setup.py` or `src/azure-cli-core/setup.py`.

Examples from 3.14: `msal-extensions` / `portalocker` / `pywin32`
([#32859](https://github.com/Azure/azure-cli/pull/32859)),
`urllib3` 2.7.0 ([#33351](https://github.com/Azure/azure-cli/pull/33351)).

If upstream has no {{new_minor}} release yet, file an issue against them and
record the blocker on the tracking issue.

#### Category B — Linter / style fallout

If the `azdev` release from step 1 bumped `pylint` / `astroid` (which happens
when new CPython syntax requires a newer `astroid` — e.g. PEP 750 template
strings in 3.14), new diagnostics may be enabled. Run

```bash
azdev style --pylint CLI
```

under {{new_minor}} and fix everything that pops up in a single cleanup PR.
Reference: [#33347](https://github.com/Azure/azure-cli/pull/33347) — bumped
`pylint` to 4.x and fixed C0123, E1206, E0102, W4902.

If a class of warnings is too noisy to fix at once, add the offending files
to [`linter_exclusions.yml`](../linter_exclusions.yml) and file follow-ups;
do not disable the check globally.

#### Category C — CPython behavioural change in CLI code

Real product fixes inside `azure-cli`. Patterns that have bitten us:

- **`importlib` parallel-import deadlock (3.14+).** CPython 3.14 changed
  `importlib` to raise `_DeadlockError` instead of blocking when two threads
  import the same module. The command-module loader in
  `src/azure-cli-core/azure/cli/core/__init__.py` (`_load_modules` →
  `ThreadPoolExecutor`) trips this. Fix: a `_prewarm_shared_imports()` helper
  invoked on the main thread before the pool starts, importing `azure.core`,
  `azure.mgmt.core`, `msrest`, `msrestazure` (and their common submodules)
  behind `try / except ImportError` and a `_prewarm_done` idempotency flag.
  Repro on Windows:
  `az --debug 2>&1 | Select-String "deadlock|Error loading"`.
- **Typing strictness** in modules that re-declare generic aliases or use
  `typing.Self` / PEP 695 syntax. Reference: [#33298](https://github.com/Azure/azure-cli/pull/33298).
- **`argparse` golden-file drift.** Reference: [#33314](https://github.com/Azure/azure-cli/pull/33314).
- **`http.client` header / `Accept-Encoding` differences** breaking
  header-sensitive recordings. Reference: [#33316](https://github.com/Azure/azure-cli/pull/33316).

#### Category D — Extension incompatibility

`scripts/ci/test_extensions.sh` adds every published extension under the new
interpreter. For each extension that fails:

1. File an issue against the extension owner, linked from the tracking issue.
2. Add the extension to the `ignore_list` in
   [`scripts/ci/test_extensions.sh`](../../scripts/ci/test_extensions.sh) with
   a comment pointing to the issue.
3. Remove it from the ignore list once the owner fixes it.

### 3. Bump Python in `azure-cli-extensions`

The `Azure/azure-cli-extensions` repo has its own CI pipeline that pins Python
independently. This bump is a **prerequisite** for merging this PR — once
`azure-cli` ships {{new_minor}}, users will install extensions on top of it,
so the extensions repo must be green on {{new_minor}} first.

Edit `azure-pipelines.yml` in the extensions repo:

- Change the default `UsePythonVersion@0` legs (build, lint, style, source
  tests) from `'{{current_minor}}'` to `'{{new_minor}}'`.
- Add a new matrix entry to the multi-version test job (which keeps legs for
  prior minors). For {{new_minor}} this looks like:

  ```yaml
  Python{{new_minor_dotless}}:
    python.version: '{{new_minor}}'
  ```

- Leave the `azdev verify history` / `azdev verify dependencies` legs on their
  existing pinned version (currently 3.11) unless those tools have explicitly
  been validated on {{new_minor}}.

Reference: 3.14 — [azure-cli-extensions#9843](https://github.com/Azure/azure-cli-extensions/pull/9843).

For each extension that fails on {{new_minor}} in the extensions pipeline,
identify the root cause and either fix it directly (if the change is small
and you own the area) or file an issue against the owning service team and
link it from the tracking issue. This is a build/test failure in the
extension's own source — distinct from the published-extension load check in
Category D above; the `ignore_list` in `test_extensions.sh` does not help here.

### 4. Land this PR

When all categories are addressed and the extensions bump has landed:

1. Rebase on `dev`.
2. Move this PR out of draft and request review.
3. Merge once approved.

---

*Generated from `scripts/python_upgrade_agent/next_steps_template.md`.
Tick the checkboxes above as you complete each step.*
