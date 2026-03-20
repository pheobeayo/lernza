# Testing Guide: CI Pipeline & Project Automation

This guide walks you through testing every automation in the Lernza repo.
Run each test in order — later tests depend on state from earlier ones.

## Prerequisites

```bash
# Verify you're on main and up to date
git checkout main && git pull

# Verify secrets exist
gh secret list --repo lernza/lernza
# Should show: PROJECT_TOKEN

# Verify workflows exist
gh workflow list --repo lernza/lernza
# Should show: CI, Project Automation, Release, Stale Issue Cleanup
```

---

## Part 1: Create the Test Issue

We'll use one issue for all project automation tests.

```bash
gh issue create --repo lernza/lernza \
  --title "test: automation and CI pipeline verification" \
  --label "tests" \
  --body "Temporary issue for testing all automation workflows. Will be closed after testing."
```

Write down the issue number (e.g., `86`). Replace `TEST_ISSUE` below with it.

```bash
TEST_ISSUE=86  # ← change this to your actual number
```

**Wait 15 seconds**, then verify it landed in Backlog:

```bash
gh api graphql -f query='
query {
  node(id: "PVT_kwDOC20SDc4BRpIZ") {
    ... on ProjectV2 {
      items(first: 100) {
        nodes {
          content { ... on Issue { number } }
          fieldValueByName(name: "Status") {
            ... on ProjectV2ItemFieldSingleSelectValue { name }
          }
        }
      }
    }
  }
}' -q ".data.node.items.nodes[] | select(.content.number == $TEST_ISSUE) | .fieldValueByName.name"
```

**Expected:** `Backlog`

---

## Part 2: Project Automation Tests

### Test 2.1 — Assign issue → In Progress

```bash
gh issue edit $TEST_ISSUE --repo lernza/lernza --add-assignee sshdopey
```

**Wait 60 seconds** (Actions workflow needs to queue + run), then check:

```bash
# Check the workflow ran
gh run list --repo lernza/lernza --workflow project-automation.yml --limit 3 --json status,conclusion,displayTitle

# Check status on board
gh api graphql -f query='
query {
  node(id: "PVT_kwDOC20SDc4BRpIZ") {
    ... on ProjectV2 {
      items(first: 100) {
        nodes {
          content { ... on Issue { number } }
          fieldValueByName(name: "Status") {
            ... on ProjectV2ItemFieldSingleSelectValue { name }
          }
        }
      }
    }
  }
}' -q ".data.node.items.nodes[] | select(.content.number == $TEST_ISSUE) | .fieldValueByName.name"
```

**Expected:** `In Progress`

---

### Test 2.2 — Open draft PR → stays In Progress

```bash
# Create branch with a small change
git checkout -b test/automation-$TEST_ISSUE main
echo "// automation test" >> frontend/src/lib/utils.ts
git add frontend/src/lib/utils.ts
git commit -m "test: automation verification"
git push -u origin test/automation-$TEST_ISSUE

# Open as DRAFT PR
gh pr create \
  --title "test: automation verification" \
  --body "Closes #$TEST_ISSUE" \
  --label "tests,frontend" \
  --draft
```

Write down the PR number. Replace `TEST_PR` below.

```bash
TEST_PR=87  # ← change this
```

**Wait 60 seconds**, then check board:

```bash
gh api graphql -f query='
query {
  node(id: "PVT_kwDOC20SDc4BRpIZ") {
    ... on ProjectV2 {
      items(first: 100) {
        nodes {
          content { ... on Issue { number } }
          fieldValueByName(name: "Status") {
            ... on ProjectV2ItemFieldSingleSelectValue { name }
          }
        }
      }
    }
  }
}' -q ".data.node.items.nodes[] | select(.content.number == $TEST_ISSUE) | .fieldValueByName.name"
```

**Expected:** `In Progress` (draft PR doesn't push to In Review)

---

### Test 2.3 — Mark PR ready for review → In Review

```bash
gh pr ready $TEST_PR
```

**Wait 60 seconds**, check board:

```bash
gh api graphql -f query='
query {
  node(id: "PVT_kwDOC20SDc4BRpIZ") {
    ... on ProjectV2 {
      items(first: 100) {
        nodes {
          content { ... on Issue { number } }
          fieldValueByName(name: "Status") {
            ... on ProjectV2ItemFieldSingleSelectValue { name }
          }
        }
      }
    }
  }
}' -q ".data.node.items.nodes[] | select(.content.number == $TEST_ISSUE) | .fieldValueByName.name"
```

**Expected:** `In Review` (PR is non-draft, awaiting review)

---

### Test 2.4 — Convert PR back to draft → In Progress

```bash
gh pr ready $TEST_PR --undo
```

**Wait 60 seconds**, check board.

**Expected:** `In Progress`

---

### Test 2.5 — Mark ready again + request review → In Review

```bash
gh pr ready $TEST_PR

# If you have a collaborator, request their review:
# gh pr edit $TEST_PR --add-reviewer THEIR_USERNAME

# If solo, requesting review via GitHub UI works too:
# Go to PR → right sidebar → Reviewers → click the gear → select yourself
```

**Wait 60 seconds**, check board.

**Expected:** `In Review`

---

### Test 2.6 — Request changes (review) → In Progress

Do this in the **GitHub UI**:
1. Go to `https://github.com/lernza/lernza/pull/TEST_PR`
2. Click "Files changed" tab
3. Click "Review changes" (green button, top right)
4. Select "Request changes"
5. Write any comment (e.g., "test: requesting changes")
6. Click "Submit review"

**Wait 60 seconds**, check board.

**Expected:** `In Progress`

---

### Test 2.7 — Close PR without merging (still assigned) → In Progress

```bash
gh pr close $TEST_PR
```

**Wait 60 seconds**, check board.

**Expected:** `In Progress` (issue is still assigned to you)

---

### Test 2.8 — Unassign (no assignees left) → Backlog

```bash
gh issue edit $TEST_ISSUE --repo lernza/lernza --remove-assignee sshdopey
```

**Wait 60 seconds**, check board.

**Expected:** `Backlog`

---

### Test 2.9 — Open a non-draft PR directly → In Review

```bash
# Reopen the PR (or create a new one)
gh pr reopen $TEST_PR

# Mark ready if it's still draft
gh pr ready $TEST_PR 2>/dev/null
```

**Wait 60 seconds**, check board.

**Expected:** `In Review` (non-draft PR opened → issue moves to In Review from Backlog)

---

### Test 2.10 — Close issue as completed → Done

```bash
# Close the PR first
gh pr close $TEST_PR

# Wait a moment, then close the issue
sleep 5
gh issue close $TEST_ISSUE --repo lernza/lernza
```

**Wait 15 seconds** (built-in workflow, faster than Actions), check board.

**Expected:** `Done`

---

### Test 2.11 — Reopen issue → Backlog

```bash
gh issue reopen $TEST_ISSUE --repo lernza/lernza
```

**Wait 15 seconds**, check board.

**Expected:** `Backlog`

---

### Test 2.12 — Close as not planned → Invalid

```bash
gh issue close $TEST_ISSUE --repo lernza/lernza --reason "not planned"
```

**Wait 60 seconds** (Actions workflow needs to override built-in Done → Invalid), check board.

**Expected:** `Invalid`

---

### Test 2.13 — Board drag to Done → auto-closes issue

```bash
# Reopen first
gh issue reopen $TEST_ISSUE --repo lernza/lernza
```

Wait for it to go to Backlog, then:
1. Go to `https://github.com/orgs/lernza/projects/1`
2. Find issue `#TEST_ISSUE` in the Backlog column
3. **Drag it to the Done column**

**Wait 15 seconds**, then check if the issue closed:

```bash
gh issue view $TEST_ISSUE --repo lernza/lernza --json state -q '.state'
```

**Expected:** `CLOSED` (Auto-close built-in workflow fired)

---

## Part 3: CI Pipeline Tests

### Test 3.1 — Setup: create a test branch

```bash
# Reopen the issue for linking
gh issue reopen $TEST_ISSUE --repo lernza/lernza

# Create fresh branch
git checkout main && git pull
git checkout -b test/ci-pipeline
```

### Test 3.2 — Frontend-only change (contracts should skip)

```bash
echo "// ci test" >> frontend/src/lib/utils.ts
git add frontend/src/lib/utils.ts
git commit -m "test: CI pipeline verification"
git push -u origin test/ci-pipeline

gh pr create \
  --title "test: CI pipeline verification" \
  --body "Closes #$TEST_ISSUE" \
  --label "tests"
```

Write down PR number → `CI_PR`.

**Wait for CI to finish** (~2-3 minutes):

```bash
# Watch CI status
gh pr checks $CI_PR --watch
```

**Expected results in the Checks tab:**

| Check | Status | Why |
|-------|--------|-----|
| auto-label | ✅ pass | Added `frontend` label (file path) + `tests` label (title prefix) |
| pr-checks | ✅ pass | Title valid + issue linked + has labels |
| changes | ✅ pass | Detected frontend=true, contracts=false |
| frontend | ✅ pass | Ran lint + build (frontend files changed) |
| contracts | ⏭ skipped | No contract files changed |

**Verify auto-labeling worked:**

```bash
gh pr view $CI_PR --json labels -q '.labels[].name'
```

**Expected:** should include `tests`, `frontend` (auto-added by path), possibly `enhancement` or other type label.

---

### Test 3.3 — Bad PR title → fix it

```bash
gh pr edit $CI_PR --title "bad title without conventional prefix"
```

**Wait 60 seconds**, check CI:

```bash
gh pr checks $CI_PR
```

**Expected:** `pr-checks` FAILS (title validation failed)

Now fix it:

```bash
gh pr edit $CI_PR --title "test: CI pipeline verification"
```

**Wait 60 seconds**, check CI again.

**Expected:** `pr-checks` PASSES

---

### Test 3.4 — Remove all labels → label check fails

```bash
# Remove all labels
LABELS=$(gh pr view $CI_PR --json labels -q '.labels[].name' | tr '\n' ',')
for L in $(gh pr view $CI_PR --json labels -q '.labels[].name'); do
  gh pr edit $CI_PR --remove-label "$L" 2>/dev/null
done
```

**Wait 60 seconds**, check CI.

**Expected:** `pr-checks` FAILS ("PR must have at least one label")

Add a label back:

```bash
gh pr edit $CI_PR --add-label "tests"
```

**Wait 60 seconds.**

**Expected:** `pr-checks` PASSES

---

### Test 3.5 — No linked issue → fails

```bash
gh pr edit $CI_PR --body "Just a test, no issue linked"
```

**Wait 60 seconds**, check CI.

**Expected:** `pr-checks` FAILS ("PR must link to an issue")

Fix it:

```bash
gh pr edit $CI_PR --body "Closes #$TEST_ISSUE"
```

**Wait 60 seconds.**

**Expected:** `pr-checks` PASSES

---

### Test 3.6 — no-issue label bypass

```bash
gh pr edit $CI_PR --body "No issue needed" --add-label "no-issue"
```

**Wait 60 seconds.**

**Expected:** `pr-checks` PASSES (linked-issue check skipped due to `no-issue` label)

---

## Part 4: Stale Bot Test

```bash
# Trigger manually
gh workflow run stale.yml --repo lernza/lernza

# Wait for it to run
sleep 30

# Check the run
gh run list --repo lernza/lernza --workflow stale.yml --limit 1 --json status,conclusion

# View logs (look for issue listing)
RUN_ID=$(gh run list --repo lernza/lernza --workflow stale.yml --limit 1 --json databaseId -q '.[0].databaseId')
gh run view $RUN_ID --repo lernza/lernza --log 2>/dev/null | tail -30
```

**Expected:** Workflow completes successfully, lists issues with their stale status.
(No warnings/unassigns unless you have an issue that's been assigned for 3+ days with no activity.)

---

## Part 5: Release Test

```bash
# Make sure you're on main
git checkout main && git pull

# Create your first release tag
git tag v0.1.0

# Push the tag — this triggers the release workflow
git push origin v0.1.0

# Watch it
gh run list --repo lernza/lernza --workflow release.yml --limit 1 --json status --watch
```

**Wait ~3-5 minutes** for WASM builds. Then check:

```bash
# View the release
gh release view v0.1.0 --repo lernza/lernza
```

**Expected:**
- Release created at `github.com/lernza/lernza/releases/tag/v0.1.0`
- Changelog grouped by commit type
- WASM binaries attached (workspace.wasm, milestone.wasm, rewards.wasm)
- Contract sizes listed
- Contributors credited

---

## Part 6: Cleanup

```bash
# Close test issue
gh issue close $TEST_ISSUE --repo lernza/lernza --reason "not planned"

# Close test PRs
gh pr close $CI_PR --delete-branch 2>/dev/null
gh pr close $TEST_PR --delete-branch 2>/dev/null

# Delete local branches
git checkout main
git branch -D test/automation-$TEST_ISSUE test/ci-pipeline 2>/dev/null

# Delete remote branches
git push origin --delete test/automation-$TEST_ISSUE test/ci-pipeline 2>/dev/null

echo "✓ Cleanup done"
```

---

## Checklist

After running all tests, verify each passed:

| # | Test | Expected | ✓ |
|---|------|----------|---|
| 2.1 | Issue assigned | → In Progress | |
| 2.2 | Draft PR opened | stays In Progress | |
| 2.3 | PR marked ready | → In Review | |
| 2.4 | PR converted to draft | → In Progress | |
| 2.5 | PR ready + review requested | → In Review | |
| 2.6 | Changes requested | → In Progress | |
| 2.7 | PR closed (assigned) | stays In Progress | |
| 2.8 | Unassigned (none left) | → Backlog | |
| 2.9 | Non-draft PR opened | → In Review | |
| 2.10 | Issue closed (completed) | → Done | |
| 2.11 | Issue reopened | → Backlog | |
| 2.12 | Issue closed (not planned) | → Invalid | |
| 2.13 | Board drag to Done | issue auto-closed | |
| 3.2 | Frontend change | contracts skipped | |
| 3.2 | Auto-label | frontend label added | |
| 3.3 | Bad title → fix | fail → pass | |
| 3.4 | No labels → add | fail → pass | |
| 3.5 | No linked issue → fix | fail → pass | |
| 3.6 | no-issue label | bypasses check | |
| 4 | Stale bot | runs successfully | |
| 5 | Release v0.1.0 | release created + WASM attached | |
