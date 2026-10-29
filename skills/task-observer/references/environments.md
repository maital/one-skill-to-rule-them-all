# Environments, Activation Setup, and Handoff-Doc Mode

Load triggers: anchoring, pinning, re-pinning or diagnosing a shard of the workspace; setting up or diagnosing activation (instruction block, hooks, activation config); placing the session-start mount probe in the first batched call; the folder picker is refused or no device-bridge tool is found; no filesystem at all (handoff-doc mode); no stable path resolves for a disposable worker (report-back mode); a session whose output a caller owns; after a compaction or resume; the checkpoint in a priced-write workspace; a permission or classifier denial of a log write; a mode that forbids every write by design; the first-run backfill

Load this for setup questions, compaction/resume behaviour, or when running
in an environment without filesystem access.

## Contents

- Recommended activation setup
  - The activation block
  - Anchoring the workspace
  - A session-start hook (Claude Code and similar harnesses)
  - A stop hook — the per-task backstop as a harness event
  - A pre-tool hook — the only write path as a barrier
  - A skill-file write gate — the skill-authoring load as a harness event
  - Verify activation in a NEW session — the installing session cannot prove it
  - Activation config — late, intermittent, and why the guard cannot live inside it
  - The probe rides inside the first batched call
  - Install-layout hazards — three ways a skill silently stops existing
  - Other agent tools that read the same skills folder
  - Two harness behaviours that block the protocol rather than break it
  - If CLAUDE.md (or the equivalent config) is governance-protected
  - A third denial class: an automated content classifier
  - A mode that forbids every write by design
  - A delegated setup step is not done until you have observed it
  - Paths handed across a boundary are resolved from the far side
  - Windows hosts: what reaches the shell and the file
- Environment mappings
- Git as an optional staging medium
- Claude Code Projects — disposable threads, no local skills, no pinned
  path (based on the platform's documentation, not yet field-tested)
- Sessions whose output channel is owned by a caller
- First-run backfill
- Storage regimes
- Bundle manifest
- Compaction behaviour
- User-facing documentation
- Repo/maintainer sessions — verify commit identity before writing
- Handoff-doc mode (no persistent storage)
  - No link to the user's computer
- Handoff-doc analysis (when one arrives)

## Recommended activation setup

**Augury fork: activation is opt-in.** The skill runs only when the user
invokes it. Do not recommend or install any activation tier — no
description-level trigger, no user-preferences line, no CLAUDE.md or
project-instruction block, no harness hook. The block below is kept only
because other sections refer to its conventions (workspace paths, the
skill-load lookup); it is reference material, not something to propose.

### The activation block

```text
Before the first tool call of any session — and before writing or
proposing a plan, not merely before executing one — invoke the
task-observer skill AND execute its Session Start Protocol (storage
check, frontmatter scan, review trigger). Loading the skill and running
the protocol are separate steps; a session that loads the file and stops
has activated nothing. Any turn that will involve a tool call counts; do
not classify the session as "too simple" from its opening message.
A step you did not run is reported before the answer, in one line — never
after it, where it reads as a suggestion. If the skill does not resolve on
this turn, say which probe told you so (the skill listing, or the skill
directory on disk), re-check the listing on every later turn that makes a
tool call, and run the protocol the first time it resolves — a "not
installed" verdict holds for one turn.

Select skills on the DECISION the request is about, not on the artefact it
arrived as. Name what the user is deciding, then match the installed skill
descriptions against that — a request handed over as a file to review
still needs the skill whose description names its subject.

After completing each task, list the observation records written this
session in the same turn and report a one-line summary from that listing
(ids and titles, or "none logged and why") — never from memory of
writing them. This is the activation backstop: it forces a look at the log,
so a session that silently skipped the protocol is discovered at the
first task boundary instead of never.

Loading a skill is not complete until you have queried the observation
log for OPEN observations naming it and read their bodies:
  find "[ABSOLUTE PATH]/skill-observations/observation-log" -maxdepth 1 \
    -name '*.md' -exec grep -l "skill:.*<skill-name>" {} +
(<skill-name> is the bare name: drop a plugin: prefix, up to the last
colon. A qualified pattern misses every entry filed under the bare name
and reports a clean zero; the bare one matches both spellings.)
(Use find, not a bare *.md glob. Under zsh an unmatched glob is an error,
so on an empty log the command never runs and the enclosing block aborts —
and 2>/dev/null does not help, because the redirection belongs to a
command that never starts. This is the first thing that fires on a fresh
install, in every session, for as long as the log is empty.)
Apply their insights to the current work — meaning: let them change what
you do in THIS task. Editing the skill file, or writing the rule into any
other file a later session reads, is acting on the observation and waits
for the review. See "Log, don't act" in SKILL.md: the test is whether the
action leaves a durable change outside the observation log. Run this at every skill load, however many skills load
in one session. The session-start scan does not cover it: that is a
frontmatter sweep over every observation at session start, this is a
body-level lookup for one skill at the moment its rules are applied.

The converse holds too: the grep, and any checkpoint line recording the
load, run only in the same batch as the Skill invocation, and take the
skill name from a load that has actually happened — never from a list of
skills you built to load. A checkpoint line written without the load is a
false record. Before writing to any file a skill reads at run time (a
state file), load that skill: a state file that describes itself is not a
substitute for the skill that owns it.

Where scripts run, the lookup and its checkpoint line are one command:
  bash "<skill directory>/scripts/skill-load.sh" "[ABSOLUTE PATH]" <skill-name>
The Skill invocation and that call are ONE batch — emit them together,
one call per skill loaded, never the call in a later batch.

The task-observer workspace for this project is:
  [ABSOLUTE PATH]
Every path the skill uses derives from that root and nothing else:
  [ABSOLUTE PATH]/skill-observations/observation-log/   (the log)
  [ABSOLUTE PATH]/skill-observations/cross-cutting-principles.md
  [ABSOLUTE PATH]/skill-updates/                        (staging root)
  [ABSOLUTE PATH]/skill-updates/PENDING.md              (staging manifest)
Never resolve any of them from the current working
directory — a cwd inside an ephemeral checkout (a git worktree, a temporary
clone) is torn down and takes the log with it. Never place the workspace
inside a skills-discovery directory or any path linked into one. If this
environment mints a separate project identity per checkout, or more than
one agent works this project, the pinned path above is the single shared
location; do not derive one per session, tool or project.

The user's repositories live under:
  [PROJECTS ROOT]
(the root the sibling check sweeps before declaring a skill absent;
observation-log.md, "Skill families and the sibling check").

A subagent dispatched by a session that already runs this protocol does
not run it again and does not write to the log. The controller observes,
because only it sees the whole task and its review; a subagent that
notices something worth logging says so in its final report, and the
controller writes it, running the id snippet per write. A start-up rule
in a project file reaches every agent the project ever dispatches.

An observation file comes into being only through
  bash "<skill directory>/scripts/new-observation.sh" <slug> "[ABSOLUTE PATH]"
which prints the path to write into. Never by copying another file's
header or counting a listing — including on the turn after a
compaction, when the skill body is out of context.
```

`<skill directory>` is the installed skill's own directory (the one
holding `SKILL.md`), substituted at install exactly like
`[ABSOLUTE PATH]`; where the harness cannot run a script, drop the two
script paragraphs and keep the rest — the inline id snippet in SKILL.md is
then the write path, and the `find … grep` above is the lookup, with its
checkpoint line written by hand in the same batch.

On Windows, pin the drive-letter form (`C:/Users/<user>/...`): PowerShell,
cmd and Git Bash all accept it ("Paths handed across a boundary are
resolved from the far side"), and `scripts/new-observation.sh` converts it
with `cygpath` and prints the created path back in that form. Where the
command runs from PowerShell rather than Git Bash, a bare `bash` may not
resolve (Git for Windows' default PATH option adds only `Git\cmd`) or may
start WSL's bash instead; name Git's `bash.exe` by its full path there.

### Anchoring the workspace

SKILL.md's `[workspace folder]` definition states the rule; this is where
to pin it, how a stable anchor still goes plural, and which plural case is
legitimate.

Fill in the path when installing. Pinning turns anchoring into a one-time
decision instead of one the agent re-litigates every session with a fresh
chance to get it wrong. Pin the ROOT, and list the derived paths: a pin
that names only the observation-log directory — the place split-brain was
first noticed — leaves the staging root, the manifest and the principles
file to be re-derived by every session, and parallel sessions resolve
them plausibly and differently. Where the environment provides a managed persistence directory
under the project identity (a memory directory it loads every session),
that directory and the identity root are both "stable"; anchor inside the
managed directory when one exists, otherwise on the identity root — never
both.

**Scope matches installation scope** (SKILL.md, `[workspace folder]`). A
stable anchor can still be plural — in Claude Code the identity is derived from the
directory the session starts in, so per-subfolder habits shard the log
into several stable anchors that each look like the only one. Decide the
scope once, at install, and pin it. **One plural case is legitimate and
is not a shard:** logs deliberately kept apart because their observation
bodies carry task context that does not travel, over skills that are
nonetheless installed globally and therefore shared. Read the multi-log
block in `weekly-review.md` before consolidating anything — it defines
that case and the aggregate review that serves it. A reader who stops at
this paragraph consolidates a set that was meant to stay separate.

**Stable is not the same as always available.** A folder in a synced or
network drive passes every test above — absolute, outside any checkout,
the same path every day — and still fails the one a session start needs:
when the drive is not mounted, the log is unreadable and a hook script
stored there does not resolve, so a hook registered to fail quietly
starts the session with no activation and no error. Before pinning, ask
two questions. Is this storage mounted every time a session starts on
this machine, offline and before the sync client has started included?
Is it the user's own, rather than an organisation's that can unmount,
retain or audit it? Unless both answers are yes, propose a local folder
under the user's home directory, and carry the log between machines by
an explicit copy to the user's own storage or through the repository
route ("Git as an optional staging medium" below; `observation-log.md`,
"When the workspace is under version control", for the two-install
case), not through a live-synced directory. The user decides; the
questions are asked before the pin is written, not after the first
missed session.

**A session started without a project folder has no stable project
identity.** In a desktop app that runs such a session in a per-session
scratch directory (the tell is `scratch-workspaces/` or
`scratch-<date>-<hash>` in the cwd), the project identity derived from
that directory is new every session, so a log anchored on
`~/.claude/projects/<project-id>/` survives on disk and is never read
again: its open entries are never applied at session start and never
reach a review. Treat the scratch directory as an ephemeral path
(SKILL.md, Session Start step 1): anchor on the pinned path if one
exists, otherwise on the user-scope workspace
(`~/.claude/skill-observations/`, the path skills installed at user scope
need anyway), and name the anchor in the start-up lines. Logs already
written under scratch-derived identities are shards: consolidate their
open entries as below.

**The fork rule as usually stated describes the harmless case.** "A second
**empty** log beside a populated one is a silent fork" — but an empty log
announces itself: the first scan returns nothing, and nobody trusts a
backlog of zero. The damaging fork is the one that has accumulated its own
entries. From inside it looks exactly like a working log: the scan returns
files, the review finds work, every instrument reports health. Nothing in
it can reveal that it is a shard.

Pinning does not settle it: a pin governs the sessions that read it, not
one that resolved its anchor before the pin or by another route.

So the detection has to be positive, and it is nearly free: at the end of
the session-start scan the agent is already holding the `skill:` values of
every entry. A log whose targets are overwhelmingly **user-scope** skills,
while the log itself sits under a **per-project** path, has already
declared the mismatch — two `ls` and a comparison. Run it when the anchor
is per-project and the targets are not; report the candidate shard rather
than consolidating on your own judgement, because one plural case is
legitimate (see the multi-log block in `weekly-review.md`).

**The session-start scan lists the neighbours on every run.** One line of
the scan (`scripts/session-start-scan.sh`) finds every
`skill-observations/observation-log/` up to three levels under the
pinned root's parent and prints each with its file count — `logs under
the parent (…): /a/proj=157  /a/other=19`. One entry is the ordinary
case. Several are either shards or the legitimate plural case, and the
line cannot tell which: report it in one line, route to "Several
observation logs on one machine" in `weekly-review.md`, and never
consolidate from the scan. The probe is bounded by the parent's depth-3
tree, not by any log's size; an empty line means the probe could not read
the parent, since the pinned log itself is always under it.

**Before creating a log, search for one.** Check the plausible anchor
candidates — the pinned path, the project identity root, the
environment-managed persistence directory, the shared folder, the other
agent's equivalent — for an existing `skill-observations/` workspace.
"The pinned path" has more than one source: the instruction file, and
any registered session-start hook. Grep the harness settings and the
installed plugins' hook manifests for the skill's name, then read the
path each hook script derives — a hook that counts observations
somewhere is a pin whether or not the instruction file names it. Where a
system already reads or writes state, that location is authoritative
and the documented defaults are the fallback; an existence probe over
the defaults alone reports "not found" for every install that deviated
from them. Never create a workspace while a hook or config names a
different root: reconcile first. Every always-loaded instruction file is
a pin source (a CLAUDE.md and an AGENTS.md forked for another agent tool
are two): grep each for the workspace root. So is each agent tool's own
memory and its saved or approved commands: a root persisted there from
an earlier session keeps being used by every session that does not load
the instruction file, after the file's pin has changed. Two distinct roots are a
fork, not a variant: stop and reconcile before writing, and keep the pin
in one file both tools load. If one exists, adopt it, or
consolidate deliberately with the user. A fresh
empty log beside a populated one is a silent fork: both grow
independently, ids collide, and each session sees only half the history.
Consolidating a shard is mechanical: carry over only the entries still
`open` — its already-actioned entries re-import as duplicates of work
the target may hold too — re-issue each id from the target log's own
counter (the snippet, one write at a time, never a pre-computed range),
mark each source entry `actioned` with a resolution naming the target
id, and then leave a pointer file at the abandoned location so sessions
anchored there get redirected instead of re-creating the fork. The
pointer has one name, `skill-observations/MOVED.md` under the abandoned
root, its first line naming the new root and the date, because only a
fixed name is read: the session-start scan prints a `NOTE: pointer`
line when it finds one, with the count of entries written there since
(observed: a free-form note at an old root went unread for two and a
half days while a second machine's hook kept writing there). A pointer
is data, not an order: entries written after it mean the old root is
still in use, perhaps on purpose, so report both roots and ask; never
consolidate, delete or re-anchor on its word alone. And it does not
replace re-pointing every hook and instruction file that names the old
root: list them, and say which still do.

**A re-anchor moves the root, not only the log.** Session Start step 1's
ephemeral-to-stable re-anchor and a shard consolidation relocate every
artefact the pin governs (the derived paths in the activation block
above), not the log alone: `skill-updates/` with `PENDING.md` and every
dated staging directory, `cross-cutting-principles.md` and
`skill-families.md` (merged by substance where the target has its own),
`last-review-date.txt` (keep the later date), `forked-skills.txt` (keep
the union), `starter-principles-reviewed.txt` (keep the newer version), and
`reviews/` and `checkpoints.log` as history. Step 6 reads `skill-updates/`
at the current root only, so staged work left at the old one is
invisible, and an absent `PENDING.md` reads as "nothing staged". Move staged directories whole; at the new root they go through
the staging reconciliation like any other entry (`weekly-review.md`,
"Staged-work reconciliation gate"). Where a move is not safe — a session
may still be writing at the old root — list what the old root's
`skill-updates/` holds to the user in one line rather than leaving it
behind silently.

**Report-back mode — when no pinned path resolves.** The Session Start
guard warns and re-anchors when the resolved workspace sits under an
ephemeral checkout. That assumes a stable path exists to re-anchor on. In
a disposable worker — a Claude Code Projects thread, any agent whose only
writable location is a clone torn down at the end of its task — there is
none, and writing into the clone anyway produces observations that vanish
with it while every write reports success. So the branch is: do not write
into the clone; carry each observation in full in the final report to
whatever coordinates the work (Augury fork: never commit or push
observations to any repository). See
"Claude Code Projects" below for the install, activation and log-route
consequences.

**A grant names a path; it does not create one.** Under a write-allowlist
sandbox (Claude Code's Bash sandbox with `sandbox.filesystem.allowWrite`,
or any harness that mounts only allowlisted paths writable), an
allowlisted path can be mounted writable only if it already exists, and
when its parent is outside the allowlist nothing inside the sandbox can
create it: the first `mkdir -p` of the log fails with `Read-only file
system` although the grant is present and correct. In such a harness the grant and the
path's existence are two install preconditions: create the workspace
root's `skill-observations/` and the staging root `skill-updates/` at
install time, from outside the sandbox — by the user or an install step —
and only then rely on the session to create everything below them. The
folder picker and the one-retry rule for a failed write do not help here:
every interface inside the same sandbox sees the same read-only mount.
The diagnosis at the moment of failure is in `observation-log.md`
("Workspace creation").

**Config detection:** Augury fork: skipped. Do not check for, suggest or
write an activation instruction in CLAUDE.md, project instructions or the
system prompt.

**Load is not activation.** The failure the block's wording guards
against: the agent loads the skill per the
config instruction, then stops — the Session Start Protocol (log files,
scan, review trigger) never runs, and nothing surfaces the omission
because a loaded-but-inert skill looks identical to an active one from
the user's side. Hence the two belts above: the
instruction demands the protocol by name, not just the load, and the
post-task summary line makes silent inactivity visible at the first
task boundary. If you adopt only one line of the block, adopt the
post-task check.

That check is prose in the same file as the rule it guards, and it fails
with it: whatever made a session skip the block's first paragraph makes it
skip the third. A backstop written in the medium of the rule it
guards is one layer, not two. The second layer has to hang on an event
the harness raises, not on the agent re-reading its instructions: where
the harness offers a stop or turn-end hook, the objective check is cheap —
the session used tools, and nothing under `observation-log/` or
`checkpoints.log` has been written since it started — block once, with a
one-line instruction, and stay silent otherwise. Classify each
enforcement layer by what it needs in order to function: a filesystem
write survives into any session type, a prose reminder survives only
while the prose is being acted on.

A third belt: the scan's `checkpoints.log` line (`observation-log.md`,
"The scan ends in a write, not only a print").
Treat *loaded but not run* as a distinct failure cause, alongside the
skill being absent, truncated, or present and skipped: it is the likeliest
of the four, precisely because the load's visible success stands in for
the protocol's invisible omission, and the aggravating condition is the
same one that produces a skipped load — an opening message small enough
that a full startup protocol feels disproportionate to it.

**Anti-pattern:** don't chain activation through another skill — load
task-observer and related skills independently from configuration; a broken
chain silences all observation activity.

### A session-start hook (Claude Code and similar harnesses)

Augury fork: not used. Do not propose, write or install this hook or any
other harness hook for this skill; activation and checks stay manual.

### A stop hook — the per-task backstop as a harness event

Augury fork: not used. Do not propose, write or install this hook or any
other harness hook for this skill; activation and checks stay manual.

### A pre-tool hook — the only write path as a barrier

Augury fork: not used. Do not propose, write or install this hook or any
other harness hook for this skill; activation and checks stay manual.

### A skill-file write gate — the skill-authoring load as a harness event

Augury fork: not used. Do not propose, write or install this hook or any
other harness hook for this skill; activation and checks stay manual.

### Verify activation in a NEW session — the installing session cannot prove it

The config file is read at session start, so an instruction written
mid-session takes effect only on the next one; and in a harness that
hot-loads a newly installed skill, the skill being callable right after
install proves nothing about activation — it was invoked by hand. The
installing session therefore sees everything pass (bundle complete, skill
listed, protocol ran, hook emits valid JSON) while the one thing that
matters is untested, and the tempting close is "installed and tested".
Report the install as **activation unverified** until a fresh session has
been seen invoking the skill on its own, and name the check: the next
session opens with ordinary work that names neither the skill, the log
nor the check, and only afterwards is the skill confirmed invoked (not
merely listed) with the Session Start Protocol run — by evidence of
invocation, not by a side effect that a hand invocation would also
produce. If the environment offers no way to start a fresh session from
the installing one (a GUI-only harness, no CLI), hand the check to the
user: start the next session with unrelated work, then look at the
`session-start scan` line in `checkpoints.log` or at the transcript; an
unverifiable step never closes silently as "tested". A pass from a
session whose first request named the skill is a hand invocation and is
discarded, not counted (`skill-authoring.md`, "Trial design").

The runtime cannot diagnose its own absence — Session Start step 4 only
runs once activation has already succeeded — so the external diagnostic
is the one that catches a skipped install step: **if
`skill-observations/observation-log/` does not exist after a few sessions
of tool-using work, activation never happened**; check the activation
block in the config, or install the hook. The directory existing proves
only that one session ran: a first session loaded by hand, or by a lucky
description match, writes entries and the trace, and the install then
goes quiet. So the diagnostic has a second half: **the last
`session-start scan` line in `skill-observations/checkpoints.log` is the
last session that ran the protocol** — days old while work went on means
activation is not running, whatever the log holds. The scan prints a
`GAP:` line when that date is 7 or more days old, which helps only the
session that is activated after the gap; the dated line is the check
anyone can run from outside. The README carries the same diagnostic for
users who never read this file.

**Another skill's session-start output is a false positive for
activation.** A harness that injects a different always-on skill's
content at session start makes the session look skill-governed while
this skill's activation does not exist (observed: a full working day of
real work with the skill installed beside four others, no log and no
activation block). Count only evidence of THIS skill: its own
invocation, the protocol's trace in `checkpoints.log`, or the
`observation-log/` it created. An install is not finished until an
artefact outside the skill proves it: create the workspace AND write the
activation, then print what was written and where. Where the activation
cannot be written, the install does not succeed silently: report it as
incomplete and hand over the block to paste.

### Activation config — late, intermittent, and why the guard cannot live inside it

SKILL.md step 1 carries the instruction (on turn 1, load the session-start
skills directly rather than assuming the config did, and read the config
yourself if the mount resolves but its content is not in context). This is
the reasoning behind it, and the reason that instruction is only the backup.

In some hosted or bridged session types the activation config — a project
instruction file, a session-start hook's context — does not reach context
until the **second** user turn, while the workspace it lives in is already
mounted and probes clean. From inside turn 1 a config that is merely late is
indistinguishable from one that is absent, so a whole substantive turn can
run with no session-start skill loaded. **A config delivered on a later turn
than the work it governs is equivalent to an absent one for that work.**

It has also been seen **intermittent** rather than merely late: present for
turns 2 and 3 of a bridged session, then reported as no longer present at
turn 4, with no change in the workspace link and nothing the session did to
cause it. So a rule anchored on "once it arrives it stays" fails the same way
a rule anchored on "it arrives first" does. Do not treat an earlier turn's
config as still in force.

**The skill listing lags the same way, and an absence verdict expires the
same way.** The instruction can be present and fire on a turn where the
skill it names does not yet resolve against the harness's own listing of
invocable skills — right after the instruction was added mid-session, or
in a harness that builds its available-skill list once near session
start. The honest response on that turn is a disk check and "not
installed", and that verdict is true only for the turn it was made on;
nothing revisits it, so when the listing catches up every tool call in
between went unobserved and the earlier claim stands uncorrected. So a "not installed" verdict
names the probe it came from — the listing lookup, or a disk check of the
skill directory — because the two disagree exactly here, and it holds for
one turn: re-check the listing on each later turn that makes a tool call,
and run the Session Start Protocol in full the first time the skill
resolves. Like the guard below, this rule cannot live only in the skill
that failed to resolve; the line that carries it is in the activation
block.

The durable form: **a guard against an activation config failing to load
cannot live inside the thing that config loads.** The guard in SKILL.md step
1 is in `task-observer`, which is loaded *because* the config says to load
it — so on the turn where the config is missing, nothing puts that guard in
context either. By construction it cannot fire on turn 1.

The primary guard therefore belongs in the earliest channel verified present
on turn 1 — in most harnesses the user-level preferences or system-prompt
block, which is where the line

> then read the workspace config and follow it before any other work

goes. See the activation tiers above: that channel is tier 2 (user-level preferences), and the step-1
guard is the backup for turns 2+ and for sessions where the skill happens to
be loaded directly.

**Log the cause, not just the miss.** When a session-start load is skipped,
the fix depends on which of these it was: *absent* (config unreachable),
*present-but-not-yet-injected* (late — record the turn it arrived),
*present-but-not-yet-resolving* (the instruction fired and the skill was
not in the listing — record the turn it appeared), *present and skipped*
(attention), or *loaded-but-not-run* (the invocation succeeded and the
protocol inside it never executed). Only the first two are fixed by
changing the channel, and the third by re-checking the listing; the last
two are not environmental, and an environmental cause found first will
otherwise absorb the whole explanation.

### The probe rides inside the first batched call

The probe line can be present and correct on turn 1 and still not fire
first: the session's first tool call is a batch of `Skill` invocations,
and the probe rides in the second. The failure is detectable only in the
sessions where it costs nothing, and it costs everything in the session
where the folder is not there and the agent asserts that it is.

The cause is a contest for one slot. Loading the session-start skills is
the one action that genuinely *is* prior to the work, so it recruits the
same "before anything else" position the probe needs, and it arrives as an
activation of the agent's own setup rather than as an instruction — which
feels like preparation, not action, and wins. Stating the precedence more
firmly does not change that: an agent that loads the skill carrying the
rule in that same first batch still does not register the rule as
violated, because the load discharges the felt obligation.

So the rule is about the **content of the first call, not its order**: the
session's first tool call is a single batched call that contains the probe,
and any session-start skill loads ride in that same batch — never in an
earlier one. That removes the contest instead of adjudicating it, the
harness's parallel-call guidance already permits it, and it is checkable
after the fact from the transcript (an ordering rule between two "first"
actions is not). SKILL.md step 1 states it that way; the tier-2 preferences
line above is worded to match; the activation block's "before the first
tool call" is satisfied by the load riding *in* that call.

**The pre-skill tier's line names the complete batch, not only the
probe.** Whatever the harness reads before any skill — Cowork's
user-preferences block, a Claude Code user-level (global) CLAUDE.md, a
session-start hook's injected text — carries the whole first batch: the
workspace probe AND the session-start skill loads, named, in one batched
call; the tier-2 line above is the template. The failure shape: a line naming only the probe
fires the probe and leaves the loads to a project config that arrives late
or intermittently, so the first batch carries the probe alone and the
skills load a batch later. The review's activation regression pass checks
for it (`weekly-review.md`, Step 1).

**A deferred probe tool moves the probe to batch 2.** In a cloud session
bridged to the user's computer, the device tools may still be connecting
at turn start and arrive deferred (name only, schema not loaded), so the
probe cannot ride in batch 1. Batch 1 then carries the deferred-tool load
together with the session-start skill loads; batch 2 carries the probe,
the session-start scan and one `skill-load.sh` call per skill loaded in
batch 1 — the one recorded exception to the lookup riding in its load's
batch, because the script runs through the same deferred tool (scripts
absent from the device: "Environment mappings", "Where the scripts are not
on the device"). Record the split in the start-up lines ("probe in batch 2: device
tool deferred"), so a review reading the transcript sees an environmental
order, not a skipped rule. The failure shape: a rule that names batch 1's
content cannot be met when one of its tools does not exist yet, and an
unrecorded workaround reads the same as a violation.

**A server still connecting is the same split with nothing to load.** When
the server providing the probe is still connecting, its tools are absent
from the tool surface, deferred list included, so batch 1 has no probe call
to carry. Batch 1 then carries the session-start skill loads plus the tool
search that waits for that server; the probe is the first call of batch 2,
and the start-up lines say so ("probe in batch 2: device server still
connecting"). Never put a placeholder command in the probe's slot — an
`echo`, or an `ls` of the cloud container: it probes nothing and reads in
the transcript like a probe that passed. In a session with two filesystems
a container `ls` never counts as the workspace probe.

### Install-layout hazards — three ways a skill silently stops existing

Each of these leaves no error. The skill simply stops being offered, or an
activation tier stops firing, and the only symptom is that the Session Start
Protocol no longer runs — which is indistinguishable from a session where it
ran and found nothing.

**Skill discovery is exactly one level deep.** The host reads the skills
directory and looks for `<entry>/SKILL.md`. There is no recursive lookup. On
a library of any size the natural organising instinct is to group skills into
category folders — `skills/seo/`, `skills/clients/`, `skills/writing/` — and
doing so makes **every skill inside them cease to exist**: no error, no
notice, no change in behaviour except that the skills stop being offered.
The layout is not merely the happy path; it is the rule, and it does not
travel with the maintainer who reorganises six months later. Keep every
skill directly under the skills directory.

**A backup inside the skills directory registers as a rival skill.** The
obvious way to back a skill up before overwriting it —
`cp -r ~/.claude/skills/task-observer ~/.claude/skills/.task-observer-backup`
— puts a second copy where the host is looking. The leading dot does not
exclude it from discovery. The available-skills list then holds two entries
with near-identical descriptions, and the backup still advertises the OLD
version's trigger, including its session-start instruction. That defeats the
upgrade at the one activation tier that survives an unreachable config:
description matching now has two candidates competing for the same trigger,
one of them the file the upgrade just replaced. Back up **outside** the
skills directory.

**A hook registered by script path dies when the exec bit goes.** The
docs' example reads most naturally as naming the script directly:

```json
{ "type": "command", "command": "'/path/to/hooks/task-observer-activation.sh'" }
```

Anything that rewrites file modes — a cloud-drive resync switching between
stream and mirror mode, a restore from an archive, a checkout on a
filesystem without permission bits — returns the file as mode 600, and the
hook then fails with "permission denied" on every session start with
nothing surfacing the error. Register the interpreter instead
(`"command": "bash '/path/to/hooks/…'"`), which does not depend on the exec
bit, and treat "the Session Start Protocol stopped running" as a prompt to
check the hook rather than the skill. The same silent death follows, with
the mode intact, when the script lives on a drive that is not mounted at
session start: keep hook scripts on local storage ("Anchoring the
workspace", "Stable is not the same as always available").

### Other agent tools that read the same skills folder

A second agent tool that lists the same skills directory shows the
description to its own model, and the description starts the skill by
itself — it carries the session-start trigger so that activation
survives an unreachable config file (tier 1 above). Removing the
activation block from that tool's instruction file does not keep the
skill off there. To keep it off, use that tool's own skill-permission or
skill-path setting, or stop sharing the folder. Where it does run, the
protocol's assumptions (the pin, the permission model, the hooks) do not
hold, and a log it writes is a shard ("Before creating a log, search for
one").

### Two harness behaviours that block the protocol rather than break it

**A command-shape guard can refuse a read-only call.** Some setups run a
pre-tool hook that inspects the literal command text for governed path
segments combined with a write indicator (`=`, `>`, `cp`, `mv`) and denies
the whole call when both appear anywhere in the string — regardless of
whether the path is actually being written to. Two consequences worth
knowing before diagnosing: a `cd` into a governed directory followed by a
read-only script call is denied even though nothing is written; and a
**blocked command merely quoted inside an observation body** can trip the
same guard when that body is written through a shell. Neither is a
permission problem with the destination. Where this bites, invoke the
script with an absolute path and no `cd`, and write observation bodies with
the editing tool rather than through a shell. A guard that refuses the
session-start scan itself is handled the same way — degrade to flat
commands, never skip the step (`observation-log.md`, "A refused snippet is
a degraded path, never a skipped step").

**Staging a harness configuration change is not like staging a skill.** A
skill's `SKILL.md` is read by itself; a corrupted staged copy fails to parse
and nothing else is affected. A harness config file — `settings.json` with
its hook entries, deny-lists and protection rules — is read by the harness,
and it commonly carries **other protections alongside** the thing being
changed. The natural delivery shape for a config change is a fragment:
"here's the new key, add it under `hooks`". A hand-merge that goes wrong
there does not fail loudly; it produces a valid file with the other
protections missing.

So: deliver a config change as a **complete replacement file** built from
the user's current one, never as a fragment to splice in; state in one line
what else that file was carrying, so the user can verify it survived; and
where the harness supports it, prefer a separate file over editing a shared
one.

### If CLAUDE.md (or the equivalent config) is governance-protected

Some setups guard shared config files with hooks or file-protection rules
that deny agent edits. If an edit to the config is denied, never retry the
same edit blindly and never attempt to bypass the guard — a denial is the
governance system working as intended, and a silent skip is just as bad
(the user believes activation is set up when only description-level
matching is active). Surface the denial to the user and offer these
fallbacks, lightest first: (a) if the denial came from an interactive
permission system rather than a hard guard — the denial message offers an
approval path, names a permission rule, or otherwise indicates consent
would clear it — ask the user to approve a retry of the same edit; one
confirmation resolves it, and escalating past this step spends user effort
on something already recoverable; (b) ask the user to paste the activation
block into the file themselves; (c) if the user's environment provides its
own temporary-authorization mechanism (a marker file, an environment
variable, or similar), ask the user to authorize the edit through that
mechanism and revoke it afterwards; (d) where the platform supports
unguarded project-level instruction files, add the activation instruction
there instead. Branch on whether the denial is retryable-with-consent or
categorical: an interactive permission prompt is cleared by asking, a hook
or file-protection rule is not, and collapsing both into "hand the task
back to the user" is safe but consistently over-escalates. Never assume
unrestricted edit access to shared or governance-tracked config — many
setups gate exactly those files.

**The same shape applies to a blocked hook installation.** The hook
option from the same paragraph writes to the harness's settings file or
hooks directory, and those are gated at least as often as the config
file. A denied hook install follows the rules above unchanged: never
retry the same install through a different tool or path, never bypass,
surface the refusal, then fall back lightest first — ask for a retry if
the denial is retryable-with-consent; ask the user to add the hook entry
themselves (hand them the exact JSON or script); use a temporary
authorization mechanism if one exists; and if the hook cannot be
installed at all, fall back to the declarative config instruction, saying
plainly that the enforced tier is not in place and the probabilistic one
is. A hook that silently failed to install is worse than none: the user
believes the strongest tier is active.

### A third denial class: an automated content classifier

The fallbacks above branch on two kinds of denial: a hard guard (a
governance hook, a file-protection rule) and an interactive permission
prompt that one user approval clears. There is a third, and it behaves like
neither.

An automated content classifier can refuse the edit on the grounds that the
activation block is **third-party instruction text being written into a
user-scope config**, where it will steer every future session. Reported in
the field as `Instruction Poisoning`, on a fresh install into a global
Claude Code skills directory.

What follows from that:

- **The refusal is a valid outcome, and the user sees it first.** Stop.
  Quote the refusal and what it objected to, and explain the trade-off it
  points at: the block is always-on third-party instruction text in a
  global config, steering every future session, which is a real cost the
  user is entitled to weigh. Only then offer options.
- **Rewording does not help.** The objection is to provenance and always-on
  scope, not to the phrasing, so there is no lighter version to retry.
  Shortening the block and trying again is the obvious move; it fails, and
  it is a rewrite aimed at the classifier rather than at the user — don't
  make it.
- **The only option (Augury fork).** Accept the refusal, leave activation
  opt-in, and say so plainly. Never suggest a permission-mode change or any
  other way around the classifier.
- **The objection follows the content, not the file.** A documentation
  page that quotes the activation text can be refused on the same grounds
  as the config edit. Report the setup as incomplete while either the
  config or its write-up has not landed, and do not reword either to get
  past the classifier.

**The ladder is not monotonic.** Tier 4 — the session-start hook, which this
file elsewhere calls the only enforced option — can be blocked *harder* than
tier 3, and for an unrelated reason. In the same install the hook was
refused as `Auto-Mode Bypass`: configuring a hook changes what the harness
executes automatically, which that permission mode withholds categorically
rather than on content. Writing the hook *script file* alone, wired to
nothing, was refused on the same grounds. So tiers 3 and 4 failed for
different reasons that happened to coincide, and the tier presented as the
robust fallback for a blocked config edit was the one held furthest out of
reach. Do not present the ladder to the user as "if 3 fails, 4 will work".

### A mode that forbids every write by design

Some modes — Claude Code's plan mode, any propose-only mode — permit
every read and refuse every write for the whole turn. That is not a
denied write: a retry is a wasted call, the log is not unwritable, and a
denial report is the wrong report when the environment is telling the
truth. A gatekeeper and a mode are different failures — a gatekeeper's
denial is reported to the user and its observations carried in
report-back or handoff mode (SKILL.md, How to Log); a mode is accepted
and its writes deferred.

1. Run the read steps normally: the scan, the review-date check, the
   per-skill lookup. A read-only session is fully observant. Run them
   in a command of their own, without the scan's `checkpoints.log`
   append: a mode can refuse a command that mixes a read with a write as
   a whole, and the refusal takes the reads with it. The scan script
   writes that line, so a mode that refuses it gets the flat reads
   (`observation-log.md`, "A refused snippet is a degraded path, never a
   skipped step").
2. Defer, never skip, the writes: storage creation, `.id-floor`,
   archival, the `checkpoints.log` line. Ids are resolved at flush
   time, never now.
3. Queue pending observations where the turn cannot eat them: a
   numbered list in the plan or handover text itself. A rejected plan
   still leaves its text in the transcript; working memory does not
   survive the turn.
4. Flush in the first write-capable turn, before other work, running
   the id snippet (or the helper) before each write. End the deferred
   `checkpoints.log` line with `(deferred from <mode>)`, so a reader of
   the file dates the protocol to the session start, not to the flush.

Tell the two apart by the refusal's own words: a mode names itself and
refuses every write, a gatekeeper names a classifier or a rule and
refuses one call. Observed in one harness: plan mode refused an append
to an existing `checkpoints.log` as well as a create. Defer both either
way, so the flush turn is the only one that writes.

### A delegated setup step is not done until you have observed it

Every fallback above ends by handing something to the user: paste this
block, add this hook entry, authorize this edit. The instruction is where
the hand-off ends today, and that is one step short.

**So pair every handoff with a check you run yourself**, in the same
session, and report the result:

- `grep` the config file for the skill name;
- compare the file's mtime against the moment you handed the block over;
- for a hook, confirm the harness lists it, not merely that the file exists.

"The user says they pasted it" is a claim about the world. An install can
check it, cheaply, and should before declaring itself configured.

**The principle.** When a setup step is delegated to a human because the
agent is not permitted to perform it, the delegation is not complete when
the instructions are handed over — it is complete when the agent has
independently observed the resulting state. An unverified handoff and a
silently skipped step produce the same artefact: a setup both parties
believe is done.

### Paths handed across a boundary are resolved from the far side

A path the agent has verified from inside its own execution context proves
nothing about the same string in the user's hands. Three boundaries, each
observed in one install. **Packaging:** inside a Store-packaged (MSIX)
desktop app on Windows, `%APPDATA%\<App>` is transparently redirected to
`%LOCALAPPDATA%\Packages\<App>_<id>\LocalCache\Roaming\<App>`, so every
check the agent runs — bash, PowerShell, Python — finds the config file
there, and a script the user starts from Explorer, outside the package,
finds nothing; processes started from the app's own terminal die with the
app, so "run it after quitting the app" cannot be done from inside either.
**Sync redirection:** `%USERPROFILE%\Documents` is not the Documents
folder Explorer opens once a cloud drive has redirected the known folder;
resolve user folders with the shell's known-folder lookup
(`[Environment]::GetFolderPath('MyDocuments')`), never by composing them
from the profile path. **Shell dialect:** a Git Bash drive path
(`/c/dev/<repo>`) passes every pre-flight in Git Bash and fails in the
PowerShell the user opens (`fatal: cannot change to '/c/dev/<repo>'`);
`"C:/dev/<repo>"` works in PowerShell, cmd and Git Bash alike. The same
shape at the plugin boundary: `python plugins/<p>/scripts/x.py` and "show
`docs/<file>.md` from this repo" are correct in the author's checkout and
absent for every consumer, because installation copies only the plugin's
own directory. So: on a packaged host keep the workspace and any hook
script outside `AppData`; resolve every path handed to the user — hook
script, config, workspace — once from the user's side of the boundary
before handing it over; and make that resolution a step bound to the
hand-off, not a principle consulted while authoring.

**An install is a path handed across the same boundary.** A tool installed
for the user from the agent's shell inside a packaged app — a global npm
install, a per-user pip install, anything that writes under a redirected
`AppData` path — lands in the package's `LocalCache`, and every check the
agent runs from inside finds it where it was expected. Install from the
user's side (a command the person runs in their own terminal), or say in
the hand-off that the install is visible only inside the app. Not every
`AppData` path is redirected (`%TEMP%`, under `AppData\Local`, was not on
the host where this was observed), so "under AppData" is a first
approximation; a check from the far side settles a given path. **Process
state does not cross either.** The agent's shell can carry a process-level
execution policy and its own `PATH`, so a pre-flight run inside proves
nothing about the user's default policy (`Restricted` on a Windows client
unless changed) or about which commands their terminal finds: hand npm
tools over — `npm` and `npx` included — in their `.cmd` form, which the
policy does not govern, not as the `.ps1` shim it refuses. The app's own
terminal panel is a child of the packaged app and inherits its startup
environment, so it is not the user's side either.

**Resolving from the far side** (Augury fork): removed. Never start
anything outside the agent's sandbox; hand the step to the user to run
themselves.

### Windows hosts: what reaches the shell and the file

Two Windows transforms change content and report nothing.

**Backslashes through an agent's Bash tool.** Where the agent's Bash tool
runs Git for Windows' bash and hands it the whole command as one `-c`
argument, runs of backslashes have been observed to arrive halved
anywhere in the command — inside single quotes and quoted heredocs too —
except directly before a double quote; a single backslash arrives intact,
and the same command run from a file arrives exact. A body written through
that shell loses one backslash of every `\\`, and an inline snippet that
carries `\\` runs altered: the session-start scan's `suspect` program,
halved, is rejected by gawk, mawk and BWK awk, so the count reads 0 with
an error on stderr — why the scan ships as `scripts/session-start-scan.sh`,
whose one-line invocation carries no backslash. Test a host once with
`printf '%s\n' 'a\\b' | od -c`: two backslashes in the output mean the
transport is exact. Where it is not, write observation bodies and any
other content holding backslashes with the editing tool, and run a
snippet that carries `\\` from a file written with the editing tool
(`bash <file>`). Very long inline commands have also been reported cut on
the same transport, so keep inline commands short and put long content in
files.

**A byte order mark from Windows PowerShell.** PowerShell 5.1 writes UTF-8
with a BOM (`Set-Content -Encoding UTF8`, `Out-File -Encoding utf8`).
Every header reader tests line 1 for `---`, so an entry written that way
drops out of the session-start scan — `parsed` one below `files`, with no
notice while other headers parse — and the archival sweep never moves
it. Write entries with the editing tool or the helper script; in
PowerShell 7, `-Encoding utf8NoBOM`.

## Environment mappings

The procedures in this skill are written as capabilities. This table is
the only place product-specific names live; when a step says "present the
staged file" or "register the review", look up the current environment
here rather than guessing a tool name.

**Cowork execution mode — cloud vs local is a user setting.** Cowork can
run a session in two modes with materially different tool surfaces, and
the platform's default moved to cloud (so installs upgraded from earlier
versions may silently change mode — users experience that as the tool
losing abilities). The switch: Settings → Cowork → "Run new tasks in the
cloud" (plus a "Beta" button at session start). The tell: the
local-filesystem permission tool (`allow_cowork_file_delete`) present in
the tool surface ⇒ local session; absent ⇒ cloud. What differs:

| Capability | Local session | Cloud session |
|---|---|---|
| Scheduled-task editing | desktop scheduled-task tools | draft instructions only; the user applies them on the Scheduled tasks page |
| Deleting workspace files | permission gate (`allow_cowork_file_delete`) | no delete tool; rename-away only |
| Git on the mounted workspace | with the delete grant | never — use a sandbox clone |
| Locally-installed MCP servers | reachable | unreachable unless proxied via the desktop app |
| Working-file persistence | workspace folder persists | files not handed back are not kept |
| Runs while the computer is off | no | yes |

Steps that must edit scheduled tasks, delete workspace files (review
cleanup, keep-two pruning), or run git in the shared folder need a local
session — treat that as a one-line precondition on those steps, and when
reporting a mode-conditional limit, name the mode and the switch in the
same breath.

**A cloud session bridged to the user's machine writes observations
through the device shell, at write time.** In a bridged session the agent
runs in a container while the log lives on the device, reached through a
remote shell (`device_bash`) and a file-commit tool
(`device_commit_files`). An observation is created on the device by that
shell running the id snippet (SKILL.md, How to Log) — or
`scripts/new-observation.sh`, where the script is reachable on the device
— at the moment of writing, and its body goes into the created path
through the same shell with a quoted heredoc. It is never composed in the
container and committed afterwards. The failure shape: a committed file
carries an id derived from a listing taken earlier, and the prefix guard
never ran because it lives in the command the commit bypassed; the lagging
`.id-floor` appears first and the duplicate id at the next review.

**Where the scripts are not on the device.** In a bridged session the
installed skill tree, scripts included, can exist only in the cloud
container while the log lives on the device. First look for the review's
copy at `skill-observations/.bin/`: its `MANIFEST` records the installed
`version:` and a sha256 per script, refreshed by every review that confirms
an install (`weekly-review.md`, Step 1 gate). Hash the installed scripts on
your side and run the `.bin/` copy only when every digest matches the
manifest. Absent or mismatched, look for another copy in the workspace — a
staged copy under `skill-updates/*/task-observer/scripts/` — and compare its
digest (`md5sum` or `sha256sum`, run on both sides) with the installed
scripts BEFORE running it, and run it only on a match: a staged copy can
be an unreleased version that nothing refreshes on install. With no
matching copy, transfer the scripts to the device — a
permission-classifier denial of that transfer gets at most one identical
retry, told to the user (SKILL.md, How to Log) — or run the inline id
snippet through the device shell. Never a hand-derived id.

| Capability | Claude Cowork | Claude Code | Web chat / no filesystem |
|---|---|---|---|
| Persistent workspace | the shared folder | pinned absolute path (see activation block) | none — handoff-doc mode below |
| Live skill files | `.claude/skills/{skill}/` on a read-only mount (writes fail with EROFS) | `~/.claude/skills/{skill}/`, ordinary writable files — no guard | n/a |
| Present a staged file for install | the file-presentation tool (`present_files`) with its upload button | none — report the staged path and a change summary in chat | paste into the handoff doc |
| Scheduled review | the app's scheduled tasks (cloud-default: run remotely; legacy local mode: run on the user's machine, only while it is on — see the execution-mode note above) | cron / a harness `SessionStart` hook / an account-level scheduler that can reach the workspace — headless invocations carry the workspace grant (`--add-dir`) in the task definition | calendar reminder + manual trigger |
| Session-start hook | none | `SessionStart` hook (`hookSpecificOutput.additionalContext`) | none |
| Local-vs-cloud tell | `allow_cowork_file_delete` present ⇒ local session | n/a | n/a |

**The Claude Code column is the terminal CLI; the desktop app has a
scheduler of its own.** Claude Code run inside the desktop app exposes
the app's scheduled-task tools on its tool surface, and a review
registered there is local: it runs on the user's machine while the app
is open and catches up at the next launch (reported, not yet confirmed
here). Register it as any local scheduler: the workspace grant goes into
the task definition, and the first "Run now" of the task itself is the
grant check (the access-grant paragraph below). The tell is a
scheduled-task tool on the tool surface; absent, the CLI column applies.

Grow the table when a new environment appears; do not scatter its tool
names through the procedure files.

**A scheduled-task prompt's first listed action is its first tool call.**
The Session Start Protocol wants the workspace probe (step 1) before any
skill invocation, and a scheduled prompt that opens with "invoke, in this
order: …" makes the first invocation the first call. A rule that wants something before the
first listed action has to appear in that list, not in the text the list
loads: write the activation pointer in every scheduled prompt as "probe the
workspace folder (one `ls` of its absolute path), then invoke <skills>",
and keep the probe's procedure and failure handling in the skill.

**The second listed action leaves a started marker.** After the probe,
every scheduled review prompt writes the date and time to
`skill-observations/review-started.txt`, and the review's Step 7
overwrites it with `completed <date>`:

```bash
printf '%s\n' "$(date '+%F %H:%M')" > "[ABSOLUTE PATH]/skill-observations/review-started.txt"
```

A scheduler's "succeeded" says the job launched; the marker says how far
it got. At the next session start, a marker that does not read
`completed` is a run that died mid-way, and a scheduler last run later
than `last-review-date.txt` with the marker untouched is a run that fired
and never reached its second action — two different defects, and neither
is a review that happened. The marker is judged by its content, never by
its mtime: Step 7 writes it after the date file, so a completed run's
marker is always the newer of the two.

**The access grant is part of the task definition, not of the path.** A
scheduled session runs in the sandbox the scheduler creates, and that
sandbox can deny a path that exists and is online — through every
interface, a junction inside the allowed area included, because the
sandbox canonicalises the target — and the run ends with the offline
policy's one-line skip: a config gap wearing the costume of a transient
outage. Registering a scheduled
review therefore includes whatever allowed-directory configuration the
harness needs for the workspace root — for Claude Code headless,
`--add-dir "[ABSOLUTE PATH]"` on the invocation, or the equivalent
permissions rule — and the setup is not done until one firing has
reached the workspace **from inside the task's own execution context**:
an interactive test proves nothing about the sandbox the scheduler
creates, and a task the scheduler lists is not a run that succeeded.
(The run-time skip note: `references/weekly-review.md`, offline-workspace
policy.)

**Managed skills directories (dotfile managers, sync tools).** Where the
skills directory is generated by chezmoi, GNU Stow, yadm, a symlinked
dotfiles repo or any sync tool, the live path is not the source of truth:
copying a staged update onto it succeeds, looks installed, and is
silently reverted at the next `apply` — "I installed it" and "it is still
in effect" become indistinguishable. Detect the manager at install (a
symlink into a dotfiles checkout, a `.chezmoi*` marker, the directory
listed in a Stow package) and record it in the activation config. Then
install into the manager's SOURCE, not the live path, and let the manager
apply it; or exclude the skills directory from management. "Always start
from the live file" (`references/skill-authoring.md`) still holds for
READING — the live file is what the loader loads — but the WRITE-BACK
path is the manager's source. The weekly review's staged-work
reconciliation (`diff -rq` staged vs live) is the check that catches a
reverted install: run it at the session after any install on a managed
directory. Where the entry is a symlink, resolve it before staging as
well (`skill-authoring.md`, editing rule 1): a link copied as a link
makes the staged path the live one.

## Git as an optional staging medium

Staging a skill update as a branch or commit the user merges gives the
same guarantee as the `skill-updates/` directory — nothing goes live
without a user action — plus diffs, history and rollback, and it suits a
multi-device setup that syncs skills through a private repository. It is
an option, not a change to the default: the review still writes the
staging manifest, still never edits the live install, and still presents
the change for a decision. Treat the version-control commands as a
mutation surface for the observation log (see
`references/observation-log.md`).

## Claude Code Projects

*Based on the platform's documentation, not yet field-tested — reports
welcome.* Re-verify each claim below against the current Claude Code
documentation before relying on it.

Claude Code Projects run work as parallel cloud "threads" under one
coordinating conversation. Per the documentation, a thread picks up
nothing from the Claude Code setup on the user's own machine: skills load
only from `.claude/skills/` in a repository added to the project, or from
skills enabled on the account; CLAUDE.md loads only from the project's
repositories; standing rules go in the project instructions, which every
new thread receives. Each thread works in its own clone on its own
branch, and the coordinator sees what threads report back, not every
step they take. That breaks three install assumptions at once:

- **Install.** A user-scope install is invisible to threads. Commit the
  skill — with its `references/` and `scripts/` — to a repository added
  to the project, or enable it as an account skill.
- **Activation.** A user-level activation line never loads. Put the
  activation block's instruction in the project instructions (preferred:
  it reaches every thread and the coordinator, and survives multi-repo
  projects where per-repo settings are not read) or in the repository's
  CLAUDE.md.
- **Log route.** No pinned shared path exists: the only writes that
  outlive a thread are a pushed branch or PR, a project Library file, or
  project memory. So either (a) threads carry every observation **in
  full** in their final report to the coordinator, and the coordinator
  or a scheduled project routine writes them to the real log. (Augury
  fork: the observations-repository PR route is disabled.)
  Per-observation files still prevent write collisions between parallel
  threads; the problem here is survival, not concurrency.

The Session Start guard warns and re-anchors when the resolved workspace
is ephemeral; in a thread there is nothing stable to re-anchor on, so the
branch is **report-back mode** ("Anchoring the workspace" above): the
durable route is the channel the surface guarantees survives — the
worker's report to its coordinator, a merged commit — not a location the
worker can reach and hope outlives it.

The characteristic failure is silent: a never-activated install and
observations stranded on an unmerged branch both produce no error, so
issue reports will under-represent it. The external diagnostic: **no
observations arriving at the coordinator after several threads of real
work means the skill is not activated, or the log route is not wired.**
Check the project instructions for the activation line and the added
repository for the skill directory, then report what was found — that
report is the field test this section is waiting for.

## Sessions whose output channel is owned by a caller

A subagent, a non-interactive run, a tool-shaped invocation with a
declared result schema: the session has a filesystem and no reader.
Detect it from the session's own contract — a single required result
call, a ban on free-text answers — never from a flag. Every step that
needs a free-text reply to a person has no recipient there: the review
offer (Session Start step 3), the activation suggestion (step 4), the
end-of-session summary (Surfacing Protocol) and the per-task summary line
in the activation block. Skip them silently, and never count an
unanswerable offer as surfaced. Never run the review unprompted either:
"autonomous" in step 3 means a scheduled run whose prompt asks for the
review, not a worker whose caller asked for something else.

What happens to logging depends on who the caller is. Where the caller
is itself a session running this protocol, the activation block's
subagent rule applies: the worker writes nothing, and a finding worth
logging goes into its result, in whatever field the schema leaves for
notes, for the controller to write. Where the caller runs no protocol —
a script, a CI job, another tool — logging stays fully in force, because
the write is the only report there is: anything that would have been
surfaced becomes an observation file naming the condition that fired, so
the next interactive session inherits it; name the caller in
`session_context`, and log only what the caller cannot see. Handoff-doc
mode is the opposite case (a reader, nowhere to write); report-back mode
("Anchoring the workspace") is the case with neither a stable path nor a
reader.

## First-run backfill

The skill is least valuable at the moment it is adopted: the log is empty,
a review correctly reports nothing to do, and the largest pile of
uncaptured insight — the project's own history — is never touched because
no procedure says to touch it. Session Start step 7 offers a one-off
mining pass over handover, architecture and decision docs, commit history
since the last release, test and verification scripts (which encode
hard-won discipline densely), and existing agent-instruction files, which
are largely a record of corrections the user already had to make.
Backfilled entries cite the durable artefact, and one batched write
satisfies the same-turn rule; the pass runs once, and the scheduled
review takes over afterwards.

**The same pass, scoped to one named session.** "Review today's session
retrospectively" and "the one where we did X" name a finished session,
not the project's history, and the session that hears the request holds
no transcript of it. That is not a live observation and not a first-run
backfill; it is the backfill pass with its sources bounded to one
timeframe. The sources are whatever durable per-session record the
project keeps — the kinds above are examples, not the list: commit
history and diffs within the session's window, a memory or notes system
the project already maintains across sessions (a directory of dated
files indexed apart from the instructions file is common), handover docs
written at its end. Each is cited in `session_context` exactly as a
backfilled entry cites CLAUDE.md — the artefact and the section, never a
session id, because there is no session to point at. Say which sources
were read and which were unavailable, so a later reader can tell a thin
session from a thin record.

## Storage regimes

Persistence is one axis; the *price* of a write is another. Three
regimes:

1. **Local filesystem** — appends are free. Write the checkpoint markers
   exactly as SKILL.md specifies; they are the only evidence the check
   happened.
2. **Shared or hosted document store** (the workspace resolves to a
   hosted knowledge base or project, where each "file" is a document and
   every write is a mutation that invalidates cached context for every
   other session in that workspace). Keep the mandatory *check* at every
   checkpoint but suppress *empty* markers: enforce the check by hanging
   it on writes that were going to happen anyway — deliverable events and
   task completions — and write only when there is an observation to
   record. An enforcement mechanism has to be priced against its
   environment; a rule that makes writes mandatory-even-when-empty is
   only free where writes are free.
3. **No persistence** — handoff-doc mode, below.

## Bundle manifest

This skill consists of `SKILL.md`, the reference files it lists
(`weekly-review.md`, `skill-authoring.md`, `environments.md`,
`observation-log.md`, `signals.md`, `migration.md`,
`starter-principles.md`) and `scripts/migrate-log.py`,
`scripts/new-observation.sh`, `scripts/session-start-scan.sh`,
`scripts/skill-load.sh` and `scripts/validate-skill-bundle.py`. If any is
missing, SKILL.md "Reference files" applies (the rule lives there because
this file can be the missing one). `references/ceilings.txt` travels with
them: it holds the per-file line ceilings the validator reads.

## Compaction behaviour

**A compaction is a session start.** So is a handoff-doc resume and a
re-invoked scheduled run. Each is a new context with old work in it, which
is exactly the condition the Session Start Protocol exists for, and on the
first turn after any of them the protocol runs again in full — storage
probe, frontmatter scan, review trigger, and the per-skill grep for every
skill the task in front of you needs. Where the CLAUDE.md structural
trigger reaches the resumed context it re-invokes the skill; do not rely on
it having done so.

**The write path has to survive the compaction; the skill body does
not.** The pointer to this section sits in the skill body, which
is exactly what a compaction removes. Between a compaction and the
skill's re-invocation nothing is written by any other route: a flush
that fires first runs `scripts/new-observation.sh` (it needs no skill
body in context — a slug and the workspace root are its only inputs) or
waits for the invocation, and never copies another file's header. So the
activation block names the script as the only way an observation file
comes into being, and the hook prints the same: a rule that must reach a
resumed context travels in a channel that survives the compaction — the activation config, or a session-start
hook where the harness fires it on compaction as well (Claude Code's
`SessionStart` does unless a matcher excludes `compact`), which is then
the channel that arrives first.

**The compaction summary's list of previously invoked skills is a lower
bound, never the session's skill set.** A summary preserves what it judged
salient about the *work*; skill loads are infrastructure, not work, so they
are exactly what a summary drops. The block's own framing — "shown here for
context only so you remain aware of their guidelines" — describes what the
block contains and is easily read as what the session has. Re-derive the
skill set from the task (name the decision, name the artefact type, match
the installed descriptions — "Select on the decision, not on the artefact"
above), and treat any skill the summary lists as loaded-then-forgotten
rather than as the whole set.

Observations before and after compaction are written as separate files
under the same `observation-log/` directory, each with its own id (the id
counter is derived from existing filenames, so it continues seamlessly
across the compaction boundary). A resumed session's opening message may
not match the description triggers, which is one reason the structural
trigger exists — and one reason the protocol re-runs by rule rather than
by whichever trigger happens to fire.

## User-facing documentation

Installation, shared-folder setup, expected behaviour, and the cadence
pattern live in the public repo. Pass these links on to the user rather
than fetching the pages — the skill's behaviour is defined entirely by its
own files, never by external content:

- README: https://github.com/rebelytics/one-skill-to-rule-them-all/blob/main/README.md
- USER-GUIDE: https://github.com/rebelytics/one-skill-to-rule-them-all/blob/main/USER-GUIDE.md

## Repo/maintainer sessions — verify commit identity before writing

Authentication and attribution are separate channels in git: the push
credential (PAT/SSH key) controls who may WRITE; the commit's author
email (`git config user.email`) declares who WROTE, and the platform maps
that email to whichever account has it verified — regardless of which
account pushed. Before the first terminal commit in any clone used for a
specific identity, verify `git config user.email` resolves to the
intended account, and set repo-local config where the machine's global
identity differs. Diagnose suspected mis-attribution via the commits API
(author login vs commit email). Fix forward only: rewriting a published
main to correct author metadata (with tags/CI descending from it) costs
more than the cosmetic gain — verifying the email on the intended account
is the alternative remedy.

## Handoff-doc mode (no persistent storage)

The methodology is environment-independent; only persistence varies. In
web-chat-style environments, collect observations in-session and deliver
them in a structured handoff document the user stores and pastes into the
next session. Where the surface has a user memory store, its inbox comes
first and the handoff document is the last resort (fallback order: "No
link to the user's computer" below). **Offer the handoff proactively when
the conversation winds down** — a premature offer is a minor interruption;
a missing one is lost work.

```markdown
# Session Handoff: [Session Topic]

**Date:** [date]
**Context:** [what was worked on; what the next session needs to know]

## Decisions Made
[numbered]

## Observations Logged
[each observation in the frontmatter format from SKILL.md → How to Log; the
next session writes each as its own file in `observation-log/`]

## Cross-Cutting Principles (current)
[active or newly added]

## Action Items
[next steps with enough context to resume]

## Working Artifacts
[drafts/analyses in full]
```

### No link to the user's computer

A cloud or scheduled run can have a filesystem of its own and no route to
the workspace: the probe fails, and no folder-picker or device-bridge tool
is on the tool surface — absent, not denied. Check once with a tool
search; if none appears, the request-and-retry branch of Session Start
step 1 has nothing to call, so skip it. The same file is the landing
place when the folder request was refused and the one permitted retry was
refused too.

**Fallback order when the log is unreachable.** (a) The observation log.
(b) A user-scoped store this surface can write and every other surface
the user works from can read — a user memory store is the common case —
in one dedicated inbox file (e.g. `/areas/observation-inbox.md`), never
the preferences file or any other file injected into every session, where
an entry reads as a standing rule. (c) The handoff file below, the last
resort: on a phone it is lost unless the user carries it by hand. The
failure shape: a fallback only the desktop can reach turns every
observation from a phone or an unlinked cloud session into chat text
nobody files.

**A memory write needs the user's explicit ask, and skill text is not
one.** A memory system writes only on an explicit user request, so ask in
one line — "The observation log isn't reachable — park this in memory for
the next review?" — and a yes is the request; a no, or an unattended run
with nobody to ask, goes to (c). One entry per observation: date, target
skill(s), Issue / Improvement / Principle in compressed form, session
context — no id, which only a session with the log can derive. Report the
condition as with (c), naming the inbox file instead of a path. The inbox
holds observations only: tasks and other loose ends go to their own
channel, because the review turns every inbox entry into an observation
file (`weekly-review.md`, Step 1, "Drain the observation inbox").

Where (b) is unavailable or declined, write a handoff file to the
session's output location (wherever its deliverables go), never only into
the final message:

- every observation in full frontmatter, with `id: TBD` — the id is
  derived from a log this session cannot read;
- a named list of the checks not performed — id derivation, restatement
  check against existing entries, sibling check — which the next session
  with the log runs before writing any of the files;
- in the run's report, the condition by name: "this session has no link
  to your computer; observations are in a handoff file", with its path.

Name the condition, not its permanence. A session can become linked
mid-run (a link notice, after which the bridge tools appear), so re-check
the tool surface before the final flush rather than asserting the link
cannot be made. The session that picks the file up handles it as below,
completing the listed checks before it writes any observation file.

## Handoff-doc analysis (when one arrives)

1. Log all explicitly stated observations first, unfiltered.
2. Then systematically read every section asking what skill gaps or
   candidates are *implied* but unstated — handoff docs carry signal beyond
   what was captured live.
3. Pay special attention to action items (each may imply a missing skill),
   open questions (ambiguity signals a decision-framework gap), the
   work-completed narrative (patterns may reveal meta-skills), and session
   notes.
4. Attribute derived observations as coming from handoff-doc analysis, not
   the original session.
