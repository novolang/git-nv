# git-nv

Git plumbing for novo-lang, without libgit2: objects, packfiles, refs,
the index, and the porcelain a tool needs — with the format half
declaring no effects at all.

**Status: NOT IMPLEMENTED — interface only.**  Every `pub fn` body is a
`todo()`, so the signatures, the effect rows and the tests are published
and nothing is implemented.  The first implementation is the `0.1.0`
published over this.

## What this is

| module | holds | rows |
| --- | --- | --- |
| `gitobj` | blob, tree, commit, tag; ids under either hash | `[]` |
| `gitpack` | the pack index, and delta resolution over ranges | `[]` |
| `gitref` | ref names, loose refs, `packed-refs` | `[]` |
| `gitindex` | the index file, and the `stat` cache `status` runs on | `[]` |
| `gitrepo` | discovery, the object store, refs and history | `[fs]` |
| `gitwork` | status, ignore rules, diff, blame | `[]`, `[fs]` |
| `gitwire` | the smart HTTP read half | `[]`, `[net]` |

## The load-bearing interface

```novo norun:pseudo
pub enum GitPackStep
    GitPackNeeds(read: GitPackRead, reader: GitPackReader)
    GitPackDone(kind: GitObjKind, payload: Bytes)
    GitPackFollows(read: GitPackRead, reader: GitPackReader)
```

**The core asks and the host answers, and a packfile leaves no other
choice.**  Reading one object out of a pack means seeking to an offset
the `.idx` gave you, inflating a header, discovering the object is a
delta against another offset, seeking there, and repeating until the
chain bottoms out.  `docs/publishing.md` § How a `core` package takes
bytes from its host calls this the third shape; it is the right one here
for the reason it is right for btree-nv's pages — a reader that streamed
the pack would read a gigabyte to answer one object.

So `GitPackRead` is a byte range, `gitpack.step` answers either another
range or a finished object, and **nothing in `gitpack` opens a file**.
A caller with the pack in memory, in a mapped region, or behind an HTTP
range request uses the same reader.

`GitPackStep` is an enum and not a `Result` because "I need these bytes"
is neither a success nor a failure, and a reader that could not say it
would have to own the file.

## The one example that will work

```novo
use gitobj
use gitrepo
use gitwork

fn main() [io]
    match gitrepo.open_from(".")
        Err(e) => println(e.message())
        Ok(r)  =>
            match gitwork.is_clean(r)
                Err(e) => println(e.message())
                Ok(yes) => println("${yes}")
```

## Adding it, and checking it

```console
$ novo pkg add git-nv
$ novo pkg build
$ novo test tests/gitobj_tests.nv
```

The suites are **red on purpose**: every body is a `todo()`, so every
assertion reaches `not implemented: git-nv.<module>.<fn>`.

`novo pkg add` says `NOT IMPLEMENTED — interface only` on the way in,
because an interface resolves, downloads and builds exactly like an
implemented package and the difference only shows the first time
something calls it.

## The layer, and the missing row

`host`, from the plan — and the split inside it is the design.  **Four
of the seven modules declare nothing**: the object model, the packfile
reader, the index file and the reference grammar are arithmetic over
bytes the caller already holds, and they are where the format actually
lives.  Three perform: `gitrepo` and `gitwork`'s working-tree half read
the disk (`[fs]`), and `gitwire` talks to a server (`[net]`).

**`git-core-nv` earns a row.**  Those four modules at `core` would let a
package that only reads objects take the format without taking a
filesystem — a registry verifying a tag's commit, a build cache keyed by
tree id, a signer that reads a commit's bytes and never touches a work
tree.  Every one of those modules is already `[]`, and the line between
them and the other three is the line between "what the format says" and
"where the bytes came from".

What would move: `gitobj`, `gitpack`, `gitref`, `gitindex`, plus
`gitwork`'s `parse_gitignore`, `gitignore_matches`, `gitignore_reason`,
`diff_blobs`, `similarity` and `paths_to_rehash`, which are `[]` today
and sit in `gitwork` only because that is where their consumers are.
What stays: `gitrepo`, `gitwire`, and the rest of `gitwork`.

The alternative — `layer = "core"` with
`host_modules = ["gitrepo", "gitwork", "gitwire"]` — would work and
would move this package's cell on the Orbit map away from where the
must-have plan put it.  That is the milestone review's call, not this
lane's.

## What replaces the subprocess

`novo pkg add` resolves a git dependency by spawning `git`
(`compiler/bin/novo.ml`), and this is what each call becomes:

| today | with this package |
| --- | --- |
| `git ls-remote --tags <url>` | `gitwire.ls_refs(remote, ["refs/tags/"], true)` |
| `git clone --depth 1 <url> <dir>` | `gitwire.ls_refs` for the tip, then `gitwire.fetch(remote, [tip], [], 1)` and `gitrepo.write_object` per object |
| `git clone <url> <dir>` | the same with `depth` 0 |
| `git -C <dir> checkout --detach <rev>` | `gitrepo.peel_to_tree` and a walk writing files; `gitrepo.write_ref` for the detached `HEAD` |
| `git -C <dir> rev-parse HEAD` | `gitrepo.head(r).commit` |
| `git -C <dir> fetch --depth 1 origin <rev>` | `gitwire.fetch(remote, [rev], have, 1)` |
| `git status --porcelain`, for the publish's dirty check | `gitwork.is_clean(r)`, or `gitwork.porcelain_line` per entry to produce the same text |

Three things that buys, beyond removing a dependency on a program being
installed. `ls_refs` takes a ref PREFIX, so resolving one tag out of a
repository with fifty thousand refs stops downloading all fifty thousand.
The errors become values with a status number on them, so a 404 and a
401 lead to different messages rather than both being "git clone
failed". And the resolver stops shelling out, which is what makes it
usable from a program that has no shell — a build server in a container
with no git, and eventually novo-lang's own registry backend.

## What is deliberately absent

- **Push.**  `receive-pack` and the send-side negotiation.  A package
  manager never pushes, and a half-implemented push is worse than none.
- **SSH and the git daemon.**  Two more transports with their own
  authentication stories; HTTPS is what a forge and a registry both
  speak, and it is the one the standard library already has.
- **Merge, rebase, cherry-pick.**  Reading a repository and writing one
  are different amounts of work, and this is the reading half.
- **`.git/config`.**  That is config-nv's grammar, and a second TOML-ish
  parser here would be a second thing to keep in step.
- **Credential helpers, `.netrc`, `GIT_ASKPASS`.**  `GitRemote` takes
  the whole `Authorization` header value the caller built, because
  deciding where a secret lives is not this package's business.
- **SHA-1DC.**  Git's SHA-1 detects the known collision attack and
  refuses; crypto-nv has plain SHA-1.  For every object anybody has
  committed the two agree; for the handful of crafted collisions this
  package would hash them and git would refuse.

## Five places the format bites, and where each one is

Every item here is a rule a hand-written git tool gets wrong, and each
is published as its own function so a test can name it:

- **Tree order is not lexicographic.**  A directory sorts as though its
  name ended in `/`, so `foo.txt` comes before `foo/bar`.  A tree in the
  wrong order is an object git reads and whose id differs from what git
  would have written — `gitobj.entries_sorted`.
- **The offset-delta base is not LEB128.**  Big-endian base-128 where
  each continuation adds one before shifting, so `0x80 0x00` is 128
  rather than 0.  A reader that reused a varint routine works on a test
  pack and fails on a real one — `gitpack.parse_ofs_delta_base`.
- **A loose ref beats a packed one, and an empty loose file is a
  deletion.**  A reader that concatenated the two sources reports a
  branch at the commit before its last — `gitref.merge_sources`.
- **A ref name is a file path.**  `..` in one walks out of the
  repository, and a fetch creates refs from names a stranger chose —
  `gitref.check_ref_format`, answering every rule broken rather than the
  first.
- **The index is a `stat` cache and `status` runs on it.**  A model
  without the `stat` block produces a `status` that is correct and forty
  times slower; the racy-clean rule is why an entry written in the
  index's own second must be hashed anyway — `gitindex.needs_rehash`
  and `gitindex.is_racy`.

## What widened, and what did not

- **A wrapper returning another module's `?Int` does not compile.**  A
  doc example for `gitpack.offset_of` was written that way and produced
  IR LLVM refuses; the example matches on the optional instead, and the
  defect is filed.
- **`gitwork` holds `[]` and `[fs]` functions side by side**, which the
  budget permits at `host` and which the `git-core-nv` split above would
  separate.
- **No device claim.**  `Bytes`, `Str` and `Result` are throughout, and
  a repository is a filesystem.

## Reference

libgit2 is the reference implementation and its test fixtures are the
oracle; git's own `Documentation/technical/` — the pack format, the
index format, `protocol-v2` — is the specification, and gitoxide and
dulwich are the two ports read for what a subset can honestly leave out.

## Licence

Apache-2.0.
