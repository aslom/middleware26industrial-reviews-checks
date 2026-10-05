# middleware26industrial-reviews-checks

Quick PDF validation for submissions to **ACM Middleware 2026 — Industrial
Track** (27th ACM International Middleware Conference, Tarragona, Spain,
14–18 Dec 2026) — plus an optional [Agent Skill](https://agentskills.io) layer on
top.

Built for the case where **one reviewer or chair has a lot of papers and a short
window**: submissions closed 28 Sep 2026, authors notified 13 Oct 2026, at least
three Industrial Track PC reviews per paper, single-blind.

## Two separable things, and you may only want the first

**1. `scripts/paper_checks.py` — a plain Python script that validates submission
PDFs against the track requirements. No AI involved at all.** No model, no API
key, no account, no network. It parses the PDFs with PyMuPDF and applies ordinary
rules and regexes: ≤6 pages, ACM SIGCONF at 9pt, "(Industry Track)" in the title,
and at least one industry author, plus page and font measurement, hidden-text detection and near-duplicate
detection. Point it at a zip of submissions and read the report.

If that is all you want, you need [section 1](#1-install-the-prerequisites) and
[section 2](#2-try-it-now-on-generated-test-pdfs) — nothing else in this README.

**2. An optional [Agent Skill](https://agentskills.io) wrapped around it**, for
when you *do* want an AI coding agent helping you review. `SKILL.md` tells the
agent how to run the checks, what the CFP requires, what it may and may not do
with a confidential submission, and when to stop and ask you.

The `check` subcommand is the no-AI part. The other two, `probe` and `zdr`, exist
only to answer "may I legally show these papers to a model at all" — ignore them
if you never will.

**Want to see it work before reading any of this?** Jump to
[section 2](#2-try-it-now-on-generated-test-pdfs) — it generates its own test
papers, so you need nothing but `uv`.

- **CFP:** <https://middleware-conf.github.io/2026/calls/call-for-industry-papers/>
- **Submission system:** HotCRP, <https://middleware26industrial.hotcrp.com/>
- **Governing policy:** [ACM Peer Review Policy](https://www.acm.org/publications/policies/peer-review) (the LLM/confidentiality clause) and [ACM Policy on Authorship, Peer Review, Readership, and Conference Publication](https://www.acm.org/publications/policies/roles-and-responsibilities) (reviewer duties, bulk-downloads)

## 1. Install the prerequisites

Two things, and only one of them is a real install:

```bash
# uv - runs the scripts and installs their dependencies for you
curl -LsSf https://astral.sh/uv/install.sh | sh

# check
uv --version          # any recent version
python3 --version     # 3.10 or newer
```

Nothing else — **no AI, no API key, no account, no network.**
`scripts/paper_checks.py check` is ordinary Python: it reads the PDFs and applies
the track requirements as rules. It runs on a laptop with the network off.

Both scripts carry a [PEP 723](https://peps.python.org/pep-0723/)
inline dependency block, so `uv run` fetches PyMuPDF into a throwaway
environment on first use. There is no `pip install`, no `requirements.txt` and no
virtualenv to activate or clean up.

**How to run anything in this repo:** `uv run <script> [args]`, from the repo
root — or just execute it, since the scripts carry a
`#!/usr/bin/env -S uv run --script` shebang:

```bash
./scripts/paper_checks.py ./papers        # equivalent to `uv run scripts/...`
```

Under a bare `python3` with nothing installed, `--version`, `--help`, `probe` and
`zdr` still work; only reading PDFs needs PyMuPDF, and that path says so plainly
instead of failing at import.

```bash
uv run scripts/paper_checks.py --version           # version, commit, tag, sha256
uv run scripts/paper_checks.py --help              # the three subcommands
uv run scripts/paper_checks.py probe --help
uv run scripts/paper_checks.py check --help
uv run scripts/paper_checks.py zdr --help
```

Network access is optional. The only commands that reach the network are
`check --hotcrp` (fetches submissions), `zdr --live` (sends a two-token probe,
never paper text), and `--version` when this copy is not a git checkout — it
then asks the GitHub API which commit `main` is at.

### Which version is this?

```bash
uv run scripts/paper_checks.py --version      # or -v
```

```
paper_checks.py 1.0.0   venue: mw26ind
  file       : .../middleware26industrial-reviews-checks/scripts/paper_checks.py
  sha256     : 566d5c87d2b4...
  git        : commit 95e039ea0e384f8fc0b413be38608237c6915dd8
  tag        : v1.0.0
  describe   : v1.0.0
  committed  : 2026-10-06
  repo       : https://github.com/aslom/middleware26industrial-reviews-checks
```

From a git checkout it reports the commit, the tag on that commit, and
`describe` (suffixed `-dirty` if you have uncommitted edits). The commit is
only reported when the surrounding repository actually tracks this file in
`HEAD`, so a copy dropped into an unrelated repo cannot claim that repo's
commit as its own. When there is no checkout — a bare copy, or `uv run` from a
raw URL — it reports the published `main` commit and tag from the GitHub API
instead, clearly labelled as `published`.

The `sha256` is the honest identifier in every mode: it pins the exact bytes
that ran, which matters most when the script was fetched from a URL rather
than checked out.

## 2. Try it now, on generated test PDFs

No real submissions needed. `make_test_pdfs.py` builds synthetic papers that
deliberately trip every check — including one carrying a hidden white-text prompt
injection — so you can see the output and judge the heuristics before trusting
them on anything that matters.

```bash
# HTTPS, no login required while the repo is public
git clone https://github.com/aslom/middleware26industrial-reviews-checks.git
cd middleware26industrial-reviews-checks

uv run scripts/make_test_pdfs.py                      # -> .tmp/testpdfs/
uv run scripts/paper_checks.py .tmp/testpdfs --out .tmp/out
```

Validating PDFs is the default action, so there is no subcommand and no venue
flag: this copy only ever checks Industrial Track submissions. Targets can be
PDFs, a papers zip, directories, or any mix:

```bash
uv run scripts/paper_checks.py middleware26industrial-paper15.pdf
uv run scripts/paper_checks.py *.pdf
uv run scripts/paper_checks.py middleware26industrial-papers.zip --out out --redact
uv run scripts/paper_checks.py a.pdf b.zip ./more
```

Expected output from the fixtures:

```
venue   : Middleware 2026 Industrial Track
source  : dir .tmp/testpdfs (4 pdf)
papers  : 4

  . paper     11  NOTE   7pp  paper11.pdf
  X paper     12  FAIL   8pp  paper12.pdf
  . paper     13  NOTE   7pp  paper13.pdf
 !! paper     14  BLOCK  7pp  paper14.pdf

wrote <repo>/.tmp/out/summary.md, summary.csv, gate.json

!! BLOCKED (hidden LLM instructions): 14
   Do not put these PDFs in front of a model. Tell the chairs.
```

Exit code `2` — the gate fired. The four fixtures are built to differ:

| Fixture | Built to be | Verdict |
|---|---|---|
| `paper11.pdf` | conforming — 9pt, `(Industry Track)` in the title, an industry affiliation, 6 technical pages | `NOTE` only |
| `paper12.pdf` | non-conforming — missing title suffix, 10pt instead of 9pt, 7 technical pages, academic-only affiliations | `FAIL` |
| `paper13.pdf` | a near-duplicate of `paper11` | flagged as an overlapping pair |
| `paper14.pdf` | hostile — hidden white-text injection plus 2pt text | `BLOCK` |

Output is detailed by default: each paper is followed by every finding behind
its verdict, then a `CHECK SUMMARY` table rolling the whole batch up per check —
worst severity, how many papers hit it, the full severity breakdown, and which
papers. Two switches:

```bash
uv run scripts/paper_checks.py ./papers -q          # one line per paper, no detail
uv run scripts/paper_checks.py ./papers --show-ok   # include the checks that passed
```

Safety findings always print their evidence, whatever the severity: `safety.hidden_text`
shows the concealed text **verbatim**, pipe-separated span by span, so you can see for
yourself whether it is a concealed message or just chart tick labels. The per-paper
files under `--out` carry the same information in full, and `papers/<id>/hidden_text.txt`
holds the untruncated extraction.

Then read what it produced:

```bash
cat .tmp/out/summary.md                  # worst-first triage, STOP list, duplicates
cat .tmp/out/papers/12/report.md         # why paper 12 failed, finding by finding
cat .tmp/out/papers/14/hidden_text.txt   # the injection itself, as evidence
column -s, -t .tmp/out/summary.csv       # paper x check matrix
```

Paper 14 is worth studying: `safety.prompt_injection` is a `BLOCK` because the
instructions are in **hidden** text, whereas the same strings in visible body text
come back only as a `NOTE` — this track publishes work on agent guardrails, so
injection strings in visible prose are usually the paper's own subject matter.

Everything lands in `.tmp/`, which is gitignored. `rm -rf .tmp` when done. These
are fixtures — never put real submissions in that directory.

## 3. What this skill is

Everything above needed no AI. This section is about the optional second
layer: a step-by-step guide an agent follows, wrapped around the same
`scripts/paper_checks.py`:

1. **Step 0 — data-retention and model posture.** Before any paper is shown to a
   model: which plan you are on, what its retention actually is, how to verify it,
   and which API surfaces to avoid. `probe` reports the credential Claude Code
   will actually use and the hygiene items still outstanding; `zdr` answers
   whether zero data retention is in effect and how strong the evidence is.
   Full detail, including the Pro/Max checklist and a local-model (Ollama) setup,
   is in [`references/RETENTION.md`](references/RETENTION.md).
2. **Steps 1–2 — ingest and triage.** PDF text extraction from a HotCRP zip
   (preferred, fully offline) or the HotCRP REST API, then a worst-first triage
   table across the whole batch.
3. **Step 3 — hidden-prompt gate.** Blocks papers carrying LLM-directed
   instructions in invisible text before they reach a model.
4. **Steps 4–8 — per-paper CFP checklist, the allowed/forbidden line for model
   use, submitting in HotCRP, batch-consistency checks, and cleanup.**

### What it does not do

It does **not** score papers, decide accept/reject, judge novelty or significance,
or write your review. That split is deliberate — see Step 5 of [`SKILL.md`](SKILL.md)
for the evidence behind it. Every check is a mechanical heuristic on PDF internals
and is reported as a pointer for a human, never a verdict.

## 4. Install the skill into your agent

The repository root **is** the skill folder, so installing is "put this directory
where your agent looks for skills, under its own name". The directory must stay
named `middleware26industrial-reviews-checks` — the Agent Skills spec requires the `name` in
`SKILL.md` to match the parent directory.

> **No login needed, as long as the repo is public.** Every URL below is HTTPS,
> so `git clone`, `curl` and the tarball/zip links all work anonymously — no SSH
> key, no token, no `gh auth login`. If the repo is **private**, all of them fail
> instead: HTTPS clone prompts for a username and the `archive/…` download links
> return 404. In that case either make it public, or authenticate once with
> `gh auth login` (which configures a git credential helper) or a
> [personal access token](https://github.com/settings/tokens) as the password.

### Fastest: clone straight into the skills directory

```bash
# Claude Code (personal, available in every project)
git clone https://github.com/aslom/middleware26industrial-reviews-checks.git \
  ~/.claude/skills/middleware26industrial-reviews-checks

# Cursor / Codex / Gemini CLI / Copilot (personal)
git clone https://github.com/aslom/middleware26industrial-reviews-checks.git \
  ~/.agents/skills/middleware26industrial-reviews-checks
```

### Install once, cover every agent

There is **no single directory that every agent reads** — Claude Code does not read
`.agents/skills/`, and Cursor, Codex and Gemini CLI do not read `.claude/skills/`
as a primary location. One copy plus a symlink covers both families:

```bash
mkdir -p ~/.agents/skills ~/.claude/skills
git clone https://github.com/aslom/middleware26industrial-reviews-checks.git \
  ~/.agents/skills/middleware26industrial-reviews-checks
ln -s ~/.agents/skills/middleware26industrial-reviews-checks \
      ~/.claude/skills/middleware26industrial-reviews-checks
```

Or let the bundled installer do it:

```bash
git clone https://github.com/aslom/middleware26industrial-reviews-checks.git
cd middleware26industrial-reviews-checks
./install.sh            # ~/.agents/skills + symlink into ~/.claude/skills
./install.sh --list     # show what it would do and which targets exist
./install.sh --project  # install into ./.claude/skills and ./.agents/skills instead
./install.sh --copy     # real copies everywhere instead of symlinks
```

### Download and copy, without git

```bash
# 1. tarball, extracted straight into place (no git, no clone dir)
mkdir -p ~/.claude/skills/middleware26industrial-reviews-checks
curl -L https://github.com/aslom/middleware26industrial-reviews-checks/archive/refs/heads/main.tar.gz \
  | tar -xz --strip-components=1 -C ~/.claude/skills/middleware26industrial-reviews-checks

# 2. zip
curl -LO https://github.com/aslom/middleware26industrial-reviews-checks/archive/refs/heads/main.zip
unzip -q main.zip && mv middleware26industrial-reviews-checks-main ~/.claude/skills/middleware26industrial-reviews-checks

# 3. GitHub CLI
gh repo clone aslom/middleware26industrial-reviews-checks ~/.claude/skills/middleware26industrial-reviews-checks

# 4. a tagged release (reproducible — pin a version)
curl -L https://github.com/aslom/middleware26industrial-reviews-checks/archive/refs/tags/v1.0.0.tar.gz \
  | tar -xz --strip-components=1 -C ~/.claude/skills/middleware26industrial-reviews-checks
```

The tarball form (1) is the best general answer: one command, no git required, no
leftover clone directory, and it lands with the correct directory name.

### Where each agent looks

Paths below were checked against each vendor's own documentation.

| Agent | Project scope | Personal scope | Docs |
|---|---|---|---|
| **Claude Code** | `.claude/skills/` (also nested dirs and `--add-dir`) | `~/.claude/skills/` | [docs](https://code.claude.com/docs/en/skills) |
| **Claude apps + API** | — (upload a zip; see below) | Settings → Capabilities → Skills | [docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) |
| **GitHub Copilot** (CLI, VS Code, JetBrains, cloud agent, code review) | `.github/skills/`, `.claude/skills/`, `.agents/skills/` | `~/.copilot/skills/`, `~/.agents/skills/` | [docs](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills) |
| **Cursor** | `.agents/skills/`, `.cursor/skills/` (legacy: `.claude/skills/`, `.codex/skills/`) | `~/.agents/skills/`, `~/.cursor/skills/` | [docs](https://cursor.com/docs/context/skills) |
| **Codex / ChatGPT** | `.agents/skills/` (cwd up to repo root) | `~/.agents/skills/`; system `/etc/codex/skills` | [docs](https://learn.chatgpt.com/docs/build-skills) |
| **Gemini CLI** | `.agents/skills/` (preferred), `.gemini/skills/` | `~/.agents/skills/`, `~/.gemini/skills/` | [docs](https://geminicli.com/docs/cli/skills/) |
| **VS Code** | via Copilot, above | via Copilot, above | [docs](https://code.visualstudio.com/docs/copilot/customization/agent-skills) |

Also implements the standard — follow each project's own install docs, then drop
this directory in the location they name:
[OpenCode](https://opencode.ai/docs/skills/) ·
[Goose](https://block.github.io/goose/docs/guides/context-engineering/using-skills/) ·
[Amp](https://ampcode.com/manual#agent-skills) ·
[OpenHands](https://docs.openhands.dev/overview/skills) ·
[Roo Code](https://docs.roocode.com/features/skills) ·
[Kiro](https://kiro.dev/docs/skills/) ·
[Factory](https://docs.factory.ai/cli/configuration/skills) ·
[Junie](https://junie.jetbrains.com/docs/agent-skills.html) ·
[Letta](https://docs.letta.com/letta-code/skills/) ·
[Augment](https://docs.augmentcode.com/cli/skills) ·
[Tabnine](https://docs.tabnine.com/main/getting-started/tabnine-cli/features/agent-skills) ·
[Firebender](https://docs.firebender.com/multi-agent/skills) ·
[Emdash](https://docs.emdash.sh/skills) ·
[Mux](https://mux.coder.com/agent-skills) ·
[OpenClaw](https://docs.openclaw.ai/tools/skills) ·
[full client list](https://agentskills.io/clients)

### Claude apps (web and desktop)

These take an uploaded zip rather than a directory:

```bash
cd .. && zip -r middleware26industrial-reviews-checks.zip middleware26industrial-reviews-checks \
  -x '.git/*' -x '*/__pycache__/*'
```

Then Settings → Capabilities → Skills → upload. Note that the Claude apps have no
shell, so `scripts/paper_checks.py` will not run there — the checklist and policy
guidance work, the automation does not. For the full workflow use a terminal agent.

### Project scope, to share with co-reviewers or the PC

```bash
mkdir -p .claude/skills .agents/skills
git submodule add https://github.com/aslom/middleware26industrial-reviews-checks.git \
  .claude/skills/middleware26industrial-reviews-checks
```

Commit it, and everyone who clones the review repo gets the same checks. Use
`.agents/skills/` instead (or as well) for Cursor, Codex and Gemini CLI.

## Verify the install

```bash
# does the agent see it?
#   Claude Code:  /skills        Cursor: /skills        Copilot CLI: /skills
# validate against the spec:
uvx --from git+https://github.com/agentskills/agentskills#subdirectory=skills-ref \
    skills-ref validate ~/.claude/skills/middleware26industrial-reviews-checks
```

Then just ask your agent in plain language, e.g.
*"run the mw26ind review checks on this zip of submissions"* — the skill is
selected from its `description`, so you don't need to name it. In Claude Code you
can also invoke it directly as `/middleware26industrial-reviews-checks`.

## What it looks like in use

Once installed, invoke the skill by name and hand it your papers. In Claude Code
that is `/middleware26industrial-reviews-checks`; in other agents, just say what
you want — the skill is selected from its `description`.

> **The transcript below is illustrative.** Paper numbers, titles, affiliations,
> figures and the wrong-venue details are invented for documentation, and no real
> submission is described. It shows the *shape* of a session: the agent runs the
> script, confirms or clears each finding by hand, and stops before judgement.

<details>
<summary><code>/middleware26industrial-reviews-checks check the two PDFs</code></summary>

```text
> /middleware26industrial-reviews-checks check the two PDFs

I'll start with Step 0 - the retention/posture gate - and locate the PDFs.

  uv run scripts/paper_checks.py probe
  uv run scripts/paper_checks.py zdr --training-off
  uv run scripts/paper_checks.py ./papers --out out/mw26ind --redact

Step 0 - settled: Case A, compliant

Claude Code on a consumer Max plan, subscription OAuth, no API key. You
confirmed Model Improvement = OFF -> no training + 30-day deletion, which
satisfies ACM's confidentiality clause. Not ZDR (never available on Max) and not
claimed as such.

  2026-10-06 - Claude MAX, subscription OAuth, no API key.
  "Help improve our AI models" = OFF, verified via /privacy-settings.
  Retention: 30 days, no training (Anthropic consumer terms).
  ZDR: not available on consumer plans; not claimed.
  Reviewer: <your name>

Local hygiene the probe still flags, if you want it: cleanupPeriodDays low
(e.g. 7), DISABLE_FEEDBACK_COMMAND=1, DISABLE_ERROR_REPORTING=1.

Step 3 - hidden-prompt gate: clear on both

gate.json -> "blocked": []. No hidden_text.txt written for either paper, i.e.
zero near-white, sub-4pt, off-page, 3 Tr or optional-content-layer text. Both
safe to put in front of a model.

Step 4 - compliance, every finding confirmed or cleared by hand

PAPER 31 - "Live Migration of Stateful Inference Workers"

Overwhelmingly conforming. One cosmetic nit, and the script's page-limit read
turned out to be a false alarm, which I cleared.

  check                   verdict
  ----------------------  --------------------------------------------------
  <=6 pages excl. refs    PASS - body and conclusion end on p6; references
                          start mid-p6 and spill to p7. No appendix. So 6
                          technical pages, not the 7 the script estimated.
  ACM SIGCONF, 9pt        PASS - acmart sigconf, 9.0pt, two-column
  US Letter               PASS - 612x792pt on all 7 pages
  single PDF              PASS
  title "(Industry        NIT, not a failure - "Industry Track" is present as
  Track)"                 its own subtitle line and in the PDF title metadata,
                          just unparenthesised and not appended. Intent to
                          comply is unambiguous. Camera-ready fix, not a
                          desk-reject.
  single-blind: names     PASS - five named authors with affiliations and
  present                 e-mails
  >=1 industry author     PASS - two industry labs alongside three universities
  no NDA                  PASS
  deployment evidence     Script counted low (2 unique markers) but the body
                          carries real production content: a table of two
                          scale-from-zero events on 8-GPU nodes, queue depth at
                          the time, p95 time-to-first-token, and measured
                          restore-to-first-token. The authors explicitly label
                          their headline percentage a projection rather than a
                          measured result - honest scoping, and your call to
                          weigh.

PAPER 44 - "Adaptive Batching for Multi-Tenant Model Serving"

Five independent, hand-confirmed format failures. Not a heuristic misfire - a
wholesale template and venue mismatch.

  check                   verdict
  ----------------------  --------------------------------------------------
  <=6 pages excl. refs    FAIL - body runs to p9, references start mid-p9.
                          9 technical pages against a limit of 6.
  ACM SIGCONF, 9pt        FAIL - 10.0pt, single column
  US Letter               FAIL - 486x720pt, an ACM journal trim size
  title "(Industry        FAIL - absent
  Track)"
  single-blind: names     FAIL - "ANONYMOUS AUTHOR(S)", running header "Anon.",
  present                 per-page line numbers. An anonymised review-mode
                          submission; this track requires names.
  >=1 industry author     UNVERIFIABLE - anonymised
  no NDA                  PASS
  deployment evidence     A trace-replay study on a public workload trace. No
                          system of their own, no deployment, no own
                          measurements. The paper itself states its metric is
                          "an opportunity proxy, not measured cold-start
                          latency" and that it makes "no latency or
                          user-experience claim". Your judgment, but the
                          track's defining requirement - real system, real
                          workload, real measurements, stated scale and
                          duration - is not met on the paper's own account of
                          itself.

The decisive evidence, worth putting in front of the co-chairs: the footer on
every page carries another ACM venue's journal proceedings line with a 2027
publication date, and reference [1] is that venue's own call for papers. This is
a journal submission for a different venue, in anonymous review mode. Either it
was uploaded to the wrong place, or it is a simultaneous submission - which the
CFP calls dishonesty or fraud. Take it to the track co-chairs with the page
numbers; do not raise it with the authors. I would frame it as a question rather
than an accusation: a wrong-file upload is both likelier and more charitable.

What I did not do

No scores, no accept/reject, no strengths or weaknesses, no novelty or
significance judgment, no review text - those are yours (Step 5). Paper 31 is
clean enough to review on the merits. Paper 44's format status is a chair
decision before any review effort goes into it.

Artifacts are in out/mw26ind/ - summary.md, summary.csv, gate.json, and per
paper report.md / checks.json / text.txt / redacted.txt. The redacted.txt files
are the ones to use for any hosted-model polish pass. rm -rf out/mw26ind when
decisions are out; there is no BLOCK evidence to preserve here.

Want me to build the claim inventory and deployment-evidence table for paper 31
(both allowed under Step 5)?
```

</details>

Four things in that transcript are the skill working as intended, not incidental:

- **Step 0 runs first and produces an attestation**, not a vague assurance, and
  explicitly declines to call a consumer plan ZDR.
- **The agent cleared a false alarm rather than passing it on.** The script
  estimated 7 technical pages for paper 31; opening the PDF showed references
  starting mid-page 6, so the real count is 6 and the paper passes. The script
  hedges on purpose — settling the hedge is the agent's job.
- **A nit is reported as a nit.** An unparenthesised subtitle is a camera-ready
  fix, not the desk-reject that `format.title_suffix` nominally implies. The CFP
  declines non-conforming papers without review, which is exactly why a human has
  to confirm before that lever is pulled.
- **The suspected dual submission goes to the chairs, not the authors**, framed as
  a question, with the page evidence attached — and the charitable explanation
  offered first.

Note also what the deployment-evidence check is *for*: on paper 31 the marker
count was low and the paper was fine; on paper 44 the paper said in its own words
that it had no measurements. The count is a prompt to read, never a verdict.


### Going deeper: claim inventory and related-work positioning

Both are explicitly on the allowed list in Step 5 — they map claims to evidence
and verify facts, without judging quality, novelty or significance.

> **Invented again.** Papers 31 and 44, and every number, figure, section
> reference and setup detail attributed to them, are fabricated for
> documentation. The uncited alternatives are described by *category* with only
> generic infrastructure named, so that nothing here resembles a particular
> submission.

<details>
<summary><code>build a claim inventory: what is claimed vs. what is actually evidenced. Are the workloads, environment and scale described well enough to judge? Watch for cherry-picked periods or unexplained baselines</code></summary>

```text
Both papers read in full, figures inspected as images, all arithmetic
recomputed.

PAPER 31 - claim inventory

  #   claim (location)                  evidence offered          gap
  --  --------------------------------  ------------------------  -------------------------
  C1  "reduces time to first token by   S4.2 p6: best of four     best case, under a tuned
      up to 58%" (abstract, concl.)     models, tuned block size  non-default config.
                                        and 16 threads            Abstract and conclusion
                                                                  state it unattributed;
                                                                  the intro does attribute
                                                                  it. None of the three
                                                                  mentions the tuning, and
                                                                  none mentions C2.
  C2  (not claimed anywhere in the      S4.1 p5, body prose only  on 2 of 4 models restore
      abstract, contributions or                                  is SLOWER than cold start,
      conclusion)                                                 and the regression is
                                                                  monotonic in model size -
                                                                  it degrades exactly where
                                                                  cold starts hurt most
  C3  "queueing delay by an estimated   Table 1 p6: two           arithmetic exact, every
      74%" (abstract, p2, concl.)       production scale-from-     cell recomputed. But n=2
                                        zero events               and the two events
                                                                  disagree widely: 81% and
                                                                  62%. The premise is the
                                                                  problem, not the sums -
                                                                  see baselines below.
  C4  startup "takes several minutes"   Fig 2 p3, eight models    supported, and consistent
      (abstract)                                                  with the prose elsewhere
  C5  "an orchestrator-level method     design prose only         never measured. S4 states
      coordinating snapshot creation,                             it deliberately uses a
      scheduling and device allocation"                           plain container runtime
      (contribution 2, p2)                                        "to isolate runtime
                                                                  performance from
                                                                  scheduling". The system
                                                                  named as the contribution
                                                                  has no numbers anywhere.
  C6  "the same snapshot can restore    none                      no multi-replica
      multiple replicas, amortizing                               experiment. Asserted three
      its cost" (abstract, S3.4,                                  times, measured zero times.
      conclusion)
  C7  "much of this traced to the       none                      causal diagnosis with no
      runtime reading the archive                                 profiling data shown.
      twice" (S4.2)                                               Load-bearing: it is the
                                                                  implied reason the
                                                                  large-model regression may
                                                                  be a harness artifact
                                                                  rather than a property of
                                                                  the method.
  C8  "being integrated into the core   footnote to a working-     status claim, not a
      APIs" (p2) / "being discussed in  group blog post           result - and the two
      the working group" (p5)                                     phrasings differ in
                                                                  strength
  C9  checkpoint latency and snapshot   Fig 5(a),(b)              supported; figure labels
      size scaling, compression cuts                              match the prose
      size 14-43% (S4.4)

C2 in full - the result that is not in the abstract

  model          startup (s)            restore (s)   outcome
  -------------  ---------------------  ------------  -------------
  4B             58.2                   24.4          -58%
  9B             61.0                   49.8          -18%
  27B            not stated (~119,      ~131          +10% slower
                 back-computed)
  35B            not stated (~148,      ~176          +19% slower
                 back-computed)

Two things compound. The motivating production data (Fig 2) is dominated by
models far larger than anything evaluated, and the largest model actually
evaluated is the one where restore is worst. Third, a presentation gap rather
than a claim gap: Fig 5(c) plots only the restore breakdown, with no startup
bars, so the paper's central comparison has no figure and the absolute startup
times for the two largest models appear nowhere - I back-computed them from the
stated percentages.

Are workloads, environment and scale described well enough?

Bench environment - yes, unusually well. OS, GPU, CPU, RAM, kernel, runtime,
checkpoint-tool and serving-engine versions, compression settings; protocol
specified with warm-up, n=5 per path, fixed seed, deterministic decoding, and
restored output verified identical to pre-checkpoint. Reproducible.

Production environment - partially. Node count, accelerator count, one week,
and the number of model configurations are given. Not stated: tenancy, request
rate, the storage backend for snapshots (the restore path is I/O-bound, so this
matters), or network and filesystem characteristics.

The gap that matters: these are two different environments and the headline
crosses between them. The 24.4 s restore comes from a single-accelerator
workstation under a plain container runtime; it is injected into production
events measured on multi-accelerator orchestrated nodes. Nothing establishes
that the transfer holds - and S4 chose that runtime precisely because it strips
out the scheduling overhead that is part of the production number.

Dispersion: n=5 is stated, but error bars appear in only one of four panels.
The three panels carrying the main result show bare bars with no variance.

Cherry-picked periods

1. The two queueing events. A 3x spread in scale-up time yields the 81% and 62%
   figures; the combined 74% is dominated by the slower event. The paper never
   says how many such events occurred in the trace week, nor how these two were
   chosen. With n=2 and no stated selection criterion this is the clearest
   exposure in the paper.
2. The slower event is an outlier against the paper's own data - slower than any
   per-model average in the motivation figure, and for the smallest model.
3. The headline model is absent from every production figure, yet it carries both
   the 58% and the queueing projection. Production data for it clearly exists.
4. Three different model sets across the paper: eight in one figure, nine in
   another, four in the evaluation. Two models appear in both motivation and
   evaluation, and two similarly-named entries are in fact different models.
5. One p95 panel looks clipped - three models sit exactly at the axis maximum,
   so the true p95 may be higher and unstated.
6. A parameter sweep is non-monotonic in the figure; the prose reports only the
   two endpoints and the uptick in the middle goes unmentioned.

Unexplained baselines

1. Cold start is the only baseline. Six related systems are cited and none is
   compared numerically. Most pointed: the predecessor this work builds on
   shares an author, and no delta against it is quantified anywhere - so the
   paper's own incremental contribution is unmeasured.
2. The same quantity appears twice with different values (58.2 s in one section,
   57.3 s in the next) with no explanation. The headline is computed against the
   smaller one; it survives either way, but one of the two is wrong.
3. The projection substitutes a scheduling-free number into a
   scheduling-inclusive measurement. The replaced interval explicitly "captures
   the complete scale-up path including scheduling, container startup, model
   loading and gateway readiness". The method does not eliminate scheduling or
   gateway readiness, yet the arithmetic credits it with doing so. The paper
   states the limitation narrowly without noting that the two intervals measure
   different things.
4. 16x decompression threads bought 0.6 s - a ~2% return on 16x parallelism is a
   notable null result and goes unremarked. It is consistent with the
   double-read diagnosis in C7, but neither is measured.

Internal inconsistencies, with anchors

  p2, contribution 1   "a seven-day trace ... over a four-week period" -
                       self-contradictory in one sentence; the figure caption
                       says one week. The four-week figure is unsupported.
  p5 vs p6             same startup quantity, two values
  p2 vs p3             a model named one way in the trace description and
                       another in the figure
  p1 abstract          "when scaling up production models" attaches to both
                       headline numbers; one of them is a single-accelerator
                       bench result, not a production measurement

Arithmetic that checks out: Table 1 is exact in every cell, including the
residual delays, the request-minute totals, both reductions, the combined row
and the derived request-hours. The S4.2 breakdown reconciles to its totals. The
problem in C3 is the premise, not the computation.

PAPER 44 - claim inventory

A different situation. I checked every number in all five tables and all of it
is exact. The claims are also unusually tightly scoped - the paper pre-empts
most of what this inventory normally catches.

  #   claim                             evidence              gap
  --  --------------------------------  --------------------  ---------------------------
  C1  41.2% proxy reduction vs the      Table 2 p6            exact (ratio verified).
      calibrated global baseline                              Scoped explicitly as a
                                                              proxy, not cold starts.
  C2  "while using 1.9% more warm       Table 2               exact. The paper states
      GB-hours"                                               plainly it is not an
                                                              equal-resource comparison.
  C3  improve / tie / worsen counts     Table 3 p7            exact; sums to the stated
                                                              active-app total, and every
                                                              quartile column sums
                                                              correctly
  C4  lowest-popularity quartile proxy  Table 3               exact, and disclosed in the
      nearly doubles                                          abstract rather than buried
  C5  the memory-aware term contributes Table 4 ablation      exact. Honest framing: most
      7.1%                                                    of the gain comes from
                                                              per-application flexibility
  C6  positive on all seven test days   Fig 2 p7              supported; the paper states
                                                              these are not independent
                                                              replicates and declines to
                                                              give a confidence interval
  C7  the cohort retains 97.4% of raw   Table 1 p3            exact; exclusions sum
      test invocations                                        exactly
  C8  "does not dominate at the         Table 2, S5.1         self-reported negative
      unconstrained endpoint"                                 result

Are workloads, environment and scale described well enough?

For what it is - a trace replay - yes, thoroughly: dataset revision and licence,
file count, bin width, the train/test split with the test period frozen,
application and invocation counts, the parameter grid, the random seed, artifact
contents with checksums, and a sensitivity run three ways.

But there is no environment, because there is no system: no deployment, no
hardware, no latency measurement, no running platform. The paper says so
repeatedly - its outcome is "an opportunity proxy, not measured cold-start
latency", and it "makes no latency, tail-latency or user-experience claim".
S6.2 lists eight numbered limitations.

So on whether the claims are supported by the data shown, this is the rare case
where the answer is yes, because the claims have been pre-shrunk to fit the data.
The live question for this track is not that; it is whether a self-declared
proxy on a public trace meets a track that exists for measurements of real-world
systems. That judgement is yours.

Cherry-picking and baselines

  - The headline operating point is pre-registered, not post-selected. The
    research plan designates it as an illustrative interior target, says it is
    not chosen for having the best test result, and the unconstrained endpoint is
    reported even though the proposed policy does not win there. That is the
    correct defence against this concern, and the losing endpoint is in fact
    reported.
  - Baselines are calibrated and plural: a global fixed policy, a tiered policy,
    a quantile policy, an ablation, and several uncalibrated fixed references.
    One of them performs catastrophically and this is reported rather than
    dropped.
  - Not budget-matched, and labelled as such - the realized fractions differ
    slightly, so the headline is a ratio at slightly unequal cost, which the
    table caption says in as many words.
  - No comparison against any prior published policy. Four are cited, none run.
    The paper states it does not claim priority, but the absence of a prior-art
    baseline is still the main evidence gap.

Two things I could not resolve from the text

1. In the final sensitivity table, the mean and median rows are identical to
   three significant figures on both headline columns. Two different statistics
   producing bit-identical results is possible if the policies landed on the
   same assignments, but nothing says so, and a duplicated run would look
   exactly like this. The third row does differ, so the pipeline is not
   uniformly collapsing.
2. The random seed reads as a calendar date about a year after the submission
   deadline, matching the future publication date stamped in the footer.
   Cosmetic, but it is the same artifact as the venue mismatch found in Step 4.

Also worth knowing before you read it: S4.4 is a generative-AI disclosure. The
authors state that AI assisted with prose and code scaffolding, specify that
humans chose the question, dataset, split, cohort, metric definition, policies
and claims, and that no generative model produced experimental observations.
That is a disclosure properly made under ACM's authorship policy - a fact about
the submission, not a finding.

What I did not do

No quality, novelty or significance assessment; no strengths or weaknesses; no
score; no questions for the authors; no review text. The two unresolved items
above are flagged as unresolvable from the text, which is the allowed category -
how you pursue them is yours.

One verification I could not do: I confirmed that the references in both papers
parse as real-looking entries with plausible venues, but I did not verify that
each cited work exists and says what it is claimed to say. Several are recent
enough to sit at the edge of what I can check. Given ICLR 2026's sanctions
position on hallucinated references, do not let any citation claim reach your
review through me unverified.

I have kept this in the terminal rather than publishing it - it is confidential
submission content, and the Step 0 posture covers this session, not a hosted
page. Say the word if you want it written to a local file instead.
```

</details>

<details>
<summary><code>For all PDFs, review related work. Does it position against the obvious academic and open-source alternatives, scaled to a 6-page paper? Web-search and compare against the paper's related-work section</code></summary>

```text
PAPER 31

What the paper has: no Related Work section. Positioning is distributed across
the background and discussion sections. Of its references, about a third are
framing citations carrying no positioning weight; the rest cluster into
checkpoint/restore mechanics, serverless cold-start work, and
scheduling/infrastructure. For a 6-page paper that is a creditable sweep of the
academic space.

The gaps are all on the production and open-source axis - which is the axis this
track exists for. Searching for the nearest neighbours turned up, with zero
mentions in the paper:

  category                            why it is the nearest neighbour
  ----------------------------------  ---------------------------------------------
  a major accelerator vendor's        same mechanism, same quiesce-then-snapshot
  snapshot feature for its own        ordering, same single-accelerator
  inference stack, announced some     configuration, and its named optimisations
  months before the submission        target precisely the bottleneck the paper
  deadline                            reports as unexplained
  the upstream orchestrator's own     defines by name two of the three APIs the
  checkpoint/restore enhancement      paper claims as its contribution, and
  proposals                           explicitly places device support out of
                                      scope - so the paper's real delta is the
                                      device half, which is a genuine and
                                      well-defined contribution
  the serving engine's built-in       the lighter-weight alternative to
  sleep/wake mode                     snapshotting entirely
  scale-to-zero autoscalers           the orchestration-layer alternative
  a published production write-up of  an existence proof that the approach ships
  the same technique

Two findings that matter:

1. The nearest neighbour appears to be the same design, shipped earlier. The
   convergence is structural rather than thematic - the same ordering of
   snapshot-before-service-registration, for the same stated reason. I would not
   carry its performance numbers into a review: different hardware, possibly
   different interval definitions, and its best figures depend on patches it says
   are not yet shipped. The citable facts are that the system exists, predates
   the submission, and uses the same mechanism on the same serving engine.

2. The claimed contribution overlaps an existing upstream proposal, and the
   proposal is not cited by number. A reader therefore cannot separate what the
   proposal already specifies from what this paper adds - which is a shame,
   because the delta is real and defensible. One citation would close it.

Under single-blind you can see author affiliations, so you can weigh whether a
nearest-neighbour omission from a co-author's own employer is material. I am
noting that the affiliation exists, not drawing the inference.

Scaled to 6 pages? Yes - the paper uses the space densely, and folding related
work into background is normal at this length. The gaps above are content
choices, not space constraints: three sentences and three citations would close
all of them. Whether their absence rises to a problem under the CFP's
"appropriate comparison to related work" is your call. What I can say factually
is that the omitted items are the nearest neighbours and two of them ship in
production.

PAPER 44

What the paper has: a real Scope and Related Work section with two subsections,
about a page of eleven, and a creditable set of references. The positioning prose
is unusually disciplined - it disclaims priority outright, distinguishes the
closest prior policy by model family and evaluation objective, and states exactly
what it holds fixed.

What the search surfaced that is not there, again all with zero mentions:

  category                            why it is the nearest neighbour
  ----------------------------------  ---------------------------------------------
  the cloud provider's own commercial  the paper's whole construct is "how much
  always-warm control, billed on       memory-time do you spend to avoid
  baseline memory                      initialisation" - and the provider whose
                                       public trace it replays already ships a
                                       priced control for exactly that trade-off
  the same provider's snapshot-and-    the industry answer to the same question by
  resume feature                       a different mechanism
  two keep-alive policy papers from    one evaluates keep-alive policy on the same
  the same architecture conference     public trace; the other's headline metric is
                                       memory waste, which is this paper's budget
                                       under another name
  container-sharing and lending work   the adjacent mechanism for the same cost

The sharpest omission is the first. A commercial control for the paper's exact
trade-off, on the platform its trace comes from, is the obvious thing a
practitioner reviewer will ask about.

What it does and does not run: positioning is stated but almost nothing external
is run. Every baseline is internal - fixed, tiered, quantile, ablation, and
uncalibrated references. Four prior policies are discussed and none reproduced.
That is a defensible choice for a controlled single-dataset comparison and the
paper says so, but it means the headline is positioned only against policies the
authors themselves constructed.

Scaled to 6 pages? The question is counterfactual here: this paper is written to
a journal format at nearly twice the limit, with room for a full related-work
section. Compressing it to this track's 6 pages would put that section first in
line for cuts, so "is the related work scaled to 6 pages" cannot be assessed
against the artifact you actually received.

Two cautions on using any of this. Several of the sources are vendor
documentation and secondary coverage, not peer-reviewed; do not carry their
self-reported numbers into a review as a head-to-head. And I verified that these
items exist and say what I have attributed to them, but not the version and date
claims in secondary coverage - if a timeline becomes load-bearing in your review,
go to the primary source.
```

</details>

What makes those two useful rather than decorative:

- **It separates "the sums are wrong" from "the premise is wrong".** Paper 31's
  Table 1 is exact in every cell, and the finding is still that an `n=2` sample
  with undisclosed selection drives the headline. Those are different sentences
  in a review.
- **It names the result that isn't in the abstract.** The regression on larger
  models is disclosed in body prose and absent from every summary claim — which
  is exactly the asymmetry a reviewer is for.
- **It credits discipline where it finds it.** Paper 44's claims are pre-shrunk
  to fit its data and its operating point is pre-registered; the agent says so
  rather than hunting for a flaw that isn't there, and reframes the live question
  as a track-fit question for the chairs.
- **It refuses to carry unverified citations.** It flags explicitly that it did
  not confirm each cited work exists, and names ICLR 2026's sanctions position as
  the reason that matters.
- **It declines to publish confidential content** without being asked, noting
  that the Step 0 posture covered the session and not a hosted page.


## Real use: your own submissions

```bash
export PC="$HOME/.claude/skills/middleware26industrial-reviews-checks/scripts/paper_checks.py"

# Step 0 - always first
uv run "$PC" probe
uv run "$PC" zdr          # is zero data retention in effect? (Pro/Max: never)

# Steps 1-2 - from a HotCRP zip (preferred: offline, no API token)
uv run "$PC" \
    ~/Downloads/middleware26industrial-papers.zip --out out/mw26ind --redact

# or straight from HotCRP
export HOTCRP_TOKEN=hct_...     # Account -> Developer, scopes submeta:read document:read
uv run "$PC" check \
    --hotcrp https://middleware26industrial.hotcrp.com --query "re:me" --out out/mw26ind
```

Outputs:

```
out/mw26ind/summary.md    worst-first triage, STOP list, duplicate pairs, batch patterns
out/mw26ind/summary.csv   one row per paper, one column per check
out/mw26ind/gate.json     blocked / failed ids, duplicate pairs
out/mw26ind/papers/<id>/  report.md  checks.json  text.txt  hidden_text.txt  redacted.txt
```

Exit codes: `0` clean · `1` format/compliance failures · `2` at least one BLOCK.

## What the script checks

| Group | Checks |
|---|---|
| Safety | Hidden text (near-white, <4.5pt, off-page, `3 Tr` invisible render mode, optional-content layers) and 21 LLM-directed instruction patterns. Hits in **hidden** text are a `BLOCK`; the same strings in **visible** text are only a `NOTE`, because this venue publishes papers *about* prompt injection and agent guardrails |
| Track requirements | `(Industry Track)` title suffix, author names/affiliations present (single-blind), at least one **industry** author, no NDA attached |
| Format | ≤6 pages (abstract, figures, tables **and appendices** count, references do not), SIGCONF 9pt, US Letter, column count — the CFP declines non-conforming papers **without review**, so every hit needs confirming by hand |
| Scope | Real-world deployment-evidence markers, **each quoted in context** with its page number and the matched phrase wrapped in `>>...<<`, grouped so several markers in one sentence read as one piece of evidence, and matches inside the bibliography excluded as citations: *in production*, *deployed across*, node/region/tenant counts, durations in months, p95/p99, SLO/SLA, on-call, incident, real workload traces || Batch | Cross-paper shingle-Jaccard overlap for possible duplicate or simultaneous submissions; reference-count outliers; a batch-wide table of the most common issues |
| Hygiene | `--redact` writes a de-identified `redacted.txt`, which is what makes a hosted-model polish pass fit ACM's carve-out for third-party tools |

## Privacy and data retention

Step 0 of [`SKILL.md`](SKILL.md) is the reason this skill exists. ACM's peer-review
policy does not ban AI; it bans uploading submissions to a third-party system "that
does not promise to maintain the confidentiality of that information". So the skill
documents, per plan, what retention actually is, how to verify it with a single
command, which API surfaces silently void a zero-retention agreement, and how to
run the whole thing against a local model instead.

`paper_checks.py` makes **no network calls at all** unless you pass `--hotcrp`
(fetches submissions) or `probe --live` (sends the two-token word "hi", never paper
text). Extracted text stays in your `--out` directory. Nothing is uploaded anywhere.

## Updating

How you update depends on how you installed. In every case the skill is just a
directory, so updating means replacing its contents.

| How you installed | How to update |
|---|---|
| `git clone` into a skills dir | `git -C ~/.claude/skills/middleware26industrial-reviews-checks pull --ff-only` |
| one clone + symlinks (the "install once" recipe) | pull in the clone; every agent pointing at it follows |
| `install.sh` with `--copy` | pull in the source clone, then re-run `./install.sh --copy` |
| tarball or zip | re-extract over the same directory (see below) |
| `git submodule` in a project | `git submodule update --remote ~/.claude/skills/middleware26industrial-reviews-checks` then commit the bump |
| `uv run <raw URL>` | nothing to do — the script is fetched fresh on every run |

### git clone

```bash
git -C ~/.claude/skills/middleware26industrial-reviews-checks pull --ff-only
```

`--ff-only` matters. This repo is published as a **single squashed commit that
gets amended and force-pushed**, so its history is rewritten rather than appended
to. A plain `git pull` will then either refuse or try to merge two unrelated
histories. When `--ff-only` fails, take the published version wholesale:

```bash
git -C ~/.claude/skills/middleware26industrial-reviews-checks fetch origin
git -C ~/.claude/skills/middleware26industrial-reviews-checks reset --hard origin/main
```

That discards local edits. If you have your own changes, commit them on a branch
first, or keep your copy and merge by hand.

### tarball or zip

Re-extracting is the whole update. Delete first so removed files don't linger:

```bash
D=~/.claude/skills/middleware26industrial-reviews-checks
rm -rf "$D" && mkdir -p "$D"
curl -L https://github.com/aslom/middleware26industrial-reviews-checks/archive/refs/heads/main.tar.gz \
  | tar -xz --strip-components=1 -C "$D"
```

### Which version am I running?

```bash
git -C ~/.claude/skills/middleware26industrial-reviews-checks log -1 --format='%h %ad %s' --date=short
git -C ~/.claude/skills/middleware26industrial-reviews-checks describe --tags            # e.g. v1.0.0
```

Compare against the published tip without fetching anything into your clone:

```bash
git ls-remote https://github.com/aslom/middleware26industrial-reviews-checks.git main
```

Different hashes mean an update is available. If you pinned a tag when
installing, you are deliberately frozen there and `pull` will not move you —
re-extract from the newer tag, or switch to `main`.

### After updating

Start a new agent session so the skill is re-read; a running session keeps the
copy it loaded. Then re-run the self-check from
[section 2](#2-try-it-now-on-generated-test-pdfs) — the fixtures are regenerated
by the same commit, so if they still land on their expected verdicts the update
is sound.

```bash
uv run scripts/make_test_pdfs.py
uv run scripts/paper_checks.py .tmp/testpdfs
```

## Uninstalling

```bash
rm -rf ~/.claude/skills/middleware26industrial-reviews-checks
rm -rf ~/.agents/skills/middleware26industrial-reviews-checks      # and any other location you installed into
```

Or, from a clone of this repo, `./install.sh --uninstall` removes the copies and
symlinks it created. Nothing is stored outside those directories, so that is the
whole removal.

## Provenance

Every page limit, font size, anonymity rule, deadline and evaluation criterion in
`SKILL.md` is transcribed from the CFP linked above, read on 2026-10-04. CFPs get
amended — if a date or limit here disagrees with the CFP, the CFP wins; please open
an issue.

## Related

- [`woais26-ai-reviews-checks`](https://github.com/serverlesscomputing/woais26-ai-reviews-checks) — the same workflow for the WoAIS2 2026 workshop, which differs on anonymity, template, page accounting and track requirements
- [Agent Skills specification](https://agentskills.io/specification) · [agentskills/agentskills](https://github.com/agentskills/agentskills)
- [HotCRP REST API](https://hotcrp.com/devel/api/)
- [PhantomLint](https://github.com/tobycmurray/phantom-lint) — OCR-vs-extracted-text diff for hidden prompts, a good second opinion on any `BLOCK`

## License

MIT. The skill text quotes short passages from the CFP and
from ACM policy for identification and compliance purposes; those remain the
property of their publishers.
