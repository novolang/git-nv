# git-nv

Git stores a project's history as objects named by the hash of their own
contents, packed into files, pointed at by references, and staged through an
index. This package reads a git repository in novo-lang, without libgit2 and
without running the `git` program. It reads the formats git documents —
[gitformat-pack](https://git-scm.com/docs/gitformat-pack),
[gitformat-index](https://git-scm.com/docs/gitformat-index),
[gitignore](https://git-scm.com/docs/gitignore),
[git-check-ref-format](https://git-scm.com/docs/git-check-ref-format) and
[gitrepository-layout](https://git-scm.com/docs/gitrepository-layout) — and it
speaks the client half of
[protocol v2](https://git-scm.com/docs/protocol-v2) over
[smart HTTP](https://git-scm.com/docs/http-protocol).
[git-core-nv](https://novo-lang.org/packages/git-core-nv) publishes the format
half on its own, for a program that wants to read git's bytes without a
filesystem.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What it is

An **object** is the unit git stores. There are four kinds: a **blob** is a
file's contents, a **tree** is a directory listing, a **commit** is one revision
of the whole project, and a **tag** is an annotated name for another object. An
object's **id** is the hash of the string `<kind> <length>\0` followed by the
object's contents. Git names objects under SHA-1 today and is moving to SHA-256,
and the two give different ids for the same contents.

A **loose object** is one object in its own file, compressed with zlib. A
**packfile** holds many objects in one file, most of them stored as **deltas**:
copy and insert instructions against another object. A packfile is paired with
an **index file**, the `.idx`, which says at what offset each id lives.

A **reference**, or ref, is a name for an object. A ref lives either in its own
small file under `.git/refs`, called a **loose ref**, or as a line in
`.git/packed-refs`. A **symbolic ref** names another ref, which is what `HEAD`
is on a branch.

The **index**, `.git/index`, is the file git stages changes in. It is also a
cache: each entry records the `stat` data the filesystem reported when the file
was last hashed, so `git status` can skip files whose size, mode and timestamps
are unchanged.

The **working tree** is the checked-out files themselves. `git status` answers
two independent questions about each path: what differs between the last commit
and the index, and what differs between the index and the working tree. A path
can be in a different state in each.

The **smart HTTP protocol** is how a client asks a server what refs it has and
downloads a packfile of the objects behind some of them. Protocol v2 lets the
client ask for refs under a prefix, so listing one repository's tags does not
download every ref it has.

Four of this package's seven modules perform no input or output. They take the
bytes as an argument and answer a value. Two read the disk. One talks to a
server.

## Install

```
novo pkg add git-nv
```

## Example

```novo
use gitrepo
use gitwork

fn main() [io, fs]
    // Walk upward from here looking for .git, and open what is found.
    match gitrepo.open_from(".")
        Err(e) => println(e.message())
        Ok(r)  =>
            // The commit HEAD is at, and the branch it is on.
            match gitrepo.head(r)
                Err(e) => println(e.message())
                Ok(h)  => println(h.branch)

            // Whether anything is staged or modified. No working-tree walk.
            match gitwork.is_clean(r)
                Err(e)  => println(e.message())
                Ok(yes) => println("${yes}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails on
purpose: every test reaches a `not implemented` panic.

## What the package contains

| Module | Contents |
| --- | --- |
| `gitobj` | Blobs, trees, commits and tags, to and from bytes, and object ids under either hash. Performs no input or output. |
| `gitpack` | The pack index, the pack entry headers, and delta resolution as a reader that asks its caller for byte ranges. Performs no input or output. |
| `gitref` | Ref names and the rules they must follow, loose refs, `packed-refs`, and resolving a short name to a full one. Performs no input or output. |
| `gitindex` | The index file, its `stat` cache, its merge stages and its extensions. Performs no input or output. |
| `gitrepo` | Finding a repository, the object store over loose objects, packs and alternates, reading and writing refs, `HEAD`, and a paginated history walk. Reads and writes files. |
| `gitwork` | Status, the `.gitignore` grammar, the diff between two trees, rename detection and blame. Part reads files, part does not. |
| `gitwire` | Protocol v2's `ls-refs` and `fetch` over HTTPS, and the `pkt-line` framing under them. Part talks to a server, part does not. |

## How to choose an entry point

**A program working with a repository on disk starts at
`gitrepo.open_from`.** It walks upward for `.git`, follows a `gitdir:` file,
parses every pack index once, and answers a value the rest of the package takes.

**A program that already holds an object's bytes starts at `gitobj`.** No
repository is needed to compute an id or parse a commit.

**A program talking to a server starts at `gitwire.remote`.** It answers a
value carrying the URL, the deadline, the agent string and the `Authorization`
header, and `ls_refs` and `fetch` take it.

**A program that wants the format without a filesystem uses
[git-core-nv](https://novo-lang.org/packages/git-core-nv).** That package is the
four pure modules on their own, with no dependency that reads a disk.

A program replacing calls to the `git` program has these equivalents.

| `git` command | This package |
| --- | --- |
| `git ls-remote --tags <url>` | `gitwire.ls_refs(remote, ["refs/tags/"], true)` |
| `git clone --depth 1 <url>` | `gitwire.ls_refs` for the tip, then `gitwire.fetch(remote, [tip], [], 1)` and `gitrepo.write_object` per object |
| `git clone <url>` | The same with `depth` 0 |
| `git -C <dir> checkout --detach <rev>` | `gitrepo.peel_to_tree`, a walk writing files, then `gitrepo.write_ref` for the detached `HEAD` |
| `git -C <dir> rev-parse HEAD` | `gitrepo.head(r).commit` |
| `git -C <dir> fetch --depth 1 origin <rev>` | `gitwire.fetch(remote, [rev], have, 1)` |
| `git status --porcelain` | `gitwork.status`, then `gitwork.porcelain_line` per entry |
| `git status --porcelain`, only to ask whether anything changed | `gitwork.is_clean(r)` |

## The rules a user needs

1. **An object's id covers a header you must not forget.** The hash is over
   `<kind> <length>\0` and then the payload. `gitobj.oid_of` prepends it.
   gitrepository-layout, "Object storage format".
2. **Tree order is not lexicographic.** A directory compares as though its name
   ended in `/`, so `foo.txt` sorts before `foo/bar`. A tree in any other order
   is an object git reads whose id differs from the one git would have written.
   `gitobj.entries_sorted` is the rule.
3. **The two hashes are separate namespaces.** `gitobj.oid_eq` answers `false`
   for a SHA-1 id and a SHA-256 id whatever their bytes.
4. **`.git` is not always a directory.** In a worktree and in a submodule it is
   a file containing `gitdir: <path>`. `gitrepo.discover` follows it.
   gitrepository-layout, "gitdir".
5. **An object may be in three places, and the order matters.** Loose first,
   then each pack, then the alternates listed in `objects/info/alternates`.
   `gitrepo.read_object` walks all three. A reader that skipped the alternates
   reports most of a `--reference` clone missing.
6. **The pack reader asks and the caller answers.** `gitpack.read_at` and
   `gitpack.step` return a byte range to read. Nothing in `gitpack` opens a
   file. `gitrepo.read_object` runs that loop for you.
   gitformat-pack, "Pack file format".
7. **The offset-delta base is not LEB128.** Bytes are big-endian, seven bits
   each, and each continuation adds one to the accumulated value before
   shifting. `0x80 0x00` is 128, not 0. `gitpack.parse_ofs_delta_base` is that
   encoding on its own. gitformat-pack, "Deltified representation".
8. **A delta declares its output size before anything is allocated.** Set
   `max_object_bytes` on `GitPackLimits` for any pack from a stranger. The chain
   depth is bounded at 50 by default, and a cycle is a fault rather than a hang.
9. **A loose ref beats a packed one, and an empty loose file is a deletion.**
   Git writes an empty loose file as a tombstone over a packed ref.
   `gitref.merge_sources` is the rule, and `gitrepo.refs` applies it.
   gitrepository-layout, "packed-refs".
10. **A ref name is a file path, and a fetch creates refs from names a stranger
    chose.** `gitrepo.write_ref` refuses a name `gitref.check_ref_format`
    rejects before it touches the filesystem. git-check-ref-format.
11. **A short name resolves in a fixed order, and a tag beats a branch.** The
    order is the name itself, then `refs/`, `refs/tags/`, `refs/heads/`,
    `refs/remotes/` and `refs/remotes/<name>/HEAD`. `gitref.expand` follows it.
12. **`HEAD` may resolve to nothing.** A fresh repository names a branch whose
    file does not exist yet. `gitrepo.head` answers `None` for the commit, and
    `GitHead.detached` is the other state a tool must tell apart.
13. **`gitrepo.read_index` answers the bytes as well as the index.** Every path
    in an index entry is a range into those bytes. A caller that dropped them
    could not read a single name.
14. **The index is keyed by path and stage together.** Stage 0 is an ordinary
    file. Stages 1, 2 and 3 are the base, ours and theirs of an unresolved
    conflict. `gitindex.conflicts` is the query. gitformat-index, "Index entry".
15. **A file written in the index's own second must be hashed anyway.** Its
    mtime cannot distinguish "unchanged" from "changed within the second".
    `gitindex.is_racy` is that test. Skipping it reports a modified file as
    clean, once, unreproducibly.
16. **`needs_rehash` takes the two configuration flags as arguments.** `ctime`
    is compared only under `core.trustCtime`, and `dev`, `ino`, `uid` and `gid`
    only under `core.checkStat = default`. A checkout on a network filesystem
    has all four wrong.
17. **A status entry carries two changes, not one.** `staged` compares `HEAD`
    against the index and `unstaged` compares the index against the working
    tree. A path staged as modified and then modified again is both.
18. **Untracked files are a separate call.** `gitwork.status` does not walk the
    working tree. `gitwork.untracked` does, with a bound, because a repository
    with an unignored build directory has half a million untracked paths.
19. **A `.gitignore` pattern with no slash matches at any depth, and one with a
    slash is anchored.** `build/` and `*/build` therefore mean different things.
    gitignore, "Pattern format".
20. **The last matching rule in the deepest file wins, and a directory exclusion
    prunes the walk.** `build/` plus `!build/keep.txt` keeps nothing, because
    git never descends into an excluded directory.
    `gitwork.gitignore_reason` names the rule and the line that decided.
21. **Rename detection is quadratic and gives up.** `GitRenameOptions` carries
    the similarity threshold, 50 per cent by default, and `rename_limit`, 1000
    by default. Above the limit the detector stops rather than take minutes.
22. **`gitwork.blame` takes a budget in commits.** `GitBlame.exhausted` says
    whether it ran out, and `commits_walked` says how far it got, so a caller
    raises the budget by a number rather than a guess.
23. **A `pkt-line`'s four hex digits are the total length including
    themselves.** `0006a\n` is a six-byte packet carrying two bytes. Writing the
    payload length instead produces a stream a server reads one packet of and
    then disconnects from, with no error. gitformat-pack, "pkt-line format".
24. **In a side-band response the first payload byte is the channel.** 1 is pack
    data, 2 is progress, 3 is a fatal error. `gitwire.sideband_of` reads it.
    Feeding channel 2 into a pack reader reports a corrupt pack for a server
    that was reporting its progress.
25. **`gitwire.fetch` answers pack bytes, not a repository.** Writing them is
    `gitrepo.write_object`. `GitFetched.shallow` carries the boundary commits of
    a shallow history, and a caller that ignored them writes a repository whose
    `log` walks off the end.
26. **A credential is a header value you built.** `gitwire.with_authorization`
    takes the whole `Authorization` header. This package reads no credential
    helper, no `.netrc` and no environment variable.
27. **`gitwire.is_retryable` decides whether to try again.** Unreachable and a
    5xx status, yes. A 401, a 403, a 404 and every protocol fault, no.

## What is not included

- **Push.** `receive-pack` and the send-side negotiation. This package is the
  reading half.
- **SSH and the git daemon.** `gitwire.is_supported_url` accepts `https://` and
  `http://` only, so a program that routes other schemes elsewhere can tell.
- **Merge, rebase and cherry-pick.** Writing a repository is a larger job than
  reading one.
- **`.git/config`.** That is a config file grammar, and
  [config-core-nv](https://novo-lang.org/packages/config-core-nv) is the package
  for it. The two settings this package needs, `core.trustCtime` and
  `core.checkStat`, arrive as arguments.
- **Credential helpers, `.netrc` and `GIT_ASKPASS`.** Deciding where a secret
  lives is the caller's.
- **The v0 protocol's full capability set.** `multi_ack_detailed`, `shallow`
  beyond depth 1 and `filter` for partial clones are absent. v0 is parsed only
  far enough to read a ref advertisement from a server that does not offer v2.
- **SHA-1DC.** Git uses a collision-detecting SHA-1 that refuses the known
  crafted collisions. This package uses plain SHA-1. For every object anyone has
  committed the two agree.
- **Running on a microcontroller.** `Bytes`, `Str` and `Result` are used
  throughout, and a repository is a filesystem. This package makes no claim.

## Related packages

- [git-core-nv](https://novo-lang.org/packages/git-core-nv) is the format on its
  own: the object model, the packfile reader, the reference grammar, the index,
  the ignore grammar and the blob diff, with nothing that opens a file. A
  registry verifying a tag's commit, a build cache keyed by a tree id or a
  signer reading a commit's bytes takes that package and not this one. Every
  type and function name is the same in both.
- [crypto-nv](https://novo-lang.org/packages/crypto-nv) supplies SHA-1 and
  SHA-256. An object's id is a hash, so this dependency is not optional.
- [flate-nv](https://novo-lang.org/packages/flate-nv) supplies zlib. Every loose
  object is a zlib stream, and so is every packed object's payload.
- [diff-nv](https://novo-lang.org/packages/diff-nv) supplies the diff itself.
  `gitwork.diff_blobs` tokenises two blobs and hands them to `diffscript.diff`,
  so the hunks a caller renders are diff-nv's.
- `std.fs` and `std.process` in the standard library are the alternatives: read
  the files yourself, or run the `git` program and parse its output. Neither
  knows the formats.
- `std.http-client` in the standard library is the transport `gitwire` uses. It
  has no notion of a ref or a packfile.

## Tests

```bash
novo test tests                          # every suite
novo test tests/gitobj_tests.nv          # the ids, and the tree order
novo test tests/gitpack_tests.nv         # the pack numbers, and the offset delta
novo test tests/gitref_tests.nv          # the ref-name rules
novo test tests/gitindex_tests.nv        # the stat cache, and the racy second
novo test tests/gitwork_tests.nv         # status, and the ignore grammar
novo test tests/gitwire_tests.nv         # the pkt-line framing and the side-band
novo test tests/gitrepo_tests.nv         # the fault messages and the signatures
```

| Suite | Tests |
| --- | --- |
| `gitobj_tests.nv` | 17 |
| `gitref_tests.nv` | 16 |
| `gitwork_tests.nv` | 15 |
| `gitindex_tests.nv` | 14 |
| `gitwire_tests.nv` | 12 |
| `gitpack_tests.nv` | 10 |
| `gitrepo_tests.nv` | 6 |

`novo test` fails on purpose today. Every assertion reaches a `not implemented:
git-nv.<module>.<fn>` panic, because every body is a `todo()`. The tests are the
specification the implementation will have to satisfy.

The reference data is git's own. The empty-blob and empty-tree ids are the
constants in git's `hash.c`. The pack numbers are
`Documentation/technical/pack-format.txt` and `packfile.c`. The ref names are
the accepted and refused cases of git's `t/t1402-check-ref-format.sh`. The index
facts are `Documentation/technical/index-format.txt` and `read-cache.c`. The
ignore pattern pairs are the shapes in git's `t/t0008-ignores.sh`. The
`pkt-line` and side-band cases are `Documentation/technical/protocol-common.txt`
and `protocol-v2.txt`.

`gitrepo_tests.nv` asserts the fault messages and the signatures only. Every
function in that module reads or writes a file, and a suite that built a
repository on disk would be testing the filesystem as much as the package. The
four pure modules carry the format coverage.

libgit2 is the reference implementation the fixtures are checked against.
gitoxide and dulwich are the two ports read for what a subset can honestly leave
out.

## Implementation status

| Item | Implemented |
| --- | --- |
| `gitobj`'s eight types, and `impl Error for GitObjFault` | declared |
| `gitobj`'s 23 functions, from `oid_of` to `mode_is_tree` | no |
| `gitpack`'s eight types, and `impl Error for GitPackFault` | declared |
| `gitpack`'s 16 functions, from `default_limits` to `depth_of` | no |
| `gitref`'s four types, and `impl Error for GitRefNameFault` | declared |
| `gitref`'s 15 functions, from `check_ref_format` to `head_name` | no |
| `gitindex`'s six types, and `impl Error for GitIndexFault` | declared |
| `gitindex`'s 13 functions, from `parse` to `path_of` | no |
| `gitrepo`'s eight types, and `impl Error for GitRepoFault` | declared |
| `gitrepo`'s 17 functions, from `discover` to `entry_at_path` | no |
| `gitwork`'s eight types, from `GitStatusEntry` to `GitBlameLine` | declared |
| `gitwork`'s 15 functions, from `default_rename_options` to `porcelain_line` | no |
| `gitwire`'s seven types, and `impl Error for GitWireFault` | declared |
| `gitwire`'s 15 functions, from `remote` to `is_retryable` | no |

114 public functions and six `Error` implementations, every body a `todo()`.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
