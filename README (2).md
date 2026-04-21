# PES-VCS — Version Control System from Scratch

**Author:** Pulkit Handa
**SRN:** PES1UG24AM208
**Course:** Operating Systems / Systems Programming Lab
**Platform:** Ubuntu 22.04

A Git-inspired, content-addressable version control system implemented in C. PES-VCS replicates the core internals of Git — blob/tree/commit object storage, a text-format staging index, SHA-256 hashing, and a linked commit history — built from the ground up without any VCS library dependencies.

---

## Build & Usage

### Prerequisites

```bash
sudo apt update && sudo apt install -y gcc build-essential libssl-dev
```

### Build

```bash
make          # Build the pes binary
make all      # Build pes + test binaries
make clean    # Remove all build artifacts
```

### Author Configuration

```bash
export PES_AUTHOR="Pulkit Handa <PES1UG24AM208>"
```

### Quick Start

```bash
./pes init
echo "hello world" > hello.txt
./pes add hello.txt
./pes commit -m "Initial commit"
./pes log
```

---

## Implementation

### File Overview

| File         | Status      | Description                                      |
|--------------|-------------|--------------------------------------------------|
| `object.c`   | Implemented | Content-addressable object store (blob/tree/commit read & write) |
| `tree.c`     | Implemented | Tree serialization and construction from index   |
| `index.c`    | Implemented | Text-format staging area (load, save, add)       |
| `commit.c`   | Implemented | Commit creation, HEAD update, history walking    |
| `pes.c`      | Provided    | CLI entry point — not modified                   |

---

## Phase 1: Object Store

**Concepts:** SHA-256 content addressing, sharded directory layout, zlib-style object headers.

Implemented `object_write` and `object_read` in `object.c`. Objects are stored under `.pes/objects/<xx>/<remaining-hash>` where the first two hex characters of the SHA-256 hash form the shard directory — identical to Git's fanout layout. The object header format is `<type> <size>\0<content>`, and the full blob (header + content) is hashed to produce the key.

### Screenshots

**Screenshot 1A — `./test_objects` passing:**

> *(Insert screenshot here)*

**Screenshot 1B — `find .pes/objects -type f` showing sharded structure:**

> *(Insert screenshot here)*

---

## Phase 2: Tree Objects

**Concepts:** Binary serialization, recursive directory representation, mode bits.

Implemented `tree_from_index` in `tree.c`. Each entry in the tree is serialized as `<mode> <name>\0<binary-hash>` (20 bytes, not hex). Entries are sorted lexicographically by name before serialization — matching Git's ordering requirement — then written to the object store as a single tree blob.

### Screenshots

**Screenshot 2A — `./test_tree` passing:**

> *(Insert screenshot here)*

**Screenshot 2B — `xxd` of a raw tree object (first 20 lines):**

> *(Insert screenshot here)*

---

## Phase 3: Staging Index

**Concepts:** Atomic writes via `fsync` + rename, filesystem metadata (mtime, size), change detection.

Implemented `index_load`, `index_save`, and `index_add` in `index.c`. The index is a plain-text file where each line holds `<mode> <hash-hex> <mtime> <size> <path>`. Saves are atomic — written to a `.pes/index.tmp` file, `fsync`-ed, then renamed over the old index. `index_add` writes the file's blob to the object store and upserts its entry in the in-memory index.

### Screenshots

**Screenshot 3A — `pes init` → `pes add` → `pes status` sequence:**

> *(Insert screenshot here)*

**Screenshot 3B — `cat .pes/index` showing text-format entries:**

> *(Insert screenshot here)*

---

## Phase 4: Commits and History

**Concepts:** Linked on-disk structures, reference files, atomic pointer updates.

Implemented `commit_create` in `commit.c`. The function calls `tree_from_index` to snapshot the current staged state, reads the existing HEAD as the parent hash (empty string on the first commit), formats the commit object text, writes it to the object store, and atomically updates HEAD via `head_update`. The `pes log` command walks the parent chain via `commit_walk` to display history.

### Screenshots

**Screenshot 4A — `./pes log` showing three commits:**

> *(Insert screenshot here)*

**Screenshot 4B — `find .pes -type f | sort` showing object growth:**

> *(Insert screenshot here)*

**Screenshot 4C — `cat .pes/refs/heads/main` and `cat .pes/HEAD`:**

> *(Insert screenshot here)*

**Integration test — `make test-integration`:**

> *(Insert screenshot here)*

---

## Phase 5: Branching and Checkout (Analysis)

### Q5.1 — Implementing `pes checkout <branch>`

A branch is nothing more than a file in `.pes/refs/heads/<branch>` holding a 64-character hex commit hash. Checkout therefore involves three coordinated steps:

1. **Resolve the target commit.** Read `.pes/refs/heads/<branch>` to get the target commit hash, then parse that commit to get its root tree hash.

2. **Update the working directory.** Recursively walk the target tree (and sub-trees) and for each blob entry, compute whether the working file differs from the stored blob. Files that are present in the target tree but not the current tree must be created. Files present in the current tree but absent from the target tree must be deleted. Files that exist in both but with different hashes must be overwritten.

3. **Update `.pes/` state.** Write the new branch name into `.pes/HEAD` as `ref: refs/heads/<branch>`. Rebuild the in-memory index to reflect every file in the new tree (mode, hash, mtime, size), then atomically save it.

What makes this complex is the multi-way diff required: you need the current HEAD tree, the target tree, *and* the working directory state all simultaneously to decide what to do for each path. You must handle files that exist in one tree but not the other, files that have been renamed (Git uses similarity scoring), and files that exist in the working directory but aren't tracked at all (untracked files should be left alone unless the target tree would overwrite them). All of this must be done atomically enough that an interrupted checkout doesn't leave the repository in a half-migrated state.

---

### Q5.2 — Detecting dirty working directory conflicts during checkout

The index acts as a three-way cache: it remembers the last-staged state of every tracked file. To detect conflicts without scanning either commit tree:

For each file in the current index:
1. **Check if it is staged but not committed** — if `index_find` returns an entry whose hash differs from the current HEAD tree's hash for that path, the file has staged changes.
2. **Check if it is modified but not staged** — stat the file on disk and compare `mtime` and `size` to the index entry. If either differs, read the file and hash it; if the hash differs from the index entry's hash, the file is dirty.

Then, look up the same path in the *target branch's* tree. If the target tree's blob hash differs from the current HEAD tree's blob hash for that path, the path is "in conflict" — the checkout would overwrite a file that has local changes.

The decision rule: if a path is dirty (in either sense above) *and* the target branch carries a different version of that path, refuse the checkout and report the conflicting paths. Paths that are dirty but identical in both branches (e.g., you edited a file to the same content the branch already has) are safe and can proceed.

---

### Q5.3 — Detached HEAD and recovery

In normal operation, `.pes/HEAD` contains a symbolic reference: `ref: refs/heads/main`. In detached HEAD state, it contains a raw commit hash directly, e.g. `a1b2c3d4...`. There is no branch file being updated.

If you make commits while detached, each new commit correctly sets its `parent` to the previous commit, and HEAD is updated to the new commit hash. A proper chain of commits is created and stored in the object store. However, no branch reference points to the tip of that chain — it is reachable only through HEAD while you remain detached.

The moment you switch to another branch, HEAD is overwritten with the new branch reference. Your detached commits become *unreachable* — no ref points to them. They are not immediately deleted (the object store is append-only), but they become invisible to `pes log` and are candidates for garbage collection.

**Recovery options:**
- **Immediately before switching:** run `pes branch <new-branch>` to create a branch at the current detached HEAD, making the chain permanently reachable.
- **After switching (objects still present):** if you remember the commit hash (from terminal history or a note), run `pes branch <new-branch> <hash>` to re-attach a branch to it. In real Git, `git reflog` records every HEAD movement and makes this recovery trivial, but PES-VCS has no reflog — so the hash must be found manually from the object store or shell history.

---

## Phase 6: Garbage Collection (Analysis)

### Q6.1 — Algorithm to find and delete unreachable objects

The approach is a **mark-and-sweep** traversal of the object graph, analogous to tracing garbage collection in memory:

**Mark phase — find all reachable objects:**
1. Start the reachable set as an empty hash set (e.g., a C `uthash` table or a sorted array with binary search).
2. Enumerate every ref: scan `.pes/refs/heads/` and read `.pes/HEAD`. Each ref yields a commit hash — add it to a work queue.
3. For each commit hash in the queue: parse the commit, add its hash to the reachable set. Add its tree hash and its parent hash(es) to the queue if not already seen.
4. For each tree hash: parse the tree entries. Add each entry's hash to the queue (blobs go straight to the reachable set; sub-trees get queued for parsing).
5. Terminate when the queue is empty. Every hash in the reachable set is live.

**Sweep phase — delete unreachable objects:**
1. Walk every file under `.pes/objects/<xx>/`. Reconstruct the full hash from the shard prefix and filename.
2. If the hash is *not* in the reachable set, delete the file.

**Data structure:** A hash set gives O(1) insert and lookup. For the traversal itself, a queue (BFS) avoids deep recursion.

**Scale estimate for 100,000 commits across 50 branches:**
A typical commit references 1 tree + N sub-trees + M blobs. Assuming an average project of ~200 blobs and ~20 trees per commit, but with heavy sharing (most commits change only a few files), a rough estimate is:
- 100,000 commits = 100,000 commit objects to visit
- ~5–10× unique tree objects (sub-trees + sharing) = ~500,000–1,000,000 tree visits
- Unique blobs are bounded by total distinct file versions; conservatively ~2–5 million

In practice you'd visit on the order of **3–6 million objects** during the mark phase. With a hash set, this is fast — essentially a single linear pass of the object store plus the traversal.

---

### Q6.2 — GC race condition with concurrent commits

Consider the following interleaving:

1. A **commit operation** begins. It calls `object_write` for a new blob (hash `X`). The blob is written to `.pes/objects/xx/...` successfully. The commit hasn't yet written its tree or commit object, so nothing points to hash `X` yet.

2. A **GC** starts simultaneously. It enumerates all refs and walks the entire object graph. Since no commit, tree, or ref yet points to hash `X`, GC marks it as unreachable.

3. GC's sweep phase runs and **deletes** the file for hash `X`.

4. The commit operation resumes. It tries to construct a tree object that references hash `X`. The tree is written. The commit is written. HEAD is updated. The repository now contains a commit whose tree references a blob that no longer exists on disk — **repository corruption.**

**How Git avoids this:**

Git's real GC (`git gc`) uses a *grace period* heuristic: it never deletes any object written in the last 2 weeks (configurable via `gc.pruneExpire`). A newly written loose object's filesystem mtime will be within that window, so GC skips it even if no ref currently points to it. This is a pragmatic safety margin rather than a strict lock. For stricter safety, Git also uses lock files (`.git/gc.lock`) to prevent concurrent GC runs, and pack-file creation is done atomically (write to temp, then rename) so a partial pack never replaces valid loose objects.

In PES-VCS, a simple mitigation would be to write a `.pes/GC_LOCK` file at the start of GC and check for it at the start of any mutating operation (commit, add), blocking one until the other completes.

---

## Submission Checklist

- [x] `object.c` — implemented
- [x] `tree.c` — implemented
- [x] `index.c` — implemented
- [x] `commit.c` — implemented
- [ ] Screenshot 1A — `./test_objects` passing
- [ ] Screenshot 1B — sharded object store
- [ ] Screenshot 2A — `./test_tree` passing
- [ ] Screenshot 2B — `xxd` of raw tree object
- [ ] Screenshot 3A — `pes init / add / status` sequence
- [ ] Screenshot 3B — `cat .pes/index`
- [ ] Screenshot 4A — `pes log` with three commits
- [ ] Screenshot 4B — `find .pes -type f | sort`
- [ ] Screenshot 4C — `cat .pes/refs/heads/main` and `cat .pes/HEAD`
- [ ] Integration test — `make test-integration`
- [x] Q5.1 — checkout implementation
- [x] Q5.2 — dirty working directory detection
- [x] Q5.3 — detached HEAD and recovery
- [x] Q6.1 — GC mark-and-sweep algorithm
- [x] Q6.2 — GC race condition and mitigation
- [ ] Minimum 5 commits per phase

---

## References

- [Git Internals — Pro Git Book](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain)
- [Git from the Inside Out](https://codewords.recurse.com/issues/two/git-from-the-inside-out)
- [The Git Parable](https://tom.preston-werner.com/2009/05/19/the-git-parable.html)
