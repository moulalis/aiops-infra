---
name: rhoai-y-stream-re-onboard
description: Re-run the Y-stream onboarding pipeline for a previously onboarded release. Reuses the same Jira. Creates new PRs/MRs if previous ones were merged, or updates existing open PRs/MRs via force-push.
allowed-tools: Bash, AskUserQuestion
user-invocable: true
---

# RHOAI Y-Stream Re-Onboard

Re-runs the complete RHOAI release onboarding pipeline for a release that was **already onboarded**. Use this when the source branch has changed since the original onboarding and you need to pick up those changes.

**What it does:**
1. Loads the existing state file (reuses the same Jira)
2. Checks each step's PR/MR state (open vs merged)
3. Re-runs all 4 steps:
   - **If PR/MR is still open** → reuses the same branch, force-pushes to update the existing PR/MR
   - **If PR/MR was already merged** → creates a new branch and new PR/MR
4. Steps: RBC Release → RBC Main → Konflux → PipelineRun Replicator

## Prerequisites

- Same as `/rhoai-y-stream-onboarding`
- An existing state file from a previous onboarding run (`rhoai-release-rhoai-*-state.json`)
- `GITHUB_TOKEN`, `KONFLUX_REPO_TOKEN`, `JIRA_API_TOKEN` set
- **VPN active** for Konflux step

## Usage

```
/rhoai-y-stream-re-onboard
```

## Implementation

SKILL_DIR is the absolute path of the directory containing this SKILL.md.
COMMON_SCRIPTS_DIR is `<SKILL_DIR>/../common/scripts`.

---

## Step 1: Find existing state file

Look for an existing state file in the current directory:

```bash
STATE_FILES=(rhoai-release-rhoai-*-state.json)
```

If no state files found, tell the user: "No existing state file found. Run `/rhoai-y-stream-onboarding` first to do the initial onboarding."

If multiple state files exist, use AskUserQuestion to let the user pick which one.

If exactly one state file exists, use it automatically.

Store the selected file path in `STATE_FILE`.

## Step 2: Confirm re-onboard

Read the state file and show the user what release is about to be re-onboarded:

```bash
cat "$STATE_FILE"
```

Parse the JSON and display:
> **Re-onboard Release:**
> - Previous: `<previous_version>`
> - New: `<new_version>`
> - Jira: `<parent_url>`

Use AskUserQuestion to ask:
- "Proceed with re-onboard?"
- Options: "Yes — re-onboard (Recommended)", "No — cancel"

If cancelled, exit.

Ask if this should be a dry run:
- "Should this run in dry-run mode?"
- Options: "No — execute for real (Recommended)", "Yes — dry-run only"

Store in `DRY_RUN`.

## CRITICAL EXECUTION RULES

**You MUST execute this pipeline as exactly 4 SEPARATE Bash calls (Steps 3–6).**
**NEVER combine multiple steps into a single Bash call.**
**After EACH Bash call, you MUST read the state file and display a progress summary as a regular text message.**

Bash output gets collapsed and the user cannot see it. The only way the user sees progress is through your text messages between Bash calls.

### How to display progress (do this after EVERY Bash call)

1. Read the STATE_FILE (use Bash: `cat "$STATE_FILE"`)
2. Parse the JSON
3. Display progress as a **regular text message**:

> **Re-Onboard Pipeline Progress [N/4]**
> ✅ Step 1 — RBC Release — <pr_url>
> ⏳ Step 2 — RBC Main — Pending
> ⏳ Step 3 — Konflux — Pending
> ⏳ Step 4 — PipelineRun Replicator — Pending

Use ✅ for `done` (include URL), ❌ for `failed`, ⏳ for `pending`.

---

## Step 3: Bash call 1 — Execute Step 1 (RBC Release)

```bash
SKILL_DIR="<absolute path to this SKILL.md's directory>"
COMMON_SCRIPTS_DIR="$SKILL_DIR/../common/scripts"
STATE_FILE="<selected state file>"
CMD="uv run --script $COMMON_SCRIPTS_DIR/run_y_stream_pipeline.py --resume $STATE_FILE --re-onboard --single-step"
if [[ "$DRY_RUN" == "yes" ]]; then CMD="$CMD --dry-run"; fi
eval "$CMD"
```

**After this Bash call completes:** Read STATE_FILE. Display progress as text. Then proceed to Step 4.

## Step 4: Bash call 2 — Execute Step 2 (RBC Main)

```bash
uv run --script "$COMMON_SCRIPTS_DIR/run_y_stream_pipeline.py" --resume "$STATE_FILE" --single-step
```

**Note:** `--re-onboard` is only needed on the first call (Step 3) — it resets all steps and stores re-onboard info in the state file. Subsequent `--resume` calls pick up the re-onboard info from state.

**After this Bash call completes:** Read STATE_FILE. Display progress as text. Then proceed to Step 5.

## Step 5: Bash call 3 — Execute Step 3 (Konflux)

```bash
uv run --script "$COMMON_SCRIPTS_DIR/run_y_stream_pipeline.py" --resume "$STATE_FILE" --single-step
```

**After this Bash call completes:** Read STATE_FILE. Display progress as text. Then proceed to Step 6.

## Step 6: Bash call 4 — Execute Step 4 (PipelineRun Replicator)

```bash
uv run --script "$COMMON_SCRIPTS_DIR/run_y_stream_pipeline.py" --resume "$STATE_FILE" --single-step
```

**After this Bash call completes:** Read STATE_FILE. Display progress as text. Then proceed to Step 7.

## Step 7: Final summary

Read the STATE_FILE one last time and display:

> 🎉 **RHOAI Release Re-Onboard Complete!**
>
> **Release:** `<previous_version>` → `<new_version>`
>
> 📋 **Jira:** `<parent_url>`
>
> **Pull Requests / Merge Requests:**
> 1. RBC Release — `<pr_url>`
> 2. RBC Main — `<pr_url>`
> 3. Konflux — `<mr_url>`
> 4. PipelineRun Replicator — `<run_url>`
>
> **Next Steps:**
> 1. Review and merge all PRs/MRs
> 2. Monitor CI/CD pipeline execution
> 3. Verify builds are successful

**Error handling:** If any step fails (exit code != 0), stop immediately. Display the progress showing which step failed (❌) and tell the user to fix the issue and re-run.

---

## Error Reference

| Error | Action |
|-------|--------|
| No state file found | Run `/rhoai-y-stream-onboarding` first |
| `GITHUB_TOKEN` not set | `export GITHUB_TOKEN=your-token` |
| `KONFLUX_REPO_TOKEN` not set | `export KONFLUX_REPO_TOKEN=your-token` |
| `JIRA_API_TOKEN` not set | `export JIRA_API_TOKEN=your-jira-token` |
| Step failed | Check output, fix issue, re-run skill |
| VPN not active | Connect to VPN before Konflux step |
