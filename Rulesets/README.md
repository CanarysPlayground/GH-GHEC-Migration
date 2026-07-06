# GitHub Repository Rulesets Migration Tool

This Python script provides a comprehensive solution for exporting and importing GitHub repository rulesets across organizations. It handles branch protection rules, tag protection, and other repository governance policies with intelligent bypass actor management.

## Features

- **Export Rulesets** - Extract all rulesets from repositories in source organization
- **Import Rulesets** - Apply rulesets to repositories in target organization
- **Bypass Actor Management** - Intelligent handling of teams, users, and repository roles
- **Bypass Actor Enrichment** - Fetches detailed information about bypass actors during export
- **Automatic Resolution** - Maps bypass actors from source to target organization
- **Duplicate Prevention** - Skips rulesets that already exist in target repositories
- **Comprehensive Reporting** - Detailed CSV reports of migration status
- **Rate Limit Handling** - Automatic rate limit detection and waiting
- **Batch Processing** - Process multiple repositories from CSV file
- **Detailed Logging** - Console output with color coding and file logging

## Prerequisites

- Python 3.x
- Required packages:
  ```
  requests
  python-dotenv
  ```

Install dependencies:
```bash
pip install requests python-dotenv
```

## Setup

Create a `.env` file in the Rulesets directory:

```env
# Source organization configuration
GH_PAT=your_source_github_token
GH_ORG=source-organization-name
SOURCE_API_URL=https://api.github.com

# Target organization configuration (for import)
TARGET_GH_PAT=your_target_github_token
TARGET_GH_ORG=target-organization-name
TARGET_API_URL=https://api.github.com

# Configuration files
REPO_LIST_FILE=repos.csv
OUTPUT_DIR=exported_rulesets
REPORT_FILE=migration_report.csv

# Bypass actors handling (optional)
ENRICH_BYPASS_ACTORS=true
SANITIZE_BYPASS_ACTORS=true
REMOVE_ALL_BYPASS_ACTORS=false
```

**Important:** Add `.env` to your `.gitignore` to protect your tokens.

## CSV File Format

Create `repos.csv` (or filename specified in `REPO_LIST_FILE`):

```csv
repo_name
repository-1
repository-2
repository-3
```

**Note:** Repository names only (no organization prefix). Same CSV works for both export and import operations.

## Usage

The script has two operation modes:

### 1. Export Mode

Export rulesets from source organization repositories:

```bash
python rulesets.py export
```

**What happens:**
- Reads repository names from `repos.csv`
- Fetches all rulesets from each repository in source organization
- Enriches bypass actors with detailed information (if enabled)
- Saves each repository's rulesets to `exported_rulesets/{repo_name}-rulesets.json`
- Logs all operations to `github_rulesets.log`

### 2. Import Mode

Import rulesets to target organization repositories:

```bash
python rulesets.py import
```

**What happens:**
- Reads repository names from `repos.csv`
- Loads exported rulesets from `exported_rulesets/{repo_name}-rulesets.json`
- Resolves bypass actors to target organization equivalents
- Checks for existing rulesets to avoid duplicates
- Creates rulesets in target repositories
- Generates `migration_report.csv` with detailed results
- Logs all operations to `github_rulesets.log`

## Bypass Actors Management

Bypass actors allow specific users, teams, or roles to bypass ruleset restrictions. This script provides three strategies for handling them during migration:

### Configuration Options

**`ENRICH_BYPASS_ACTORS`** (default: `true`)
- When `true`: Fetches detailed information about bypass actors during export
- Stores team names, slugs, user IDs, and repository role details
- Enables intelligent resolution during import
- **Recommended** for smooth migrations

**`SANITIZE_BYPASS_ACTORS`** (default: `true`)
- When `true`: Removes Team bypass actors that don't exist in target org
- Keeps RepositoryRole actors (admin, maintain, write, triage, read)
- Warns about User actors that may not exist
- Fallback when enriched data is not available

**`REMOVE_ALL_BYPASS_ACTORS`** (default: `false`)
- When `true`: Removes ALL bypass actors from imported rulesets
- Use when you want clean rulesets without any bypass exceptions
- Overrides other bypass actor settings

### Bypass Actor Types

1. **Team** - GitHub teams with bypass permissions
   - Enrichment fetches: team slug, name, privacy, permissions
   - Resolution: Finds existing team by slug in target org (does NOT create teams)
   - If team doesn't exist: Skipped with warning

2. **User** - Individual GitHub users with bypass permissions
   - Enrichment stores: user ID for later resolution
   - Resolution: Validates user ID exists
   - User must be member of target organization

3. **RepositoryRole** - Repository permission levels
   - Types: admin, maintain, write, triage, read
   - These work across organizations without modification
   - Always preserved during migration

### Bypass Actor Resolution Workflow

**During Export (if `ENRICH_BYPASS_ACTORS=true`):**
```
1. Export ruleset
2. Find bypass actors in ruleset
3. Fetch detailed information via API:
   - Teams: slug, name, privacy
   - Users: user ID
   - Roles: role ID and name
4. Store enriched data in JSON
```

**During Import (if enriched data available):**
```
1. Load ruleset with enriched bypass actors
2. For each bypass actor:
   - Team: Look up existing team by slug in target org
   - User: Validate user ID
   - Role: Use same role ID (universal)
3. Build resolved bypass_actors array
4. Create ruleset with resolved actors
```

**During Import (fallback to sanitization):**
```
1. Load ruleset without enriched data
2. Remove Team bypass actors (likely don't exist)
3. Keep RepositoryRole actors
4. Keep User actors with warning
5. Create ruleset with sanitized actors
```

### Recommended Strategy

For best results:
```env
ENRICH_BYPASS_ACTORS=true
SANITIZE_BYPASS_ACTORS=true
REMOVE_ALL_BYPASS_ACTORS=false
```

This will:
- ✅ Fetch detailed bypass actor information during export
- ✅ Intelligently resolve actors during import
- ✅ Only use existing teams (no team creation)
- ✅ Preserve repository roles
- ✅ Skip unmapped actors with clear warnings

## Ruleset Components

GitHub rulesets can include:

- **Branch Protection Rules** - Require pull request reviews, status checks, signed commits
- **Tag Protection Rules** - Protect tags from deletion or modification
- **Push Restrictions** - Limit who can push to protected branches
- **Required Status Checks** - Enforce CI/CD checks before merging
- **Deletion Restrictions** - Prevent branch/tag deletion
- **Force Push Restrictions** - Block force pushes to protected branches
- **Enforcement Levels** - `active`, `evaluate` (test mode), or `disabled`
- **Bypass Actors** - Users, teams, or roles that can bypass restrictions

All of these are preserved during export and recreated during import.

## Output Files

### Exported Rulesets

Location: `exported_rulesets/{repo_name}-rulesets.json`

Each file contains an array of rulesets with complete configuration:
```json
[
  {
    "id": 12345,
    "name": "main-branch-protection",
    "target": "branch",
    "enforcement": "active",
    "bypass_actors": [...],
    "enriched_bypass_actors": [...],
    "conditions": {...},
    "rules": [...]
  }
]
```

### Migration Report

Location: `migration_report.csv` (or as specified in `REPORT_FILE`)

Columns:
- **source_repository** - Source org/repo where ruleset came from
- **target_repository** - Target org/repo where ruleset was created
- **ruleset_name** - Name of the ruleset
- **source_ruleset_id** - Original ruleset ID in source
- **target_ruleset_id** - New ruleset ID in target (or "N/A" if failed)
- **enforcement** - Enforcement level (active/evaluate/disabled)
- **target** - Ruleset target (branch/tag)
- **migration_status** - Success, Failed, or Skipped
- **details** - Additional information or error messages
- **bypass_actors_info** - Details about bypass actor resolution

### Log File

Location: `github_rulesets.log`

Contains detailed execution logs:
- Timestamp for each operation
- API requests and responses
- Bypass actor enrichment details
- Resolution results
- Success/failure messages
- Warning and error details

## Token Requirements

### For Export (`GH_PAT`)

Required scopes:
- `repo` (full control of repositories)
- `read:org` (read organization teams and members)

### For Import (`TARGET_GH_PAT`)

Required scopes:
- `repo` (full control of repositories)
- `admin:org` (manage organization rulesets and teams)

**Note:** For organization-level rulesets (not implemented in this version), you would need `admin:org` scope.

## Important Notes

### Team Handling

- **No Team Creation**: The script will NOT create teams in the target organization
- **Existing Teams Only**: Bypass actors resolve only to teams that already exist
- **Team Slug Matching**: Uses team slug for lookup (e.g., `security-team`)
- **Manual Team Migration**: Use the `../Teams/` scripts to migrate teams first if needed

### Repository Requirements

- Repositories must exist in both source and target organizations
- Repository names must match (or update CSV accordingly)
- User must have admin access to repositories
- For private repos, tokens must have access

### Enforcement Modes

- `active` - Rules are enforced immediately
- `evaluate` - Rules are checked but not enforced (test mode)
- `disabled` - Rules are inactive

Consider using `evaluate` mode first to test rulesets before activating them.

### Duplicate Detection

The script checks if a ruleset with the same name already exists before creating. If found:
- Creation is skipped
- Existing ruleset ID is logged
- Status marked as "Skipped" in report
- No modification to existing ruleset

To update existing rulesets, delete them first or use a different name.

## Examples

### Example 1: Export rulesets from source organization

```bash
# 1. Set up .env with source credentials
# GH_PAT=ghp_source_token
# GH_ORG=source-company

# 2. Create repos.csv
echo "repo_name" > repos.csv
echo "backend-api" >> repos.csv
echo "frontend-app" >> repos.csv
echo "mobile-app" >> repos.csv

# 3. Run export
python rulesets.py export
```

**Output:**
```
ℹ️ INFO: Export configuration:
ℹ️ INFO:   ENRICH_BYPASS_ACTORS: True
ℹ️ INFO: Will fetch detailed information for teams and users during export
ℹ️ INFO: Exporting rulesets from 'source-company/backend-api'...
ℹ️ INFO: Found 2 ruleset(s) on 'source-company/backend-api'.
ℹ️ INFO: Enriching bypass actors for ruleset 'main-protection'...
✅ SUCCESS: Export operation completed!
```

**Result:** 
- `exported_rulesets/backend-api-rulesets.json`
- `exported_rulesets/frontend-app-rulesets.json`
- `exported_rulesets/mobile-app-rulesets.json`

### Example 2: Import rulesets to target organization

```bash
# 1. Update .env with target credentials
# TARGET_GH_PAT=ghp_target_token
# TARGET_GH_ORG=target-company

# 2. Ensure repos.csv lists target repositories
# (can be same as source if names match)

# 3. Run import
python rulesets.py import
```

**Output:**
```
ℹ️ INFO: Bypass actors handling configuration:
ℹ️ INFO:   ENRICH_BYPASS_ACTORS: True
ℹ️ INFO:   SANITIZE_BYPASS_ACTORS: True
ℹ️ INFO:   REMOVE_ALL_BYPASS_ACTORS: False
ℹ️ INFO: Will use enriched bypass actor data from export
ℹ️ INFO: Will only use existing teams in target organization
ℹ️ INFO: Resolving enriched bypass actors for ruleset 'main-protection'...
ℹ️ INFO: Resolved Team 'security-team' to ID 54321 in target-company
✅ SUCCESS: Ruleset 'main-protection' created successfully on 'target-company/backend-api' with ID 67890
✅ SUCCESS: Import operation completed! Check 'migration_report.csv' for detailed results.
```

**Result:**
- Rulesets created in target repositories
- `migration_report.csv` with detailed status
- `github_rulesets.log` with full execution details

### Example 3: Migration without bypass actors

For clean rulesets without any bypass exceptions:

```env
REMOVE_ALL_BYPASS_ACTORS=true
```

```bash
python rulesets.py import
```

All bypass actors will be stripped from rulesets during import.

## Troubleshooting

### "GitHub token not found"
- Ensure `.env` file exists in the script directory
- Check that `GH_PAT` (for export) or `TARGET_GH_PAT` (for import) is set
- Verify token is not expired

### "403 Forbidden" errors
- Token lacks required scopes (needs `repo` and `admin:org`)
- User doesn't have admin access to repository
- Organization may have SAML SSO requiring token authorization

### "404 Not Found" errors
- Repository doesn't exist in organization
- Organization name is incorrect
- Token doesn't have access to private repository
- Team slug is incorrect or team doesn't exist

### "Ruleset already exists" (Skipped)
- A ruleset with the same name already exists in target repository
- This is expected behavior to prevent duplicates
- Delete existing ruleset first if you want to recreate it
- Or modify the ruleset name in the exported JSON

### "CSV file must have a 'repo_name' column"
- CSV header must be exactly `repo_name`
- Check for typos or extra spaces
- Ensure CSV is properly formatted

### "No rulesets found for repository"
- Repository has no rulesets configured
- User may not have permission to view rulesets
- Check that repository exists and token has access

### "Could not resolve Team 'team-name' in target org"
- Team doesn't exist in target organization
- Team slug may be different in target org
- Migrate teams first using `../Teams/` scripts
- Or set `SANITIZE_BYPASS_ACTORS=true` to skip missing teams

### "Failed to create ruleset" with validation errors
- Ruleset conditions may reference branches/tags that don't exist
- Bypass actor IDs may be invalid in target organization
- Try setting `REMOVE_ALL_BYPASS_ACTORS=true` to exclude bypass actors
- Check API error details in log file

### Rate limit warnings
- GitHub API limits: 5,000 requests/hour (authenticated)
- Script automatically waits when limit is reached
- For large migrations, consider using GitHub App tokens (15,000/hour)
- Process repositories in smaller batches if needed

## Use Cases

1. **Organization Migration** - Move rulesets when migrating repositories to new organization
2. **Multi-Repository Governance** - Apply consistent branch protection across multiple repositories
3. **EMU Adoption** - Migrate rulesets during Enterprise Managed Users transition
4. **Repository Standardization** - Enforce security policies across all repositories
5. **Disaster Recovery** - Backup and restore repository governance policies
6. **Testing Protection Rules** - Export production rulesets to test environment
7. **Compliance Requirements** - Ensure consistent protection policies for regulated environments
8. **Template Repositories** - Create standard rulesets and deploy to new repositories

## Security Best Practices

1. **Token Security**
   - Never commit `.env` files or tokens to git
   - Use fine-grained personal access tokens when possible
   - Rotate tokens regularly
   - Use separate tokens for source and target organizations

2. **Test First**
   - Export rulesets from a test repository first
   - Import to a test repository before production
   - Use `evaluate` enforcement mode for testing
   - Review `migration_report.csv` carefully

3. **Team Management**
   - Migrate teams to target organization first
   - Verify team slugs match between organizations
   - Ensure team members are present in target org
   - Consider using `REMOVE_ALL_BYPASS_ACTORS=true` initially

4. **Audit and Verify**
   - Review exported JSON files before import
   - Check migration report for failures
   - Verify rulesets in GitHub UI after import
   - Test protected branches with actual pushes

5. **Backup**
   - Keep exported JSON files as backup
   - Store migration reports for audit trail
   - Document any manual adjustments made
   - Version control your CSV and configuration files

## Limitations

- **Organization-level rulesets** - Not currently supported (only repository-level)
- **Team creation** - Script does NOT create teams; they must exist in target org
- **Ruleset updates** - Cannot update existing rulesets (only create new ones)
- **Custom properties** - Some advanced ruleset features may not be fully supported
- **API rate limits** - Large migrations may take time due to rate limits
- **Branch existence** - Does not verify that referenced branches exist in target repo

## Related Scripts

- **`../Teams/`** - Migrate teams before migrating rulesets with team bypass actors
- **`../Migration/`** - Migrate repositories themselves between organizations
- **`../Repo Permissions/`** - Manage direct collaborator permissions (different from rulesets)
- **`../Webhooks/`** - Migrate repository webhooks
- **`../Add environment secrets/`** - Manage environment-level protections and secrets

## Alternative Approaches

If this script doesn't meet your needs, consider:

1. **Manual Configuration** - For small numbers of repositories
2. **GitHub Terraform Provider** - Infrastructure as code approach
3. **GitHub CLI + jq** - For custom scripting workflows
4. **GitHub REST API directly** - For highly customized migrations
5. **Third-party tools** - Specialized migration platforms

## Summary

| Feature | Details |
|---------|---------|
| **Purpose** | Export and import GitHub repository rulesets |
| **Operations** | `export`, `import` |
| **Input** | repos.csv with repository names |
| **Export Output** | JSON files in exported_rulesets/ directory |
| **Import Output** | Migration report CSV + log file |
| **Bypass Actors** | Intelligent enrichment and resolution |
| **Team Handling** | Uses existing teams only (no creation) |
| **Duplicate Prevention** | Skips rulesets that already exist |
| **Tokens Required** | repo + admin:org scopes |
| **Configuration** | .env file with organization and token details |

---

**Note:** This script handles repository-level rulesets only. Organization-level rulesets require different API endpoints and are not currently supported.
