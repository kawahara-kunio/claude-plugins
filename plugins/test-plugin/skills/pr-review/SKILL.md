---
name: pr-review
description: |
  Comprehensive multi-perspective code review for Pull Requests. Launches parallel
  subagent reviews covering security, performance, code quality, requirements alignment,
  and language/framework best practices (Kotlin/Spring, TypeScript/Next.js).

  Use when: (1) reviewing a PR for potential issues before merge, (2) performing thorough
  code review across multiple quality dimensions, (3) checking PR changes against linked
  requirements/issues, (4) getting framework-specific review feedback for Kotlin/Spring
  or TypeScript/Next.js projects.

  Accepts PR URLs (e.g., https://github.com/org/repo/pull/123).
allowed-tools: Bash, Read, Write, Grep, Glob, Task, TodoWrite, AskUserQuestion
---

# PR Review

Comprehensive multi-perspective code review. Gathers PR data and linked issue requirements, then launches 7 parallel subagent reviews covering security, performance, redundancy, requirements alignment, error handling, maintainability, and framework best practices.

## Quick Start

```
# User provides a PR URL
https://github.com/org/repo/pull/123
```

**Workflow overview:**
1. Gather PR data and linked issues
2. Ask reviewer for additional context
3. Detect tech stack
4. Launch 7 parallel review subagents
5. Merge results into final report

## Workflow

### Step 1: Input Collection and PR Data Gathering

#### 1.1 Parse PR URL

Extract OWNER, REPO, PR_NUMBER from the provided URL.

Supported format: `https://github.com/OWNER/REPO/pull/NUMBER`

If only a PR number is provided, ask for the repository.

#### 1.2 Fetch PR Metadata

```bash
gh pr view <PR_NUMBER> --repo <OWNER/REPO> \
  --json number,title,body,headRefName,baseRefName,state,url,labels,author,files,additions,deletions
```

#### 1.3 Fetch PR Diff

```bash
gh pr diff <PR_NUMBER> --repo <OWNER/REPO>
```

#### 1.4 Extract Linked Issues

Parse PR body for issue references:
- Patterns: `closes #N`, `fixes #N`, `resolves #N` (case-insensitive)
- Issue URLs: `https://github.com/OWNER/REPO/issues/N`

For each linked issue:
```bash
gh issue view <ISSUE_NUMBER> --repo <OWNER/REPO> --json number,title,body,labels,state
```

#### 1.5 Ask Reviewer for Additional Context

Use AskUserQuestion to ask:

**Question**: "PR情報を取得しました。レビューに役立つ追加情報はありますか？"

Options:
1. "追加情報なし（このまま進める）"
2. "要件・設計ドキュメントを参照してほしい" → パスを聞いて Read で読み込み
3. "特に注目してほしい点がある" → 詳細を聞く

### Step 2: Technology Detection

Detect tech stack from PR file list and diff content.

**Kotlin/Spring detection** (2+ signals で確定):
- File extensions: `.kt`, `.kts`
- Build files: `build.gradle.kts`, `build.gradle`
- Config: `application.yml`, `application.properties`
- Imports: `org.springframework`, `kotlinx.coroutines`
- Annotations: `@SpringBootApplication`, `@RestController`, `@Service`

**TypeScript/Next.js detection** (2+ signals で確定):
- File extensions: `.ts`, `.tsx`
- Config: `next.config.*`, `tsconfig.json`
- Imports: `from 'next/`, `from 'react'`
- File paths: `app/`, `pages/`, `components/`
- Special files: `middleware.ts`, `layout.tsx`, `page.tsx`

If detected → Load corresponding reference file:
- Kotlin/Spring → `references/kotlin-spring-checklist.md`
- TypeScript/Next.js → `references/typescript-nextjs-checklist.md`

If neither detected → Skip framework-specific review (Step 4 task #5), note in report.

### Step 3: Prepare Context Files

Save shared data for subagents:

**File 1**: `.tmp/{REPO}-pr-{NUM}-context.md`
```markdown
# PR Context

## PR Information
- Number: #{NUM}
- Title: {TITLE}
- Author: {AUTHOR}
- Branch: {HEAD} → {BASE}
- URL: {URL}
- Files changed: {COUNT} (+{ADDITIONS}/-{DELETIONS})

## PR Description
{PR BODY}

## Linked Issues
### Issue #{N}: {TITLE}
{ISSUE BODY}

## Additional Context
{User-provided context or "None"}

## Detected Tech Stack
{Kotlin/Spring | TypeScript/Next.js | Both | None detected}

## Changed Files
{File list with additions/deletions per file}
```

**File 2**: `.tmp/{REPO}-pr-{NUM}-diff.txt`
Full PR diff output.

### Step 4: Launch Parallel Reviews

Launch **7 subagent tasks in parallel** using the Task tool with `subagent_type: "general-purpose"`.

Each subagent is **READ-ONLY**, **INDEPENDENT**, and **NON-DESTRUCTIVE**.

**Parallelize this step AS MUCH AS POSSIBLE** — launch all 7 tasks in a single message.

Each subagent receives a prompt containing:
1. Instruction to read `.tmp/{REPO}-pr-{NUM}-context.md` and `.tmp/{REPO}-pr-{NUM}-diff.txt`
2. The specific review perspective and criteria (from `references/review-perspectives.md`)
3. If framework-specific: the relevant checklist content
4. Instruction to write findings to its designated output file

#### Subagent 1: Security Review
- Output: `.tmp/{REPO}-pr-{NUM}-security.md`
- Focus: Injection, auth/authz, secrets exposure, XSS/CSRF, input validation
- See `references/review-perspectives.md` § Security Review for full criteria

#### Subagent 2: Redundancy Review
- Output: `.tmp/{REPO}-pr-{NUM}-redundancy.md`
- Focus: Dead code, duplicated logic, over-engineering, unused imports
- See `references/review-perspectives.md` § Redundancy Review for full criteria

#### Subagent 3: Performance Review
- Output: `.tmp/{REPO}-pr-{NUM}-performance.md`
- Focus: N+1 queries, blocking ops, missing pagination, inefficient algorithms
- See `references/review-perspectives.md` § Performance Review for full criteria

#### Subagent 4: Requirements Alignment Review
- Output: `.tmp/{REPO}-pr-{NUM}-requirements.md`
- Focus: Issue requirements match, acceptance criteria, scope creep, missing edge cases
- See `references/review-perspectives.md` § Requirements Alignment for full criteria

#### Subagent 5: Framework Best Practices Review (conditional)
- Output: `.tmp/{REPO}-pr-{NUM}-best-practices.md`
- **Only launch if tech stack detected in Step 2**
- Include the relevant checklist content in the subagent prompt:
  - Kotlin/Spring → content from `references/kotlin-spring-checklist.md`
  - TypeScript/Next.js → content from `references/typescript-nextjs-checklist.md`
  - Both → include both checklists
- See `references/review-perspectives.md` § Framework Best Practices for full criteria

#### Subagent 6: Error Handling & Resilience Review
- Output: `.tmp/{REPO}-pr-{NUM}-error-handling.md`
- Focus: Exception handling, logging, retry logic, graceful degradation
- See `references/review-perspectives.md` § Error Handling for full criteria

#### Subagent 7: Maintainability & Design Review
- Output: `.tmp/{REPO}-pr-{NUM}-maintainability.md`
- Focus: Naming conventions, SRP, coupling, test implications, backward compatibility
- See `references/review-perspectives.md` § Maintainability for full criteria

#### Subagent Output Format

All subagents MUST use this format for each finding:

```markdown
## Finding: {Title}
- **Severity**: CRITICAL | WARNING | INFO | SUGGESTION
- **File**: `path/to/file:L42-L55`
- **Issue**: {Description of the problem}
- **Code**:
  ```{language}
  {relevant code snippet from diff}
  ```
- **Recommendation**: {What should change and why}
```

If no issues found in a perspective, write:
```markdown
## No Issues Found
This perspective review found no concerns in this PR.
```

### Step 5: Merge Results and Generate Report

1. Read all subagent output files from `.tmp/`
2. Deduplicate: Merge findings that reference the same file and line range
   - Combine perspective attributions (e.g., "Security, Performance")
   - Keep the highest severity level
3. Sort by severity: CRITICAL → WARNING → INFO → SUGGESTION
4. Generate final report following `references/output-template.md`
5. Determine overall assessment:
   - **REQUEST_CHANGES**: Any CRITICAL findings exist
   - **COMMENT**: WARNING findings exist but no CRITICAL
   - **APPROVE**: Only INFO/SUGGESTION findings or no issues
6. Write final report to `.tmp/{REPO}-pr-{NUM}-review.md`
7. Delete intermediate files: context, diff, and individual perspective files
8. Display the report path and executive summary to the user

## Subagent Prompt Template

When launching each subagent, construct the prompt as follows:

```
You are reviewing a Pull Request from the perspective of {PERSPECTIVE_NAME}.

Read the following files:
- .tmp/{REPO}-pr-{NUM}-context.md (PR metadata, linked issues, tech stack)
- .tmp/{REPO}-pr-{NUM}-diff.txt (complete PR diff)

{If framework-specific: Also use these review criteria:
{CHECKLIST_CONTENT}}

Review the PR diff ONLY from the {PERSPECTIVE_NAME} perspective.
Focus areas:
{CRITERIA_LIST from review-perspectives.md}

For each issue found, write a finding using this exact format:

## Finding: {Title}
- **Severity**: CRITICAL | WARNING | INFO | SUGGESTION
- **File**: `path/to/file:L42-L55`
- **Issue**: {Description}
- **Code**:
  ```{language}
  {code snippet}
  ```
- **Recommendation**: {What to change}

If no issues are found, write:
## No Issues Found
This perspective review found no concerns in this PR.

Write all findings to: .tmp/{REPO}-pr-{NUM}-{PERSPECTIVE_SLUG}.md
```

## Important Rules

1. **Never modify the PR source code** — this skill is read-only analysis
2. **Always ask for additional context** before starting reviews
3. **Launch subagents in parallel** — use a single message with all 7 Task tool calls
4. **Include the review criteria in each subagent prompt** — subagents cannot read reference files, so embed the relevant criteria directly
5. **Clean up intermediate files** — only the final `-review.md` should remain
6. **Use Japanese for the final report** — follow output-template.md conventions
7. **Merge all results into a single file** — 全サブエージェントの結果を `.tmp/{REPO}-pr-{NUM}-review.md` に統合し、最終レポートは必ず1ファイルにまとめる

## References

Load these as needed during execution:

- `references/review-perspectives.md`: Detailed criteria for all 7 review perspectives — **Read this before constructing subagent prompts**
- `references/kotlin-spring-checklist.md`: Kotlin/Spring specific checklist — Load if Kotlin/Spring detected
- `references/typescript-nextjs-checklist.md`: TypeScript/Next.js specific checklist — Load if TypeScript/Next.js detected
- `references/output-template.md`: Final report template — **Read this before Step 5**
