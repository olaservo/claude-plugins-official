---
description: Add a co-author trailer to a commit, giving credit to a GitHub collaborator
argument-hint: <github-username> [commit-message]
allowed-tools: Bash(git config:*), Bash(git diff:*), Bash(git log:*), Bash(git commit:*), Bash(git remote:*), Bash(gh api:*), Bash(curl:https://api.github.com/*), Bash(curl:https://*/api/v3/users/*)
---

# Co-Author Commit

Give credit to a GitHub collaborator by adding a `Co-authored-by` trailer.

## Arguments

$ARGUMENTS

- **First argument** (required): GitHub username of the co-author (e.g., `octocat`)
- **Remaining arguments** (optional): Commit message for a new commit

## Etiquette

Before adding a co-author, keep these principles in mind:

- **Get consent** — Only credit someone who knows about and agrees to the attribution. Co-authorship implies they reviewed or endorsed the code.
- **Credit real contributions** — The person should have genuinely contributed (code, design, debugging, pair programming, review feedback, etc.). Don't inflate credit.
- **Shared responsibility** — Co-authorship ties someone's name to the code. If it has bugs or security issues, they're associated with it.
- **Never implicate** — Don't add someone to problematic or controversial code to shift blame or create false association.

If the user's request seems like it might violate these principles, raise the concern before proceeding.

## Instructions

### Step 1: Parse arguments
- Extract the GitHub username from the first argument
- Everything after the username is the commit message (if provided)

### Step 2: Validate the GitHub user
Try these methods in order to look up the user:

1. **`gh` CLI** (preferred): `gh api users/{username}`
2. **`curl` with token** (fallback): Check for a `GH_TOKEN` or `GITHUB_TOKEN` environment variable, then:
   ```
   curl -s -H "Authorization: Bearer $TOKEN" https://api.github.com/users/{username}
   ```
   For GitHub Enterprise, derive the API URL from the repo's remote: parse the hostname from `git remote get-url origin` and use `https://{hostname}/api/v3/users/{username}`
3. **`curl` unauthenticated** (fallback): `curl -s https://api.github.com/users/{username}`
4. **WebFetch profile page** (last resort, works without API access, **public GitHub users only**): Fetch `https://github.com/{username}` and parse:
   - The **numeric ID** from the avatar URL (e.g., `https://avatars.githubusercontent.com/u/583231` → ID is `583231`)
   - The **display name** from the profile page
   - If the page 404s, the user doesn't exist

If all methods fail or return a 404, tell the user the GitHub account doesn't exist (or that auth is needed) and stop.

From the response, extract their **name** (fall back to login if name is null), **numeric ID**, and **login**.
- Build the noreply email: `{id}+{login}@users.noreply.github.com`
- Show the user who they're about to credit: display name, @username

### Step 3: Verify it's not the committer themselves
- Get the current git user with `git config user.name` and `git config user.email`
- If the co-author login matches the committer's GitHub username (compare against email or name), warn the user and ask if they want to continue

### Step 4: Check for staged changes
Run `git diff --cached --quiet` to determine the mode.

### Step 5a: If there ARE staged changes → new commit
- If no commit message was provided in the arguments, ask the user for one
- Show `git diff --cached --stat` for the user to review
- Create the commit using a HEREDOC:
  ```
  git commit -m "$(cat <<'EOF'
  <commit-message>

  Co-authored-by: Name <id+login@users.noreply.github.com>
  EOF
  )"
  ```

### Step 5b: If there are NO staged changes → amend last commit
- Show the last commit with `git log -1 --format="%h %s"`
- Get the full message with `git log -1 --format=%B`
- If it already contains a properly formatted `Co-authored-by` line for this user (match on login or noreply email), tell the user and stop

#### Clean up existing co-author trailers
Before building the amended message, fix any malformed co-author lines in the existing message. Trailers must follow the git trailer format strictly:
- **No emoji prefixes** or other characters before the key (e.g., `🤖 Co-Authored-By:` → `Co-Authored-By:`)
- **All trailers must be in a single contiguous block** at the end of the message with no blank lines between them
- Scan the message for lines matching co-author patterns (e.g., lines containing `Co-Authored-By:` or `Co-authored-by:` with leading non-whitespace prefixes) and strip the prefixes
- Move all co-author trailer lines to the end as a contiguous block, separated from the body by exactly one blank line

#### Build the amended message
- Take the commit body (everything before the trailer block)
- Append all existing trailers (cleaned up) plus the new one as a contiguous block:
  ```
  git commit --amend -m "$(cat <<'EOF'
  <body>

  <existing-trailers-cleaned>
  Co-authored-by: Name <id+login@users.noreply.github.com>
  EOF
  )"
  ```

### Step 6: Verify the result
- Run `git log -1 --format='%(trailers:key=Co-authored-by)'` to extract co-author trailers
- Confirm the trailer is present and correctly formatted with the expected name and email
- Show the final commit: hash, full message, and all co-author trailers
- If the trailer is missing or malformed, warn the user that something went wrong
