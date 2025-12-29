# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a GitHub composite action that automates the creation of hotfix branches by cherry-picking commits labeled "hotfix" from a release tag range. The action is designed to work within the devopspolis organization's release workflow.

## Key Architectural Concepts

### Release Versioning Model

The action implements a specific semantic versioning pattern:
- **Hotfix releases**: vX.Y.Z where Z > 0 (e.g., v1.2.3)
- **Feature releases**: vX.Y.0 (e.g., v1.2.0) - these exit early as they don't need hotfix branches
- **Base release**: The vX.Y.0 tag that serves as the branching point

### Workflow Dependencies

This action is part of a larger ecosystem and depends on three companion actions:
- `devopspolis/git-tag-exists`: Validates that required tags exist
- `devopspolis/git-matching-commits`: Finds commits with specific labels between two refs
- `devopspolis/git-cherry-pick-commits`: Applies selected commits to a branch

When modifying this action, be aware that changes may require coordinated updates to these dependencies.

### Critical Prerequisite: Version Tag Must Exist

Unlike typical branching workflows, this action requires the version tag (e.g., v1.2.3) to exist BEFORE execution. The tag defines the end ref for the commit search range. The action will fail with clear instructions if the tag is missing (action.yml:114-122).

### State Management via Environment Variables

The action uses GitHub Actions environment variables extensively to pass state between steps:
- All outputs are set in `$GITHUB_ENV` and referenced in later steps
- Default outputs are initialized at the start (action.yml:46-55)
- This pattern allows skipping steps while maintaining valid outputs

## Linting and Code Quality

The repository uses Trunk for linting and code quality checks:

```bash
# Run all linters
trunk check

# Run specific linter
trunk check --filter=actionlint
trunk check --filter=yamllint
trunk check --filter=markdownlint
```

Enabled linters:
- `actionlint`: GitHub Actions workflow validation
- `checkov`: Infrastructure as code security scanning
- `git-diff-check`: Git whitespace checking
- `markdownlint`: Markdown formatting
- `prettier`: Code formatting
- `trivy`: Security vulnerability scanning
- `trufflehog`: Secret detection
- `yamllint`: YAML linting

## Testing the Action Locally

Since this is a composite action, you cannot run it directly. To test:

1. Create a test repository or use an existing one with:
   - A base release tag (e.g., v1.2.0)
   - Commits with PRs labeled "hotfix"
   - A version tag pointing to the target commit (e.g., v1.2.3)

2. Create a workflow file that uses the action:
```yaml
- uses: devopspolis/create-hotfix-branch@your-branch
  with:
    version: 'v1.2.3'
    create_remote_branch: false
```

3. Use `act` (https://github.com/nektos/act) for local testing:
```bash
act -j your-job-name
```

## Important Implementation Details

### Branch Existence Check

The action checks if the remote branch already exists (action.yml:96-106) and skips creation if found. This is idempotent behavior that prevents errors on re-runs.

### Empty Hotfix Handling

If no commits with the "hotfix" label are found, the action creates a branch identical to the base release and issues a warning rather than failing (action.yml:166-178). This allows for manual cherry-picking if needed.

### Git Configuration

The action configures git with generic GitHub Actions credentials (action.yml:90-94). If you need to trigger other workflows from the pushed branch, you'll need to modify this to use a PAT.

### Output Variable Names

The action has inconsistent naming between README and implementation:
- Input: `create_remote_branch` (action.yml:14)
- README documents: `remote_branch` (README.md:36)
- Output: `remote_branch_created` (action.yml:36)

When making changes, maintain backward compatibility or update documentation accordingly.

## Automated Major Version Tags

The workflow `.github/workflows/create-major-version-tag.yml` automatically creates/updates major version tags (e.g., v1) when a release is published. This allows users to reference `@v1` for automatic updates within the major version.
