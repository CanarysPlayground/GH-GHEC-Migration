# GitHub Teams Management Scripts

This folder contains Python scripts for comprehensive GitHub team management including exporting team configurations, recreating teams in target organizations, and assigning repositories to teams.

## Scripts

1. **`get-teams.py`** - Exports all team details, members, and repository permissions from a source organization
2. **`team-recreation.py`** - Recreates teams with members in a target organization from CSV
3. **`adding-repo-to-team.py`** - Assigns repositories to teams with specific permissions from CSV

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

Create a `.env` file in this directory:

```env
# For exporting teams from source organization
GH_PAT=your_source_github_token
GH_ORG=source-organization-name

# For creating teams in target organization
TARGET_GH_PAT=your_target_github_token
TARGET_GH_ORG=target-organization-name

# CSV file configuration (optional)
INPUT_CSV_FILE=github_teams.csv
```

**Important:** Add `.env` to your `.gitignore` to prevent exposing tokens.

## Script 1: Export Teams (`get-teams.py`)

### Purpose
Exports comprehensive team information from a source GitHub organization, including team hierarchy, direct members (excluding inherited), repository permissions, and all metadata needed for recreation.

### Usage

```bash
python get-teams.py
```

**With options:**
```bash
# Estimate API calls without fetching data
python get-teams.py --estimate-only

# Custom rate limit delay
python get-teams.py --rate-limit-delay 1.0

# Custom output file
python get-teams.py --csv-file my_teams.csv

# Override organization from command line
python get-teams.py --org my-organization
```

### Features

- **Fetches all teams** in the organization with full details
- **Direct member detection** - Excludes inherited members from parent teams
- **Member role tracking** - Captures maintainer vs member roles
- **Repository permissions** - Exports what repos each team can access and permission level
- **Team hierarchy** - Captures parent-child team relationships
- **Pagination handling** - Automatically handles large organizations
- **Rate limit management** - Automatic rate limit detection, waiting, and retry logic
- **API call estimation** - Estimates total API calls needed before execution
- **Comprehensive logging** - Detailed logs to both console and `team_fetch.log`

### Member Detection Logic

**Important:** This script only exports **DIRECT** members of each team, not inherited members.

- For teams with parent teams, inherited members are **excluded**
- Only users explicitly added to that specific team are included
- Uses GitHub's team membership API to verify direct membership
- Prevents duplicate member listings across team hierarchies

### Output Format

CSV file: `github_teams.csv`

| Column | Description |
|--------|-------------|
| team_name | Team display name |
| team_slug | URL-friendly team identifier |
| team_description | Team description text |
| team_privacy | `closed` (visible) or `secret` (hidden) |
| parent_team | Parent team name (empty if top-level) |
| member | GitHub username of team member |
| member_role | `maintainer` or `member` |
| repo_name | Repository name the team has access to |
| repo_permission | Permission level: `pull`, `push`, `maintain`, `triage`, `admin` |

**Note:** Each row represents a unique combination of team-member-repository. A team with 3 members and 2 repos will have 6 rows (or fewer if repos are empty).

### Example CSV Output

```csv
team_name,team_slug,team_description,team_privacy,parent_team,member,member_role,repo_name,repo_permission
Engineering,engineering,Core engineering team,closed,,john_doe,maintainer,backend-api,push
Engineering,engineering,Core engineering team,closed,,jane_smith,member,backend-api,push
DevOps,devops,DevOps team,closed,Engineering,bob_jones,maintainer,infrastructure,admin
Security,security,Security team,secret,,alice_brown,maintainer,security-tools,admin
```

### Token Requirements

**Required scopes for `GH_PAT`:**
- `read:org` (read organization teams, members, and data)
- `repo` (read repository information and permissions)

### Performance Considerations

For large organizations, this script can make **thousands** of API calls:
- Approximately: `(number_of_teams × number_of_org_members)` membership checks
- Plus repository and team info fetches
- Example: 50 teams × 100 members = ~5,000 API calls

**Recommendations:**
- Use `--estimate-only` first to see expected API calls
- Increase `--rate-limit-delay` to be more conservative (default: 0.5 seconds)
- GitHub rate limit: 5,000 requests/hour for authenticated users
- For very large orgs, consider using GitHub App tokens (15,000/hour)

## Script 2: Recreate Teams (`team-recreation.py`)

### Purpose
Recreates teams in a target organization from the exported CSV, maintaining team hierarchy and adding members with their roles. Does NOT assign repositories (use script 3 for that).

### Usage

1. First, export teams using `get-teams.py` to generate `github_teams.csv`
2. (Optional) Add an `emu_members` column if migrating to EMU organization
3. Run the recreation script:

```bash
python team-recreation.py
```

**With options:**
```bash
# Custom rate limit delay
python team-recreation.py --rate-limit-delay 1.5

# Custom CSV file
python team-recreation.py --csv-file custom_teams.csv
```

### Features

- **Hierarchical team creation** - Creates parent teams before child teams
- **Duplicate detection** - Skips teams that already exist
- **EMU username mapping** - Supports mapping to Enterprise Managed Users
- **Member role preservation** - Maintains maintainer vs member roles
- **Parent-child relationships** - Correctly links child teams to parents
- **Automatic retries** - Handles transient API failures
- **Rate limit handling** - Automatic waiting when rate limit is reached
- **Comprehensive logging** - Logs to both console and `team_recreation.log`

### CSV Format for Recreation

The script expects the same format as exported by `get-teams.py`, with an optional additional column:

**Required columns:**
- `team_name`, `team_slug`, `team_description`, `team_privacy`, `parent_team`, `member`, `member_role`

**Optional column for EMU:**
- `emu_members` - EMU username to use instead of `member` column

### EMU Username Mapping

For Enterprise Managed Users (EMU) migrations:

1. Add an `emu_members` column to the CSV
2. Map classic usernames to EMU usernames:

```csv
team_name,team_slug,team_description,team_privacy,parent_team,member,member_role,emu_members
Engineering,engineering,Core team,closed,,john_doe,maintainer,john_doe_mycompany
Engineering,engineering,Core team,closed,,jane_smith,member,jane_smith_mycompany
```

**Behavior:**
- If `emu_members` is present and not empty, uses that username
- Otherwise uses the `member` column username
- Empty `emu_members` values will skip that member

### Team Creation Order

1. **First pass:** Creates all parent (top-level) teams
2. **Second pass:** Creates all child teams with proper parent linkage
3. **Member addition:** Adds members to teams as they're created

### Token Requirements

**Required scopes for `TARGET_GH_PAT`:**
- `admin:org` (full organization access to create teams and manage members)

### Important Notes

**What this script does:**
- ✅ Creates teams with correct names, descriptions, privacy settings
- ✅ Establishes parent-child team relationships
- ✅ Adds members with correct roles (maintainer/member)
- ✅ Skips teams that already exist

**What this script does NOT do:**
- ❌ Does NOT assign repositories to teams (use `adding-repo-to-team.py` for that)
- ❌ Does NOT validate that members exist in the organization (will fail if not)
- ❌ Does NOT create users (invite users to organization first)

## Script 3: Assign Repositories to Teams (`adding-repo-to-team.py`)

### Purpose
Assigns repositories to teams with specific permissions from a CSV file. Handles permission consolidation when multiple permissions are specified for the same team-repo pair.

### Usage

1. Ensure teams exist in the target organization (use `team-recreation.py` if needed)
2. Prepare `github_teams.csv` with team and repository assignments
3. Run the script:

```bash
# Dry run first (recommended)
python adding-repo-to-team.py --dry-run

# Execute assignments
python adding-repo-to-team.py
```

**With options:**
```bash
# Dry run without making changes
python adding-repo-to-team.py --dry-run

# Custom CSV file
python adding-repo-to-team.py --csv-file custom_assignments.csv

# Custom rate limit delay
python adding-repo-to-team.py --rate-limit-delay 1.5

# Estimate API calls only
python adding-repo-to-team.py --estimate-only
```

### Features

- **Dry run mode** - Test assignments without making changes
- **Permission consolidation** - Automatically uses highest permission when duplicates exist
- **Validation checks** - Verifies teams and repositories exist before assignment
- **Hierarchical processing** - Processes parent teams before child teams
- **Duplicate prevention** - Skips repositories already assigned to teams
- **Rate limit handling** - Automatic rate limit detection and waiting
- **API call estimation** - Estimates total API calls before execution
- **Comprehensive reporting** - Shows success, failed, and skipped assignments
- **Detailed logging** - Logs all operations to console

### CSV Format for Repository Assignment

Uses the same CSV format as `get-teams.py` output:

```csv
team_name,team_slug,team_description,team_privacy,parent_team,member,member_role,repo_name,repo_permission
Engineering,engineering,Core team,closed,,john_doe,maintainer,backend-api,push
Engineering,engineering,Core team,closed,,jane_smith,member,frontend-app,push
DevOps,devops,DevOps team,closed,Engineering,bob_jones,maintainer,infrastructure,admin
```

**Columns used for repository assignment:**
- `team_slug` - Team to assign repository to
- `repo_name` - Repository to assign
- `repo_permission` - Permission level for the team

**Other columns are ignored** by this script but must be present for CSV validity.

### Permission Levels

Supported GitHub repository permissions (in order of privilege):

1. **pull** (read) - Clone and pull
2. **triage** - Read + manage issues/PRs (no code changes)
3. **push** (write) - Read + Triage + push commits
4. **maintain** - Push + manage repository settings (no destructive actions)
5. **admin** - Full administrative access

### Permission Consolidation

If the CSV has multiple entries for the same team-repo pair with different permissions, the script automatically uses the **highest** permission level:

**Example:**
```csv
team_name,team_slug,...,repo_name,repo_permission
Engineering,engineering,...,backend-api,pull
Engineering,engineering,...,backend-api,push
Engineering,engineering,...,backend-api,admin
```

**Result:** Team `engineering` gets `admin` permission to `backend-api` (highest)

### Processing Order

1. **Read CSV** and parse all team-repo assignments
2. **Consolidate permissions** for duplicate team-repo pairs
3. **Sort teams** - Parent teams processed before child teams
4. **Validate teams** - Check each team exists in the organization
5. **Validate repositories** - Check each repository exists
6. **Assign repositories** - Add repository to team with permission

### Token Requirements

**Required scopes for `TARGET_GH_PAT`:**
- `admin:org` (manage team repository assignments)
- `repo` (access to repository information)

### Dry Run Workflow

**Always test with dry run first:**

```bash
# 1. Test assignments without making changes
python adding-repo-to-team.py --dry-run

# 2. Review the output - look for any errors or warnings

# 3. If everything looks good, run for real
python adding-repo-to-team.py
```

Dry run output:
```
[DRY] backend-api -> engineering (push)
[DRY] frontend-app -> engineering (push)
[DRY] infrastructure -> devops (admin)
Success: 3 | Fail: 0 | Skipped: 0 | Total: 3
```

## Complete Workflow Example

### Scenario: Migrate teams from source to target organization

```bash
# ===== STEP 1: Export teams from source organization =====
# Set up .env with source credentials
# GH_PAT=ghp_source_token
# GH_ORG=source-company

# Estimate API calls first
python get-teams.py --estimate-only

# Export teams (may take several minutes for large orgs)
python get-teams.py

# Result: github_teams.csv created with all team details

# ===== STEP 2: Prepare CSV for target organization =====
# If migrating to EMU, add emu_members column and map usernames
# Otherwise, verify that all members exist in target organization

# ===== STEP 3: Recreate teams in target organization =====
# Update .env with target credentials
# TARGET_GH_PAT=ghp_target_token
# TARGET_GH_ORG=target-company

# Create teams and add members
python team-recreation.py

# Check team_recreation.log for any errors
# Verify teams were created in GitHub UI

# ===== STEP 4: Assign repositories to teams =====
# Dry run first to validate
python adding-repo-to-team.py --dry-run

# If dry run looks good, execute for real
python adding-repo-to-team.py

# Verify repository assignments in GitHub UI
```

### Expected Timeline

For an organization with:
- 50 teams
- 100 members
- 200 repositories

**Estimated times:**
- Export (get-teams.py): 30-60 minutes
- Recreation (team-recreation.py): 5-10 minutes
- Repository assignment (adding-repo-to-team.py): 10-15 minutes

**Total:** ~45-85 minutes

Adjust `--rate-limit-delay` to trade off between speed and rate limit safety.

## Troubleshooting

### "TOKEN environment variable is required"
- Ensure `.env` file exists in the script directory
- Check `GH_PAT` (for export) or `TARGET_GH_PAT` (for recreation/assignment) is set
- Verify token is not expired

### "403 Forbidden" errors
- Token lacks required scopes (needs `admin:org` for creation/assignment)
- Token doesn't have access to organization
- Organization may have SAML SSO requiring token authorization
- For EMU orgs, ensure you're using an EMU user token

### "404 Not Found" errors
- Organization name is incorrect in .env
- Team slug doesn't exist in organization
- Repository doesn't exist in organization
- Token doesn't have access to private resources

### "422 Unprocessable Entity" when creating teams
- Team name already exists (script should detect this, but API may report it)
- Invalid team name (contains special characters not allowed)
- Parent team doesn't exist or invalid parent team ID

### "Failed to add member to team"
- Member doesn't exist in the organization (invite them first)
- Member username is incorrect or typo in CSV
- For EMU orgs, ensure EMU username format is correct (e.g., `username_company`)
- Token lacks `admin:org` scope

### "Rate limit exceeded" errors
- Script should automatically wait and retry
- Increase `--rate-limit-delay` for more conservative pacing
- Check rate limit status: script logs remaining requests
- Consider using GitHub App tokens for higher limits (15,000/hour)

### "CSV file not found"
- Ensure you run `get-teams.py` first to generate the CSV
- Check the CSV filename matches `INPUT_CSV_FILE` in .env
- Verify you're running the script from the correct directory

### Parent team not found during recreation
- CSV may list child team before parent team (script handles this)
- Parent team name is incorrect or typo
- Parent team wasn't exported or was manually removed from CSV

### Repository assignment says "already in team"
- Repository is already assigned to the team (this is normal)
- Script treats this as a skip, not an error
- Permission may still be different - script doesn't update existing permissions

### EMU username mapping issues
- Ensure `emu_members` column has correct EMU format: `username_company`
- Verify EMU users exist in the target organization
- Check for typos in username mappings
- Leave `emu_members` empty to use classic `member` username

### Very slow execution
- Large organizations make many API calls (especially get-teams.py)
- Use `--estimate-only` to see expected duration
- Increase `--rate-limit-delay` if hitting rate limits frequently
- Consider running during off-peak hours
- Check internet connection stability

## Rate Limiting Best Practices

GitHub API rate limits:
- **Standard authenticated:** 5,000 requests/hour
- **GitHub App:** 15,000 requests/hour

**Recommendations:**
1. **Always estimate first** - Use `--estimate-only` to see expected API calls
2. **Start with dry run** - Test assignments with `--dry-run` before executing
3. **Adjust delay** - Increase `--rate-limit-delay` if you have time or hitting limits
4. **Monitor logs** - Scripts warn when rate limit is low (<100 remaining)
5. **Check status** - Scripts display rate limit before and after execution
6. **Batch operations** - Process smaller groups if you have many teams/repos

## Security Best Practices

1. **Token Security**
   - Never commit `.env` files to git
   - Use fine-grained personal access tokens when possible
   - Rotate tokens regularly, especially after migrations
   - Use separate tokens for source and target organizations
   - Revoke tokens immediately after migration is complete

2. **Access Control**
   - Use least privilege - only grant scopes that are needed
   - For read-only exports, use `read:org` + `repo` only
   - For creation, use `admin:org` but be cautious
   - Audit token usage in GitHub settings

3. **Data Protection**
   - Export CSVs may contain sensitive team information
   - Store CSV files securely
   - Delete temporary CSV files after migration
   - Don't share CSV files publicly (may expose team structure)

4. **Validation**
   - Always use `--dry-run` first
   - Review CSV files before recreation
   - Verify teams in GitHub UI after creation
   - Test with a few teams before bulk operations
   - Check logs carefully for errors

5. **Backup**
   - Keep exported CSV as backup of team structure
   - Document any manual changes made to teams
   - Export teams from target org after migration for verification

## Important Notes

### Direct Members Only

`get-teams.py` exports **only direct members** of each team:
- Members explicitly added to that team
- Does NOT include members inherited from parent teams
- This prevents duplicate member listings
- Ensures accurate recreation of team membership

### Repository Assignment Separation

**Design decision:** Team recreation and repository assignment are separate:
- `team-recreation.py` - Creates teams and adds members
- `adding-repo-to-team.py` - Assigns repositories to teams

**Why separate?**
- Allows creating teams before repositories exist
- Can assign repos in batches or incrementally
- Easier to troubleshoot and re-run if needed
- Follows single responsibility principle

### Team Hierarchy

Teams must be created in correct order:
1. Parent teams first
2. Child teams second (with parent references)

Scripts handle this automatically by:
- Detecting parent teams vs child teams
- Processing in two passes
- Linking child teams to already-created parent teams

### Existing Teams

All scripts check if teams already exist:
- **team-recreation.py**: Skips creation, but still adds members
- **adding-repo-to-team.py**: Attempts assignment (may skip if already assigned)

This allows re-running scripts safely without duplicating teams.

### CSV Completeness

The CSV from `get-teams.py` may have many rows:
- One row per team-member-repository combination
- Team with 5 members and 10 repos = 50 rows
- Large organizations can produce CSVs with thousands of rows
- This is normal and expected

## Use Cases

1. **Organization Migration** - Move teams when migrating to new GitHub organization
2. **EMU Adoption** - Recreate teams during Enterprise Managed Users transition
3. **Disaster Recovery** - Backup and restore team structures
4. **Team Standardization** - Export from one org, modify, import to multiple orgs
5. **Compliance Auditing** - Export team memberships for security audits
6. **Team Restructuring** - Export, modify in CSV, reimport with new structure
7. **Multi-Organization Setup** - Apply same team structure across multiple orgs
8. **Documentation** - Generate reports of who has access to what

## Related Scripts

- **`../Rulesets/`** - Requires teams to exist before migrating rulesets with team bypass actors
- **`../Repo Permissions/`** - Direct collaborator permissions (different from team access)
- **`../Migration/`** - Migrate repositories themselves before assigning to teams
- **`../Add users to org with role/`** - Invite users to organization before adding to teams
- **`../Fetch_GitHub_Org_Users_roles/`** - Export organization members and roles

## Limitations

- **Team synchronization** - Does not support LDAP/AD team sync
- **Team discussions** - Does not export or recreate team discussions
- **Team avatars** - Does not preserve team profile images
- **Nested permissions** - Exports flat permission structure, not inherited permissions
- **Code review assignments** - Does not export code review assignment settings
- **Team notifications** - Does not preserve team notification preferences
- **Organization-wide teams** - Handles organization-level teams normally

## Alternative Approaches

If these scripts don't meet your needs:

1. **GitHub Terraform Provider** - Infrastructure as code approach with state management
2. **GitHub CLI + jq** - Custom scripting with gh cli and JSON processing
3. **GitHub GraphQL API** - For more complex queries and relationships
4. **Manual Configuration** - For very small numbers of teams
5. **Third-party migration tools** - Specialized GitHub migration platforms

## Summary

| Script | Purpose | Input | Output | Token Required |
|--------|---------|-------|--------|----------------|
| **get-teams.py** | Export teams | Organization access | github_teams.csv | GH_PAT (read:org, repo) |
| **team-recreation.py** | Create teams & members | github_teams.csv | Teams in target org | TARGET_GH_PAT (admin:org) |
| **adding-repo-to-team.py** | Assign repos to teams | github_teams.csv | Repo assignments | TARGET_GH_PAT (admin:org, repo) |

**Typical workflow:**
1. Export from source org → `get-teams.py`
2. Create teams in target org → `team-recreation.py`
3. Assign repos to teams → `adding-repo-to-team.py`

---

**Note:** These scripts are designed for repository-level teams. Organization-level access and role assignments are handled by other scripts in the `../Add users to org with role/` folder.
