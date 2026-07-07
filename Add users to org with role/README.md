# Add Users to GitHub Organization

This script automates adding users to a GitHub organization with specified roles from a CSV file.

## Prerequisites

- Python 3.x
- Required packages: `requests`, `python-dotenv`

Install dependencies:
```bash
pip install requests python-dotenv
```

## Setup

1. Create a `.env` file with the following variables:
```
GH_PAT=your_github_personal_access_token
GH_ORG=your_organization_name
```

2. Create a CSV file named `users.csv` with the following format:
```csv
username,role
user-1,admin
user-2,member
user-3,member
```

## CSV Format

| Column | Description |
|--------|-------------|
| `username` | GitHub username to add |
| `role` | Role to assign: `member` or `admin` (defaults to `member` if not specified) |

## Usage

Run the script:
```bash
python add_users_org.py
```

## Features

- ✅ Sends organization invitations to users
- ✅ Assigns member or admin roles
- ✅ Skips users already in the organization
- ✅ Handles rate limiting automatically
- ✅ Validates token permissions before execution
- ✅ Generates detailed output CSV with results
- ✅ Pagination support for large organizations

## Output

The script generates an `output.csv` file with the following columns:
- `username` - GitHub username
- `role` - Assigned role
- `status` - `success`, `failed`, or `skipped`
- `message` - Detailed status message

## Token Permissions

The GitHub Personal Access Token must have:
- `admin:org` scope - Required to invite users and manage organization membership
- Organization admin access - You must be an admin of the organization

## Important Notes

- Users receive an invitation to join the organization
- Invited users must accept the invitation to become active members
- Script skips users who are already organization members
- Validates usernames exist on GitHub before sending invitations
- Includes automatic rate limit handling and delays between requests

## Example Output

```
[*] Checking token permissions...
[*] Getting existing org members...
[*] Processing user: user-1
[+] Invitation sent to user as admin
[*] Processing user: user-2
[+] Invitation sent to user as member
[*] Processing user: existing-user
[-] Already a member of my-org

[*] Summary:
    Successfully added: 2
    Failed: 0
[*] Results have been written to output.csv
```

## Troubleshooting

### Permission Errors
- Ensure your PAT has `admin:org` scope
- Verify you have admin access to the organization
- Check that the token hasn't expired

### User Not Found
- Verify the username is spelled correctly
- Ensure the user has a GitHub account
- Check that the username is active (not deleted)

### Rate Limiting
- Script automatically handles rate limits
- Adds 1-second delay between requests
- Will pause and wait when limits are reached

## Role Types

| Role | Permissions |
|------|-------------|
| `member` | Standard organization member with basic access |
| `admin` | Organization administrator with full management access |
