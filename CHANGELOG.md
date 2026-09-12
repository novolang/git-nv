# Changelog

All notable changes to git-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `gitobj` — the four object types to and from bytes, ids under SHA-1 or
  SHA-256 with the algorithm carried on every id, names and messages as
  ranges into the caller's bytes, and git's tree order published as a
  predicate.
- `gitpack` — `GitPackStep`, the ask-and-answer pack reader; the pack
  index with its fanout kept; the offset-delta base encoding on its own;
  and the two bounds a pack from a stranger needs.
- `gitref` — `check_ref_format` answering every rule a name breaks,
  `packed-refs` with its capabilities, and the loose-beats-packed merge
  with the empty-file tombstone.
- `gitindex` — the index file at versions 2, 3 and 4, the `stat` cache
  `status` runs on, the racy-clean rule, and conflicts as stages.
- `gitrepo` — discovery through a `gitdir:` file, the object store over
  loose objects, packs and alternates, refs, `HEAD`, and a paginated
  history walk.
- `gitwork` — status as two independent comparisons per path, the
  `.gitignore` grammar with its three rules a glob does not have, diff
  over diff-nv with rename detection, and a bounded blame.
- `gitwire` — protocol v2's `ls-refs` and `fetch` over HTTPS, the
  `pkt-line` codec as a pure feed-and-drain, and the side-band channel
  check.

### Known

- **`GitPackStep` is the load-bearing interface**, and a packfile leaves
  no other shape: the core asks for a byte range and the host answers.
- **`git-core-nv` earns a row.** Four of the seven modules are `[]`
  already; the README names exactly what would move.
- **The subprocess table is in the README**: which `gitwire` and
  `gitrepo` call replaces each `git` invocation `novo pkg add` spawns
  today, and the three things that buys.
- **Push, SSH, merge, `.git/config` and credential helpers are absent**,
  each with its reason.
- **The SHA-1 is the plain one, not SHA-1DC**, which differs from git on
  crafted collisions and nothing else.
- **A toolchain defect was filed on the way**: a wrapper that returns
  another module's `?Int` lowers the callee's unboxed result as a
  pointer and the IR fails to verify.
- **No device claim.** `Bytes`, `Str` and `Result` are throughout.
- **Three dependencies**, all `core`: crypto-nv, flate-nv, diff-nv.
