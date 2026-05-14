# Committing the Graph Artifact

Once the user confirms they want the artifact committed, store it on the `radius-graph`
orphan branch following these rules exactly.

## Storage location

| Property | Value |
|---|---|
| Branch | `radius-graph` (orphan — contains ONLY graph artifacts, no source code) |
| File path | `<source-branch>/app.json` |
| Commit message | `Generate app graph for <source-branch>` |

Where `<source-branch>` is the name of the branch that contains the `app.bicep` file
(e.g. `main`, `add-radius-app-definition`).

## Steps

### 1. Check if the orphan branch exists

```bash
git ls-remote --heads origin radius-graph
```

### 2a. If the branch does NOT exist — create it as an orphan

```bash
git checkout --orphan radius-graph
git rm -rf .
git commit --allow-empty -m "Initialize radius-graph artifact branch"
git push origin radius-graph
git checkout <source-branch>
```

### 2b. If the branch EXISTS — fetch it

```bash
git fetch origin radius-graph:radius-graph
```

### 3. Write the artifact file

Using the GitHub Contents API (preferred for agents without a full git checkout):

```bash
# Get the current SHA of the file if it exists (needed for updates)
EXISTING_SHA=$(gh api repos/{owner}/{repo}/contents/{source-branch}/app.json \
  --ref radius-graph --jq '.sha' 2>/dev/null || echo "")

# Write the file (create or update)
if [ -z "$EXISTING_SHA" ]; then
  gh api repos/{owner}/{repo}/contents/{source-branch}/app.json \
    --method PUT \
    --field message="Generate app graph for {source-branch}" \
    --field content="$(base64 -w0 < app.json)" \
    --field branch="radius-graph"
else
  gh api repos/{owner}/{repo}/contents/{source-branch}/app.json \
    --method PUT \
    --field message="Generate app graph for {source-branch}" \
    --field content="$(base64 -w0 < app.json)" \
    --field sha="$EXISTING_SHA" \
    --field branch="radius-graph"
fi
```

### Alternative: using git worktree

```bash
# Add a worktree for the radius-graph branch
git worktree add /tmp/radius-graph-worktree radius-graph

# Create the directory and write the file
mkdir -p /tmp/radius-graph-worktree/<source-branch>
cp app.json /tmp/radius-graph-worktree/<source-branch>/app.json

# Commit and push
cd /tmp/radius-graph-worktree
git add <source-branch>/app.json
git commit -m "Generate app graph for <source-branch>"
git push origin radius-graph

# Clean up
git worktree remove /tmp/radius-graph-worktree
```

## After committing

Tell the user:
> The graph artifact has been committed to the `radius-graph` branch at `<source-branch>/app.json`.
> Install the [Radius GitHub Extension](https://github.com/radius-project/github-extension) to
> see the interactive graph on this repository's pull requests and in the **Applications** tab.

## Notes

- The `radius-graph` branch is an orphan: it has no parent commits and contains ONLY artifact JSON files.
- Never mix source code with artifacts on this branch.
- The directory structure is flat: one subdirectory per source branch, one `app.json` per app.
- The GitHub Extension fetches `{source-branch}/app.json` from the `radius-graph` branch
  using the GitHub Contents API with `?ref=radius-graph`.
