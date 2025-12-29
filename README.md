# create-hotfix-branch

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Create%20Hotfix%20Branch-blue?logo=github)](https://github.com/marketplace/actions/create-hotfix-branch)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub release](https://img.shields.io/github/v/release/devopspolis/create-hotfix-branch)](https://github.com/devopspolis/create-hotfix-branch/releases)

This GitHub Action creates a hotfix git branch by cherry-picking commits from a release tag range. It identifies commits labeled with "hotfix" between a base release (e.g., v1.2.0) and the target version tag (e.g., v1.2.3) and creates a new branch with those commits applied.

**Prerequisite:** The version tag (e.g., v1.2.3) must already exist before running this action.

The action:
1. Validates that the version follows hotfix format (vX.Y.Z where Z > 0)
2. Verifies the version tag exists (fails if not)
3. Identifies the base release (vX.Y.0)
4. Finds all commits labeled "hotfix" between the base release and version tag
5. Creates a new branch from the base release
6. Cherry-picks the identified commits
7. Optionally pushes the branch to the remote repository

**Important:** The version tag must be created before running this action.

## Usage

```yaml
# First, create the version tag
- name: Create version tag
  run: |
    git tag v1.2.3
    git push origin v1.2.3

# Then create the hotfix branch
- uses: devopspolis/create-hotfix-branch@v1
  with:
    version: 'v1.2.3'
    create_remote_branch: true
```

### Inputs

| Name                 | Description                                    | Required | Default |
| -------------------- | ---------------------------------------------- | -------- | ------- |
| version              | Release version (must be vX.Y.Z where Z > 0)   | Yes      | -       |
| create_remote_branch | Push branch to remote repository               | No       | false   |

### Required Permissions

When using this action in a GitHub Actions workflow, the following permissions are required:

```yaml
permissions:
  contents: write        # Required to push branches and fetch repository data
  pull-requests: read    # Required to read PR labels for commit filtering
```

If using the default `GITHUB_TOKEN`, these permissions are typically available. For more restrictive environments, ensure your token has these permissions.

### Outputs

| Name                  | Description                                      |
| --------------------- | ------------------------------------------------ |
| version               | Release version (same as input)                  |
| base_release          | Base release version (vX.Y.0)                    |
| release_type          | Release type (hotfix, feature, or unknown)       |
| commits               | Space-delimited list of cherry-picked commit SHAs|
| remote_branch_created | true if branch was pushed to remote              |
| created               | true if new branch was created, false otherwise  |
| branch_name           | Name of the created branch                       |
| commit_count          | Number of commits cherry-picked                  |

## Examples

### Create hotfix branch locally

```yaml
- name: Checkout repository
  uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Required to fetch all tags and history for commit range analysis

- name: Create version tag
  run: |
    git tag v1.2.3
    git push origin v1.2.3

- name: Create hotfix branch
  id: create-hotfix
  uses: devopspolis/create-hotfix-branch@v1
  with:
    version: 'v1.2.3'

- name: Display results
  run: |
    echo "Base release: ${{ steps.create-hotfix.outputs.base_release }}"
    echo "Commits: ${{ steps.create-hotfix.outputs.commits }}"
    echo "Branch created: ${{ steps.create-hotfix.outputs.created }}"
```

### Create and push hotfix branch to remote

```yaml
- name: Checkout repository
  uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Required to fetch all tags and history for commit range analysis

- name: Create version tag
  run: |
    git tag v1.2.3
    git push origin v1.2.3

- name: Create hotfix branch
  id: create-hotfix
  uses: devopspolis/create-hotfix-branch@v1
  with:
    version: 'v1.2.3'
    create_remote_branch: true

- name: Display results
  run: |
    echo "Hotfix branch ${{ steps.create-hotfix.outputs.version }} created"
    echo "Cherry-picked ${{ steps.create-hotfix.outputs.commits }}"
```

## Prerequisites

1. **Repository checkout** - Must checkout the repository with full history before running this action:
   ```yaml
   - uses: actions/checkout@v4
     with:
       fetch-depth: 0  # Required to fetch all tags and history
   ```

2. **Version tag must exist** - Create and push the version tag before running this action:
   ```bash
   git tag v1.2.3
   git push origin v1.2.3
   ```
   This tag defines the endpoint for the commit search range and must point to the commit on main where the hotfix release should be created from.

3. **Base release tag** - Base release tag (vX.Y.0) must exist (e.g., v1.2.0 for hotfix v1.2.3)

4. **Hotfix label** - Commits to be included must have the "hotfix" label on their associated pull requests

5. **Version format** - Version must follow semantic versioning pattern: vX.Y.Z where Z > 0

## Troubleshooting

### Error: "Version tag 'vX.Y.Z' does not exist"

**Cause**: The version tag has not been created in the repository.

**Resolution**: Create and push the version tag before running this action:
```bash
git tag v1.2.3
git push origin v1.2.3
```

The version tag must exist because it defines the endpoint for the commit search range.

### Error: "Base Release tag vX.Y.0 not found"

**Cause**: The base release tag (vX.Y.0) doesn't exist in the repository.

**Resolution**: This action requires a base release tag to branch from. For example, if you're creating v1.2.3, the tag v1.2.0 must exist. Ensure you've created the initial feature release tag for this minor version.

### Warning: "No hotfix commits found"

**Cause**: No commits between the base release and version tag have PRs labeled with "hotfix".

**Resolution**:
1. Verify that PRs for commits you want included have the "hotfix" label
2. Check that commits are actually between the base release and version tag
3. The action will still create a branch identical to the base release, allowing manual cherry-picking if needed

### Error: "Invalid version format"

**Cause**: The version doesn't follow the required vX.Y.Z format where Z > 0.

**Resolution**: Use a valid hotfix version format:
- ✅ Valid: v1.2.3, v2.0.1, v10.5.27
- ❌ Invalid: v1.2.0 (feature release, not hotfix), 1.2.3 (missing 'v'), v1.2 (incomplete)

### Notice: "Remote branch already exists"

**Cause**: The branch for this version has already been created in the remote repository.

**Resolution**: This is normal for re-runs. The action is idempotent and will skip creation if the branch already exists. If you need to recreate it, delete the remote branch first:
```bash
git push origin --delete v1.2.3
```

### Error: Cherry-pick conflicts

**Cause**: Commits cannot be cleanly applied to the base release due to conflicts.

**Resolution**: This action requires conflict-free cherry-picks. You'll need to:
1. Manually create the branch and resolve conflicts
2. Or ensure commits can be cleanly applied to the base release

## How It Works

1. **Version Validation**: Ensures the version follows hotfix format (vX.Y.Z where Z > 0). Feature releases (vX.Y.0) will cause the action to exit.

2. **Tag Validation**: Verifies that the version tag exists. If not found, the action fails with instructions to create it.

3. **Base Release Identification**: Extracts the major.minor version and constructs the base release tag (vX.Y.0).

4. **Commit Discovery**: Uses `git-matching-commits` action to find all commits labeled "hotfix" between the base release and version tag.

5. **Branch Creation**: Creates a new branch from the base release tag (unless it already exists).

6. **Cherry-picking**: Uses `git-cherry-pick-commits` action to apply the identified commits to the new branch.

7. **Remote Push**: If `remote_branch` is true, pushes the branch to the remote repository.

## Related Actions

This action uses the following internal actions:
- `devopspolis/git-tag-exists`
- `devopspolis/git-matching-commits`
- `devopspolis/git-cherry-pick-commits`

## License

The MIT License (MIT)
