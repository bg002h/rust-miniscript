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
