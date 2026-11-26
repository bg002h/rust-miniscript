# Bug hunt 2026-11-25 (Fable): rust-miniscript

Report-only review of `rust-miniscript` at `b68bf13`, part of a constellation-wide hunt across all 11 bg002h repositories. Nothing in this repository was changed by the hunt; this file is the only addition.

**How it was done.** Eight Fable reviewers each audited an area (one of them the boundaries between repositories), reproducing what they could with locally built binaries or scratch tests that were not committed. Three further Fable verifiers then tried to disprove every High and Medium finding (and several Low ones) and re-graded them. Grades below are the verifiers' final grades where they ran.

**Grades.** Critical: can lose funds or leak a seed in ordinary use. High: wrong wallet, addresses or keys, or secret exposure, plausible in normal use, or recovery broken. Medium: needs an unusual but realistic input, or fails in a way a careful user might not notice. Low: fails closed, hardening, or a small exposure. Info: context, not a bug.

**Categories.** security (secret or key exposure, injection, integrity of what is trusted), funds (wrong address, amount, descriptor or recovery that could lose or strand coins), other.

**Status.** "Confirmed (reproduced)" means a command or test showed it. "Confirmed by reading" means the code path was traced but not executed. "Reviewer-confirmed, not re-verified" means the first reviewer reproduced or read it and no verifier re-checked it.

**Totals:** 2 Info. No Critical findings.

## Summary

| ID | Grade | Category | Status | Finding |
|---|---|---|---|---|
| M1 | Info | other | Confirmed (reproduced) | No repository uses this fork; every workspace patches to upstream rust-bitcoin/rust-miniscript `ff4732e5` |
| M2 | Info | other | Confirmed by reading | What can be said about `ff4732e5` from the repositories alone |

IDs: `md#`, `mk#`, `ms#`, `toolkit#`, `G`, `refugium#`, `wallet#`, `E`, `T`, `SH#`, `M` are per-repository findings; `X` findings are at a boundary between repositories and appear in every report they touch.

## Findings

## M1: No repository uses this fork; every workspace patches to upstream rust-bitcoin/rust-miniscript `ff4732e5`

**Grade:** Info · **Category:** other · **Status:** Confirmed (reproduced)

**Reviewer write-up** (source: `miniscript`)

### M1. No repository uses `bg002h/rust-miniscript`; the local clone is unmodified upstream from March 2024

- **Severity:** Low (informational)
- **Status:** CONFIRMED
- **Repos:** rust-miniscript (and, as non-users, all ten others)
- **Evidence:**
  - `grep -rln "bg002h/rust-miniscript" --include=Cargo.toml --include=Cargo.lock --include='*.toml' --include='*.yml' /home/user`
    (vendor directories excluded) finds nothing. Every workspace that overrides miniscript uses
    `[patch.crates-io] miniscript = { git = "https://github.com/rust-bitcoin/rust-miniscript", rev = "ff4732e5f75aa555682343cb180fa72ee3e8e9d5" }`:
    `mnemonic-toolkit/Cargo.toml:39`, `mnemonic-toolkit/fuzz/Cargo.toml:28`, `descriptor-mnemonic/Cargo.toml:55`,
    `descriptor-mnemonic/fuzz/Cargo.toml`, `mnemonic-engrave/Cargo.toml:25`, `refugium-wallet/Cargo.toml:54`, and
    the refugium and gui lockfiles resolve the same rev. `mnemonic-key` resolves crates.io `miniscript 13.0.0`
    unpatched (its md-codec is 0.42.0, which does not need the patch). The "fork" in the toolkit's
    `miniscript-fork-tripwire` workflow name means this upstream git pin, not Brian's fork.
  - `/home/user/rust-miniscript`: remote `github.com/bg002h/rust-miniscript`; HEAD `b68bf13` "Merge
    rust-bitcoin/rust-miniscript#654" (Andrew Poelstra, 2024-03-08); `Cargo.toml` version `11.0.0`. Branches
    `master` and `claude/project-thread-ly86x4` are the same commit. In the 234 commits the shallow clone holds,
    `git log --author='Goss\|bg002h\|Claude'` is empty: no change of Brian's. It is two major versions behind the
    13.0.0-based rev the suite builds on and does not hold `ff4732e` (`git cat-file -t ff4732e...` fails), so the
    two cannot be diffed locally.
- **Impact:** none on any build. The fork is a stale mirror; a reader could take it for the source the suite uses.
- **Fix direction:** archive the fork or say in its README that the suite pins `rust-bitcoin/rust-miniscript@ff4732e5`.

## M2: What can be said about `ff4732e5` from the repositories alone

**Grade:** Info · **Category:** other · **Status:** Confirmed by reading

**Reviewer write-up** (source: `miniscript`)

### M2. What can be said about `ff4732e5` from the repositories alone

- **Severity:** Low (informational)
- **Status:** CONFIRMED for the facts below; "bugs fixed upstream after ff4732e" cannot be determined offline
  (the rev is in no local git history; only its vendored tree is here)
- **Repos:** descriptor-mnemonic, mnemonic-toolkit, mnemonic-refugium, refugium-wallet, mnemonic-engrave, mnemonic-gui, mnemonic-key
- **Evidence:**
  - The vendored trees are the same code everywhere it is vendored: `diff -rq mnemonic-toolkit/vendor/miniscript
    mnemonic-refugium/vendor/miniscript` and against `descriptor-mnemonic/vendor/miniscript` print nothing. The
    crate reports `13.0.0` (`vendor/miniscript/Cargo.toml:16`) although it is a post-13.0.0 master rev.
  - Known and written down by the owner (`descriptor-mnemonic/Cargo.toml:22-52`): at this rev `sortedmulti` is a
    miniscript `Terminal` (PR 915) that upstream parses anywhere with no depth guard, which is more permissive than
    BIP-388; md-codec holds the BIP rule itself (`to_miniscript.rs`). A consumer that parses descriptor text with
    miniscript directly, without md-codec's check, inherits the permissive parse; refugium-view depends on
    `miniscript = "13.0.0"` directly (`refugium-wallet/crates/refugium-view/Cargo.toml:30`, for fee-rate
    estimation) and the toolkit depends on it directly (`crates/mnemonic-toolkit/Cargo.toml:110`). I did not audit
    those call sites (the per-repo reviewers cover them).
  - The patch applies only while each `Cargo.lock` selects `miniscript 13.0.0`. crates.io has 13.1.0 and 13.2.0;
    a `cargo update -p miniscript` moves the lock off the patch and md-codec >= 0.47 then fails to compile
    (`mnemonic-engrave/Cargo.toml:16-23` records this). That fails closed.
  - mnemonic-key's unpatched crates.io 13.0.0 has the taptree `Display` flattening that PR 953 fixed (per
    `descriptor-mnemonic/Cargo.toml:38-43`). mk-cli takes only identity data from md-codec 0.42.0 for
    `--from-md1`; I found no place where it prints a taproot descriptor, so this is not reached today.
- **Fix direction:** none required now; when a crates.io release carries PRs 910, 911, 915 and 929, drop the patch
  in all seven places in one change (the tripwire workflow watches for it).

---

_Generated by [Claude Code](https://claude.ai/code) on 2026-11-25. Findings were not fixed; each fix needs its own review under this repository's gates._

---

## Addendum: the 2026-11-25 bug hunt compared with the 2026-11-19 bug hunt

_Written 2026-11-26 for the coordinator that holds the 2026-11-19 fix queue. Read-only comparison: nothing here was merged or pushed to a default branch._

_This is the per-repo copy. The full addendum, with every repo's 11-19 table, is appended to `BugHunt-2026-11-25-fable.md` in mnemonic-refugium (report PR #247, branch `claude/project-thread-ly86x4`)._

### Why this exists, and how to use it

Two hunts ran a week apart:

- **2026-11-19** (OpenAI Codex, gpt-6). It covered 7 repos: descriptor-mnemonic (md), mnemonic-engrave (me), mnemonic-key (mk), mnemonic-secret (ms), mnemonic-toolkit (tk), mnemonic-transaction (mt) and seedhammer (sh). The reports are `design/agent-reports/bug-hunt-2026-11-19.md` on each default branch. The keys are `bh-<repo>-<id>`.
- **2026-11-25** (Fable). It covered all 11 repos. Each repo has a `BugHunt-2026-11-25-fable.md` report in an open, add-only PR on branch `claude/project-thread-ly86x4`. The fixes are in a second set of open PRs on branch `claude/project-thread-utm1aa`, one per repo.

Before you run any queued 11-19 item:

1. Look the key up in the per-repo tables below. Each table says where the fix is (the merged PR on the default branch), whether the 11-25 hunt found that bug again or a related one, and what is still left to do.
2. If a row's last column says "nothing", the item is done. Don't redo it.
3. Every 11-25 fix branch starts from the current default branch (checked 2026-11-26: 0 commits behind in all 10 repos). So every 11-19 fix merged so far is already underneath the 11-25 fix PRs, and the 11-25 PRs do not undo any of them.

#### The 11-25 PRs

| Repo | Report PR (11-25 findings) | Fix PR (11-25 fixes) |
|---|---|---|
| descriptor-mnemonic | #75 | #76 |
| mnemonic-engrave | #67 | #68 |
| mnemonic-key | #42 | #43 |
| mnemonic-secret | #47 | #48 |
| mnemonic-toolkit | #226 | #227 |
| mnemonic-transaction | #31 | #32 |
| seedhammer | #28 | #29 |
| mnemonic-refugium | #247 | #248 |
| mnemonic-gui | #57 | #58 |
| refugium-wallet | #228 | #229 |
| rust-miniscript | #1 | none (it has only Info findings) |

All of these PRs were open and unmerged when this addendum was written.

#### How the 11-19 fix status was worked out (and its limits)

- The starting point is the closure record `coordinator/agent-reports/recon-bughunt-1119-closure.md` in mnemonic-refugium (2026-11-24). It maps 125 Critical/Important keys to 58 FIX rows and found 27 of them open and 6 unrouted.
- After that record was written, more PRs merged on 11-24 and 11-25. I matched them to keys by the key names their commit titles cite: md#70, #71, #74; mk#37, #38, #39; ms#41, #42; tk#211, #212, #214, #220; mt#26, #27, #29; me#64; sh#25. With those, **every 11-19 Critical/Important key that had a FIX row is now merged.** What is still open are keys that were graded Minor and filed as FOLLOWUPS lines.
- "Merged" means the PR that names the key is on the default branch. I did not re-run each key's reproduction against the merged code.
- The closure record lists ms "I5" as the stdin-alias bug, but that bug is ms M5 in the source report. Source I5 is "verify calls a Japanese Mnem card v0.1 entr". Both are open FOLLOWUPS lines (see the ms table).

### Summary of the overlap

The 11-25 hunt has 50 distinct findings. Compared with the 11-19 hunt:

- **6 are the same bug as an 11-19 finding.** In every case the 11-19 fix merged, but only in one place: one repo, or Rust and not the Go port. The 11-25 hunt found the same bug somewhere else.
- **5 are related, and I'm not sure whether they count as the same bug** (marked UNCERTAIN).
- **About 39 are new.** That includes all 16 findings from the four repos the 11-19 hunt did not cover: mnemonic-gui, mnemonic-refugium, refugium-wallet and rust-miniscript.
- **None of the 11-19 bugs that are still open was found again on 11-25.**

| 11-25 finding | 11-19 key | Match | 11-19 fix (merged) | Where the bug was still present on 11-25 | 11-25 fix PR (open) |
|---|---|---|---|---|---|
| X7 (md-codec pin before 0.48.6 can mint a card the decoder refuses) | bh-md-C2 | SAME | md#59, released in md-codec 0.48.6/0.48.7 | consumers' md-codec pins (engrave, refugium) | engrave#68 `fc5f7c5`, refugium#248 `f8a0d9b`: both pin descriptor-mnemonic `c8cde23` |
| E5 (engrave's mt-codec pin) | bh-mt-I18 | SAME | mt#21 | engrave's mt-codec pin `eec7910b` | engrave#68 `67c3f1c`: pins mt `9b094e0b` |
| X4 (WIF slot writes a depth-0, zero-chain-code xpub onto cards) | bh-mk-C1 | SAME | mk#30 (mk-cli; not released yet) | toolkit mints these cards; md and the Go port derive from them; GUI and installer pin mk-cli 0.17.0 | tk#227 `59057e10` (refuses a WIF slot in template mode), md#76 `59c158e` (md refuses it). Still to do: release mk-cli and move the GUI/installer pins (gui#58 records that no release exists yet) |
| SH#6 (Go: change list equals receive list) | bh-md-I6 | SAME, Go port | md#64 (Rust) | seedhammer `address/`, `gui/` | seedhammer#29 `c2326a3` |
| SH#4 (Go `mk.Encode` lacks the origin refusals) | bh-mk-I9 (same function as bh-sh-I10) | SAME, Go port | mk#29 (Rust, mk-codec 0.6.0); sh#13 fixed only the network check | seedhammer `mk/encode.go` | seedhammer#29 `19b6075` |
| md#3 (placeholder with no wildcard minted as `/*`) | bh-md-C1 | SAME root (same lexer line, `parse/template.rs:345`) | md#60 fixed only the `md decompose` route | md template lexer (encode/address/descriptor) | md#76 `59c158e` and `6604c88` (bare `@N` refused at mint) |
| toolkit#3 (Word Card parity budget below what is stated) | bh-tk-I6 | UNCERTAIN: same weakness (one wrong word loses the card), different positions (header and checkpoint words today; the stop-sign word on 11-19) | tk#212 (stop-sign recovery) | header and checkpoint words | tk#227 `e6fd50c8`, `573dc14f`: the help and manual now state the budget the decoder really has. The decoder itself is unchanged |
| md#4 (`c:` over a non-key child mints a card md cannot restore) | bh-md-I5 | UNCERTAIN: same class (the encoder accepts what restore refuses); md#62 added the mint-time self-decode, but that check does not type-check | md#62 | md-codec mint check | md#76 `f8b4a67`, `ca48a05`, `5ad5a9b`, `fcefa83` (type rule at mint; address/descriptor refuse invalid miniscript unless `--allow-invalid-miniscript`) |
| SH#3 (Go `md.Decode` accepts four shapes Rust refuses) | bh-md-C2 | UNCERTAIN: one of the four shapes (an override keyed on baseline `@0`) is the card C2 used to mint | md#59 (Rust encoder) | Go decoder | seedhammer#29 `216a8af` |
| mk#2 (`mk verify --from-md1` passes a non-cosigner) | bh-mk-I2, I15 | RELATED: encode got the membership check, verify did not | mk#31 | `mk verify` | mk#43 `015d106` |
| toolkit#6 (`--account` >= 2^31 panics in six commands) | toolkit FOLLOWUP `xpub-search-account-window-2p31-panic` (listed as "previously filed" in the 11-19 tk report) | RELATED: same panic class; the earlier fix covered xpub-search only | tk#214 | bundle, verify-bundle, export-wallet, addresses, restore, convert | tk#227 `2d0057e8` |

Not an 11-19 match, but worth knowing:

- X6 (non-ASCII characters read as codex32 symbols by older ms/mk codec pins) repeats a bug from the **October 31** hunt that was fixed in the codecs. mk#43 `34c69b8` covers mk-cli's own md1 input. The consumer pins are tracked by the 11-25 X6/X8 entries.
- toolkit#7 (legacy `sh(sortedmulti)` defaults to BIP-48 `1'`) sits next to the 11-19 "previously filed" toolkit line `xpub-search-sh-wsh-default-origin-notice`, but it is a different wrapper and a different bug. It is fixed by tk#227 `6c99456c` plus the BIP-defaults follow-ups.
- engrave E3 (`--as md1` accepts a contradictory origin) is the md1 route of the ruling in `FIX_sec_canonical_keeps_xpub_depth` (me#54, 11-23). That ruling was not an 11-19 key. engrave#68 `094979e` fixes it.

### Work that the 11-25 PRs create for later

These items follow from the 11-25 fixes. Check the coordinator's queue against them so nothing is done twice:

- **md-codec 0.49.0 re-pins.** md#76 changes minting (explicit origins on keyed cards, the bare `@N` refusal, the type rule, thresh "any k of n"). After it merges, mnemonic-refugium, mnemonic-engrave (their md-codec pins must stay equal), mnemonic-toolkit, refugium-wallet and the seedhammer port each need a re-pin or a check. engrave#68 and refugium#248 pin `c8cde23` (0.48.7), not 0.49.0.
- **mk-cli release.** The bh-mk-C1 fix (mk#30) is still unreleased. gui#58 and the toolkit installer still pin `mk-cli-v0.17.0` (11-25 X4, G5).
- **seedhammer provenance pins.** seedhammer#29 `dde21fe` re-pins md, mk and mt to what the ports converge to (SH#5). Re-check them after md#76 merges.

### Repos the 11-19 hunt did not cover

mnemonic-gui, mnemonic-refugium, refugium-wallet and rust-miniscript have no 11-19 report, so their 11-25 findings are new by definition. Cross-repo entries that appear in these reports are covered above: X7 in refugium (= bh-md-C2, refugium#248 `f8a0d9b`), and X4 / G5 in gui (= bh-mk-C1; the mk-cli release is still owed).

