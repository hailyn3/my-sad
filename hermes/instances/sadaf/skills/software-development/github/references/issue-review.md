# Reviewing an audit wave of open issues

## When to Use

Use this when asked to act as an expert reviewer for a batch of open issues
(audit wave, milestone, or triage queue) and to leave one structured comment
per issue — never to implement, label, close, or edit issue metadata.

## Selection rules

- Only touch issues with **zero comments**. A commented issue means another
  reviewer already handled it — skip it.
- Tiers: P1, then P2, then P3; within a tier, ascending issue number.
- When several agents share the same queue, **pick randomly within the
  highest available tier** to reduce collisions. Random selection does not
  override tier priority.
- One issue at a time. Do not start the next until the comment for the
  current one is posted.
- Anti-collision (mandatory, twice):
  1. Re-check the comment count immediately before deep review.
  2. Re-check it again immediately before posting.
  If it is non-zero at either point, skip the issue and move on — do not
  post a duplicate analysis.
- Done only when no uncommented issue remains in scope.

## Preflight

```bash
gh auth status
gh issue list -R OWNER/REPO --state open --limit 100 --json number,comments
```

Filter to zero-comment issues. For a specific milestone or number range,
fetch with `gh issue list --milestone ...` or `--search` and filter in Python.

## Code baseline without disturbing the working tree

Create a detached worktree at the base branch so reads/searches use clean
line numbers and the checked-out branch stays untouched:

```bash
git worktree add /tmp/REPO-base origin/beta --detach
```

Read files there with `read_file` / `search_files`. Remove the worktree when
finished.

## Per-issue workflow

1. Fresh comment count:
   `gh issue view N -R OWNER/REPO --json comments --jq '.comments|length'`
   If non-zero → skip.
2. Fetch the body to a file:
   `gh issue view N -R OWNER/REPO --json title,labels,body > /tmp/issue_N.json`
   Read it fully. Bodies may contain a ready-made plan — treat it as a claim,
   not as authority.
3. Verify every claim (file, line, behavior) against the current code in the
   base worktree. If the base branch moved past the audit commit, declare the
   drift explicitly (file and line-number shifts). If the audit commit is not
   in the local object store, say so and verify against current files only.
4. If the finding no longer holds (already fixed, duplicate, or auditor
   misread), post only section 1 (verification) with evidence and stop — do
   not write sections 2-6.
5. Otherwise draft the comment in the exact template below, naming real test
   files/classes that exist now (paths, not descriptions).
6. Re-check the comment count. If still zero, post with `--body-file`:
   `gh issue comment N -R OWNER/REPO --body-file /tmp/comment_N.md`
   Then verify the count is now 1.

## Comment template (Persian; technical terms, file names, test names in English)

```
جمع‌بندی کارشناس: <مسیر روشن | نیازمند تصمیم | قبلاً رفع شده / تکراری | وریفای نشد>
مبنای بررسی: beta@<sha> — <بدون drift / drift در فایل‌های …>

## ۱) راستی‌آزمایی یافته
<برقرار است یا نه، با ارجاع دقیق به فایل و خط و رفتار فعلی کد>

## ۲) بعد از اجرا چه چیزهایی تغییر می‌کند
<تغییر رفتار، فایل‌ها/کامپوننت‌ها/مسیرهای درگیر، اثر روی API یا UI یا داده/مایگریشن یا کانفیگ — blast radius>

## ۳) تست‌هایی که ممکن است بشکنند
<نام و مسیر تست‌های موجودی که احتمالاً قرمز می‌شوند و چرا؛ اگر هیچ‌کدام، صریح بنویس «موردی شناسایی نشد»>

## ۴) تست‌های جدید موردنیاز
<سناریوهای مشخص Feature/Unit/E2E و چه رفتاری را قفل می‌کنند>

## ۵) تصمیم‌گیری
<اگر مسیر روشن: «تصمیم لازم ندارد — مسیر روشن است» + مسیر در یک خط.
 اگر تصمیم لازم: سؤال دقیق + گزینه‌های حرفی (الف/ب/ج) + پیشنهاد خودت با دلیل.>

## ۶) طرح اجرایی پیشنهادی (خلاصه)
<گام‌های اجرا به ترتیب، کوتاه و اجرایی>
```

Sections 2-6 are required only when the finding holds.

## Red lines

- Comment only. Never change title, body, labels, or milestone; never close;
  never add `ready` or any other label; never write code or open a PR.
- Never quote secrets or credentials — for security issues, name only the
  credential type and file location.
- If code and issue text conflict, code wins — write the conflict with
  evidence in section 1.
- If you cannot verify a claim, write "وریفای نشد" and the reason.

## Pitfalls observed

- Audit commit hashes may not exist in the local object store; do not try to
  fetch them. Verify against the current base branch and declare drift.
- Line numbers drift between audit and current code; always re-read the live
  file instead of trusting the issue's line cites.
- `gh issue list --json comments` gives a count, but it is stale by the time
  you post. Always re-check immediately before posting.
- A comment appearing between your review and your post is expected in a
  multi-agent queue. Skip cleanly; do not retry or overwrite.
- Issue bodies can contain a complete implementation plan. Read it, then verify
  each claim independently; a plan is a claim, not authority.
- For "already fixed / duplicate" findings, section 1 alone is sufficient;
  do not pad with speculative sections.
