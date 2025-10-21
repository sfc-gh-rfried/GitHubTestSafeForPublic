# Snowflake Workspaces + Git Workflow Guide

**Version:** 1.0.0
**Date:** October 21, 2025
**Purpose:** Comprehensive guide for using Snowflake Workspaces with GitHub repositories

---

## Table of Contents

1. [Overview](#overview)
2. [Authentication Methods: OAuth vs. Secrets](#authentication-methods-oauth-vs-secrets)
3. [Prerequisites](#prerequisites)
4. [Authentication Setup](#authentication-setup)
5. [Creating a Workspace from Git](#creating-a-workspace-from-git)
6. [Development Workflow](#development-workflow)
7. [Git Operations in Workspaces](#git-operations-in-workspaces)
8. [Collaborative Development](#collaborative-development)
9. [Secret Management](#secret-management)
10. [Best Practices](#best-practices)
11. [Troubleshooting](#troubleshooting)
12. [Quick Reference](#quick-reference)
13. [Additional Resources](#additional-resources)

---

## Overview

Snowflake Workspaces provide an integrated development environment (IDE) within Snowsight for collaborative development with Git version control. This guide focuses on setting up and using Workspaces with GitHub repositories.

**What You'll Learn:**

- Two authentication methods (OAuth vs. Personal Access Tokens)
- Creating Workspaces from GitHub repositories
- Git workflow within Snowsight
- Collaborative development with pull requests
- Token management and security best practices

**Prerequisites:** Before starting, complete the initial setup using `snowflake_github_integration_setup_template_v2_0_0.sql`.

---

## Authentication Methods: OAuth vs. Secrets

### 🔐 Authentication Options Comparison

#### Option 1: OAuth Authentication (Recommended for github.com)

**✅ Advantages:**

- **No secrets to manage** - Snowflake handles authentication
- **Enhanced security** - Short-lived tokens that auto-refresh
- **User-level access** - Uses individual GitHub credentials
- **No token rotation** - GitHub manages token lifecycle
- **Revocable** - Easily revoke access through GitHub settings
- **SSO compatible** - Works with GitHub organizations using SAML SSO

**❌ Limitations:**

- **Only for github.com** - Doesn't work with GitHub Enterprise Server, GitLab, Bitbucket
- **Per-user authentication** - Each user authenticates individually

**When to Use:**

- Repositories hosted on github.com
- Multiple developers need access
- Organization uses GitHub SSO/SAML
- Security requirements prohibit storing tokens

#### Option 2: Personal Access Token (PAT) with Secrets

**✅ Advantages:**

- **Works everywhere** - GitHub Enterprise, GitLab, Bitbucket, AWS CodeCommit
- **Simple setup** - Just create secret and API integration
- **Shared authentication** - One token for multiple users (if needed)
- **Fine-grained permissions** - Control exactly what token can access
- **Works for automation** - Good for CI/CD pipelines

**❌ Limitations:**

- **Manual token management** - Must create and rotate tokens
- **Expiration** - Tokens expire (90 days recommended)
- **Security risk** - If compromised, needs immediate rotation

**When to Use:**

- Repositories NOT on github.com
- Automated processes/CI-CD
- Single developer or small team
- Need specific permission scopes

### 🆚 Do You Need a Secret with OAuth?

**NO!** When using OAuth authentication:

- ❌ NO secret required
- ❌ NO personal access token needed
- ❌ NO password storage
- ✅ Users authenticate directly with GitHub when creating Workspace
- ✅ Token managed automatically by Snowflake and GitHub

---

## Prerequisites

Before creating a Workspace, ensure you have:

### 1. Snowflake Setup Completed ✅

Run the setup template first: `snowflake_github_integration_setup_template_v2_0_0.sql`

This creates:

- **OAuth API Integration** (for OAuth method) OR
- **PAT API Integration + Secret** (for PAT method)
- Git repository clones (optional)

### 2. GitHub Repository ✅

- Repository URL (HTTPS format): `https://github.com/your-org/your-repo`
- For private repos: Appropriate access permissions
- For OAuth: GitHub account with repository access
- For PAT: Token with `repo` and `read:org` scopes

### 3. Snowflake Permissions ✅

- `USAGE` on API Integration
- `CREATE WORKSPACE` privilege on schema
- `READ` on Secret (only for PAT method)

---

## Authentication Setup

Choose **ONE** method based on your requirements.

### Method 1: OAuth Authentication (github.com only)

#### Create OAuth API Integration

```sql
USE ROLE ACCOUNTADMIN;

CREATE OR REPLACE API INTEGRATION github_oauth_integration
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com/')
  API_USER_AUTHENTICATION = (TYPE = SNOWFLAKE_GITHUB_APP)
  ENABLED = TRUE
  COMMENT = 'OAuth integration - no secrets required';

GRANT USAGE ON INTEGRATION github_oauth_integration TO ROLE DEVELOPER_ROLE;
```

**Key:** `API_USER_AUTHENTICATION = (TYPE = SNOWFLAKE_GITHUB_APP)` enables OAuth.

**That's it!** No secrets needed. Users authenticate when creating Workspace.

---

### Method 2: Personal Access Token with Secret

#### Step 1: Create Secret

```sql
USE ROLE ACCOUNTADMIN;

CREATE OR REPLACE SECRET GEORGIA_DCH.MANUALS_SEARCH.github_pat
  TYPE = PASSWORD
  USERNAME = 'not_used'
  PASSWORD = 'ghp_YourPersonalAccessTokenHere'
  COMMENT = 'GitHub PAT - Expires 2026-01-15';

GRANT READ ON SECRET GEORGIA_DCH.MANUALS_SEARCH.github_pat TO ROLE DEVELOPER_ROLE;
```

#### Step 2: Create PAT API Integration

```sql
CREATE OR REPLACE API INTEGRATION github_pat_integration
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com/sfc-gh-rfried/')
  ALLOWED_AUTHENTICATION_SECRETS = (GEORGIA_DCH.MANUALS_SEARCH.github_pat)
  ENABLED = TRUE
  COMMENT = 'PAT-based integration';

GRANT USAGE ON INTEGRATION github_pat_integration TO ROLE DEVELOPER_ROLE;
```

**Note:** Manual token rotation required every 90 days!

---

## Creating a Workspace from Git

### Step 1: Navigate to Workspaces

1. Log into Snowsight
2. Click **Projects** in left navigation
3. Click **Workspaces** tab
4. Click **+ Workspace** button

### Step 2: Configure Workspace

1. Choose **"Create from Git Repository"**
2. Enter details:
   - **Name:** `manuals_search_workspace` (descriptive, unique)
   - **Database:** `GEORGIA_DCH`
   - **Schema:** `MANUALS_SEARCH`

### Step 3: Connect to Git

**For OAuth Method:**

```
Repository URL: https://github.com/sfc-gh-rfried/snowflake_integration
API Integration: github_oauth_integration
```

- Click **Authenticate with GitHub**
- Log into GitHub in popup window
- Authorize Snowflake app
- Select branch: `main`

**For PAT Method:**

```
Repository URL: https://github.com/sfc-gh-rfried/snowflake_integration
API Integration: github_pat_integration
Secret: GEORGIA_DCH.MANUALS_SEARCH.github_pat
Branch: main
```

### Step 4: Create

1. Click **Create**
2. Snowflake clones repository
3. Workspace opens with repository files

---

## Development Workflow

### Workspace Interface

```
┌──────────────────────────────────────────────────┐
│ Workspace: snowflake_integration   Branch: main  │
├──────────┬───────────────────────────────────────┤
│ FILES    │ CODE EDITOR                           │
│ 📁 sql/  │ -- Your SQL code here                 │
│ 📁 python│                                       │
│ README.md│                                       │
│ Git ● 2  │                                       │
├──────────┴───────────────────────────────────────┤
│ RESULTS                                          │
└──────────────────────────────────────────────────┘
```

### Working with Files

**SQL Files:**

1. Open file from tree
2. Edit code
3. Click ▶ **Run** to execute
4. View results below
5. Auto-saves locally

**Python Files:**

1. Create/open Python file
2. Write procedures, functions, or Streamlit apps
3. Test in worksheet or deploy
4. Auto-saves locally

---

## Git Operations in Workspaces

### View Status

- **Git panel** (left sidebar) shows:
  - Modified files (marked with ●)
  - Current branch
  - Unpushed commits

### Commit Changes

**Step 1: Review**

1. Click **Git** icon
2. View modified files
3. Click file to see diff

**Step 2: Stage**

1. Hover over file
2. Click **+** to stage
3. Or **"Stage All"**

**Step 3: Commit**

1. Enter commit message:
   ```
   feat: Add manual search query

   - Implemented search functionality
   - Optimized performance
   ```
2. Click **Commit**

**Step 4: Push**

1. Click **Push** button
2. Changes uploaded to GitHub

---

## Collaborative Development

### Feature Branch Workflow

**1. Create Branch**

- Click branch dropdown
- **+ New Branch**
- Name: `feature/search-optimization`
- Base: `main`
- **Create**

**2. Develop**

- Make changes
- Commit regularly
- Push frequently

**3. Create Pull Request**

- Push all commits
- Go to GitHub
- Click **"Compare & pull request"**
- Fill in details:
  ```
  Title: Add search optimization

  Description:
  - Improved query performance 40%
  - Added indexing

  Testing:
  - Tested with 1M records
  ```
- Request reviewers
- Submit

**4. Code Review**

- Reviewer checks code
- Adds comments
- Approves or requests changes
- Developer updates in Workspace
- Commits and pushes
- PR automatically updates

**5. Merge**

- All checks pass ✅
- Approved ✅
- Click **"Merge pull request"**
- Delete feature branch

**6. Sync Main**

- Switch to `main` in Workspace
- Click **Pull**
- Latest changes downloaded

---

## Secret Management

### OAuth (No Management Needed!)

**Automatic:**

- Tokens auto-refresh
- Snowflake handles lifecycle
- Users can revoke in GitHub settings

### PAT (Manual Rotation Required)

**Every 90 days:**

**Step 1: Generate New Token**

1. Go to https://github.com/settings/tokens
2. Generate new token (classic)
3. Scopes: `repo`, `read:org`
4. Expiration: 90 days
5. Copy immediately

**Step 2: Update Secret**

```sql
USE ROLE ACCOUNTADMIN;

ALTER SECRET GEORGIA_DCH.MANUALS_SEARCH.github_pat
  SET PASSWORD = 'ghp_NewTokenHere';

COMMENT ON SECRET GEORGIA_DCH.MANUALS_SEARCH.github_pat IS
  'GitHub PAT - Rotated 2025-10-21 - Expires 2026-01-18';
```

**Step 3: Test**

1. Open Workspace
2. Pull latest changes
3. If successful, rotation complete
4. Delete old token from GitHub

---

## Best Practices

### Branch Naming

```
feature/    - New features
bugfix/     - Bug fixes
hotfix/     - Urgent fixes
refactor/   - Code refactoring
docs/       - Documentation
```

### Commit Messages

**Format:**

```
<type>: <short summary>

<detailed description>

<footer>
```

**Types:** `feat`, `fix`, `docs`, `refactor`, `test`, `perf`

**Example:**

```
feat: Add search optimization

- Parallel query execution
- Caching layer
- Reduced time from 5s to 2s

Closes #123
```

### When to Commit

- After logical unit of work complete
- Before switching tasks
- At least daily
- Before end of work session

### Testing Before Committing

1. Run all queries
2. Test with sample data
3. Check dependencies
4. Review changes (diff view)
5. Write descriptive commit message

---

## Troubleshooting

### Cannot Create Workspace

**OAuth:**

```
Error: Failed to authenticate with GitHub
```

**Solutions:**

- Check popup blocker
- Verify repository access
- Try new browser tab
- Check if org requires SSO authorization

**PAT:**

```
Error: Secret not found or not authorized
```

**Solutions:**

- Verify secret exists: `SHOW SECRETS;`
- Check READ permission on secret
- Verify token hasn't expired
- Ensure API integration allows secret

### Cannot Push Changes

```
Error: Authentication failed
```

**OAuth:**

- Re-authenticate in Workspace settings
- Check repository permissions

**PAT:**

- Verify token hasn't expired
- Check token has `repo` scope
- Rotate token if needed

### Changes Not Appearing

**Problem:** Pushed but not visible in GitHub

**Solutions:**

1. Check correct branch
2. Verify push completed
3. Refresh GitHub page
4. Check for merge conflicts

### Cannot Pull Latest

```
Error: Local changes would be overwritten
```

**Solutions:**

1. Commit or stash local changes first
2. Then pull from remote
3. Resolve any conflicts
4. Commit merged result

---

## Quick Reference

### Common Actions

| Action                     | Steps                                             |
| -------------------------- | ------------------------------------------------- |
| **Create Workspace** | Projects → Workspaces → + Workspace → From Git |
| **Switch Branch**    | Branch dropdown → Select branch                  |
| **Create Branch**    | Branch dropdown → + New Branch                   |
| **Commit**           | Git panel → Stage → Message → Commit           |
| **Push**             | Git panel → Push button                          |
| **Pull**             | Git panel → Pull button                          |
| **View Diff**        | Git panel → Click modified file                  |

### Authentication Decision

```
Are repositories on github.com?
├─ YES → Use OAuth
│         ✓ No secrets
│         ✓ No token rotation
│         ✓ Per-user access
│
└─ NO → Use PAT
          ✓ Works with any Git provider
          ✗ Manual rotation every 90 days
```

---

## Additional Resources

### Official Documentation

- [Snowflake Workspaces Overview](https://docs.snowflake.com/en/user-guide/ui-snowsight-workspaces)
- [Git Integration Setup](https://docs.snowflake.com/en/developer-guide/git/git-setting-up)
- [OAuth for GitHub](https://docs.snowflake.com/en/developer-guide/git/git-setting-up)

### Community Resources

- [Collaborative Development in Snowflake Workspaces](https://medium.com/@majaf/collaborative-development-in-snowflake-how-to-set-up-workspaces-65a4ac7cf5f2)
- [GitHub Flow Guide](https://guides.github.com/introduction/flow/)

### Related Templates

- `snowflake_github_integration_setup_template_v2_0_0.sql` - Initial setup
- Git basics workflow guide (if available)

---