---
name: repo-pii-scanner
description: >
  Scans a git repository for PII (personally identifiable information) and secrets before any
  public action. Trigger this skill PROACTIVELY — without waiting to be asked — whenever the user
  is about to: make a repo public or change visibility, create or merge a pull request, post a
  GitHub issue or comment, publish a release or release notes, push code to a public repo, or share
  code/paste/gist publicly. Also trigger when the user explicitly asks to scan for PII, sensitive
  data, credentials, or anything that shouldn't be in a public repo. If the action makes content
  visible to others, run this scanner first.
---

# Repository PII & Secrets Scanner

You are auditing a git repository before a public action to find personally identifiable information
(PII) or secrets that would be problematic if exposed.

**Two layers are always required — neither alone is enough:**
- **Current files**: what's visible right now
- **Git history**: everything ever committed lives permanently in `git log -p`, even deleted files

## Step 1: Establish the repo path

Identify the repo to scan from context. On Windows you need two forms:
- Windows path for git commands: `D:\projects\code\myrepo`
- WSL path for Docker: `/mnt/d/projects/code/myrepo`

## Step 2: Run all three scans in parallel

Launch these simultaneously — don't wait for one to finish before starting the others.

---

### Scan A — Gitleaks (catches secrets and credentials)

Gitleaks automatically scans the full git history for tokens, API keys, passwords, and credentials.
Try options in this order until one works:

```bash
# Option 1: native
gitleaks detect --source=<repo-path> -v

# Option 2: WSL + Docker
wsl bash -c "docker run --rm -v <wsl-path>:/path ghcr.io/gitleaks/gitleaks:latest detect --source=/path -v 2>&1"

# Option 3: Windows Docker
docker run --rm -v <repo-path>:/path ghcr.io/gitleaks/gitleaks:latest detect --source=/path -v
```

If none are available, note it in the report and proceed — don't block on it.

---

### Scan B — Source file PII (Explore subagent)

Spawn an Explore subagent to read every file in the repo except `.git/`. Ask it to surface any
potential PII or sensitive content, including:

- Names that look real (not placeholder names — see the safe list below)
- Email addresses
- IP addresses (any `x.x.x.x` pattern)
- Machine/computer hostnames (e.g. `DESKTOP-AB12345`)
- Windows home paths with specific names (`C:\Users\<name>\`)
- Linux home paths with specific names (`/home/<name>/`)
- API keys, tokens, passwords in plaintext
- Internal URLs or private network addresses

Tell the subagent: **report everything that could plausibly be real PII**, flagging file and line.
It should NOT evaluate whether findings are real — just report them.

---

### Scan C — Git history grep passes (run yourself)

```bash
REPO=<repo-path>

# 1. Home directory paths with specific usernames
git -C "$REPO" log --all -p | grep -iE \
  "(C:\\\\Users\\\\[A-Za-z][A-Za-z0-9_-]+|/mnt/[a-z]/Users/[A-Za-z][A-Za-z0-9_-]+|/home/[a-z][a-z0-9_-]+)"

# 2. IP addresses (skip trivial ones)
git -C "$REPO" log --all -p | grep -iE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b" \
  | grep -v "0\.0\.0\.0\|127\.0\.0\|255\.255\|go\.sum"

# 3. Email addresses (skip known-safe domains)
git -C "$REPO" log --all -p | grep -iE "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" \
  | grep -iv "noreply\|example\.com\|github\.com\|golang\.org\|anthropic\.com\|pkg\.go\|go\.dev\|users\.noreply"

# 4. Machine hostnames
git -C "$REPO" log --all -p | grep -iE "DESKTOP-[A-Z0-9]{5,}" | head -20

# 5. Commit author names and emails
git -C "$REPO" log --all --format="%an | %ae" | sort -u
```

---

## Step 3: Compile and present findings

Your role in this step is **collector and reporter, not judge**. The user decides what is and isn't
PII in their context — you surface everything so they can make that call.

The only things you may silently skip (not report at all) are the **safe placeholder list** — names
and values that are unambiguously fictional across all contexts:

> `foo`, `bar`, `baz`, `alice`, `bob`, `charlie`, `johndoe`, `john`, `jane`, `user`, `username`,
> `.local`, `example.com`, `test.com`, `localhost`, `127.0.0.1`, `0.0.0.0`, `255.255.255.255`,
> `your_token_here`, `<token>`, `<name>`, `<username>`, `YOUR_API_KEY`

**Everything else goes in the table, regardless of how innocent it looks to you.** Do not add a
"verdict" or "assessment" column to the table. Do not write phrases like "this looks safe" or
"probably just a placeholder" inline — that's the user's job. Just report it neutrally.

### Report format

```
## PII / Secrets Scan Results

| # | Category | Finding | Location | Context |
|---|---|---|---|---|
| 1 | Home path | `/home/alex/projects` | `config_test.go:12` | YAML fixture |
| 2 | Email | `dev@acme.com` | commit `a3f2b1` (diff) | added in feat commit |
| 3 | IP address | `10.0.1.5` | `README.md:30` | example command |
| 4 | Secrets (gitleaks) | 0 findings | full history | — |
| 5 | Commit authors | `user@users.noreply.github.com` | all commits | GitHub no-reply |
```

**Always end the table with this exact prompt — do not issue a verdict yourself:**

> "Please review each finding and let me know which are safe to ignore and which need fixing.
> I haven't evaluated any of them — only you know whether these are real names/paths or examples."

Do not write "SAFE", "CLEAN", or any verdict until the user has responded and confirmed.

### Verdict (only after user responds)

Once the user has explicitly classified the findings, issue a final verdict:

- **✅ SAFE** — no unresolved findings
- **⚠️ REVIEW NEEDED** — some findings still need a decision
- **🚫 NOT SAFE** — confirmed PII or live secrets found

---

## Remediation options (if needed)

| Problem | Fix |
|---|---|
| PII in current files only | Remove/replace the content, commit the change |
| PII in git history | `git filter-repo --replace-text substitutions.txt` to rewrite history; requires force push and coordination with collaborators |
| Live credentials in history | **Rotate the credential immediately first**, then clean history |
| Sensitive file ever committed | `git filter-repo --path <file> --invert-paths` removes it from all history |
