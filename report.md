# PES-VCS Lab Report
**Author:** Bhavyaa Garg  
**SRN:** PES1UG24CS116  
**Repository:** https://github.com/Fickleimagination/PES1UG24CS116-pes-vcs

---

## Phase 1: Object Storage

### Screenshot 1A — `./test_objects` output
See screenshots in OS_JACKFRUIT.pdf

### Screenshot 1B — `find .pes/objects -type f`
See screenshots in OS_JACKFRUIT.pdf

---

## Phase 2: Tree Objects

### Screenshot 2A — `./test_tree` output
See screenshots in OS_JACKFRUIT.pdf

### Screenshot 2B — `xxd` of raw tree object
See screenshots in OS_JACKFRUIT.pdf

---

## Phase 3: Index (Staging Area)

### Screenshot 3A — `pes init` → `pes add` → `pes status`
See screenshots in OS_JACKFRUIT.pdf

### Screenshot 3B — `cat .pes/index`
See screenshots in OS_JACKFRUIT.pdf

---

## Phase 4: Commits and History

### Screenshot 4A — `pes log` showing three commits
See screenshots in OS_JACKFRUIT.pdf

### Screenshot 4B — `find .pes -type f | sort`
See screenshots in OS_JACKFRUIT.pdf

### Screenshot 4C — `cat .pes/refs/heads/main` and `cat .pes/HEAD`
See screenshots in OS_JACKFRUIT.pdf

### Final — `make test-integration`
See screenshots in OS_JACKFRUIT.pdf

---

## Phase 5: Branching and Checkout (Analysis)

### Q5.1: How would you implement `pes checkout <branch>`?

To implement `pes checkout <branch>`, three things must happen:

1. **Update HEAD** — write `ref: refs/heads/<branch>` into `.pes/HEAD`.
2. **Update the working directory** — read the target branch's commit, walk its tree object, and overwrite every tracked file in the working directory with the blob content from the object store.
3. **Update the index** — replace the current index entries with the entries corresponding to the target tree.

What makes this complex: the working directory contains files not tracked by PES (untracked files) that must not be touched. Files present in the current branch but absent in the target must be deleted. Files modified but not committed create conflicts — checkout must detect these by comparing the working file's hash against the index entry and refuse if they differ.

### Q5.2: How to detect a "dirty working directory" conflict?

For each file tracked in the index:
1. Compute the SHA-256 of the current working file content.
2. Compare it against the blob hash stored in the index entry.
3. If they differ, the file is locally modified and unstaged.

Then check if that same file path exists in the target branch's tree with a different blob hash. If both conditions are true — locally modified AND differs between branches — checkout must refuse with an error. This uses only the index (for the current staged state) and the object store (for both branch snapshots), with no other metadata needed.

### Q5.3: What happens in detached HEAD state?

When HEAD contains a raw commit hash instead of `ref: refs/heads/<branch>`, new commits are written to the object store and HEAD is updated to point to the new commit hash directly. However, no branch pointer moves — the commits exist but are only reachable via HEAD itself.

If the user then checks out a branch, HEAD changes and the detached commits become unreachable — no branch or tag points to them. To recover, the user must know the commit hash (visible in the terminal history or via `pes log` before switching) and create a new branch pointing to it: create a file `.pes/refs/heads/<newbranch>` containing that hash, then update HEAD to reference it.

---

## Phase 6: Garbage Collection (Analysis)

### Q6.1: Algorithm to find and delete unreachable objects

**Algorithm (mark-and-sweep):**

1. Start from all branch refs in `.pes/refs/heads/` and collect their commit hashes into a set called `reachable`.
2. For each commit hash in `reachable`, read the commit object, add its tree hash to `reachable`, add its parent hash (if any) to `reachable`, and enqueue the parent for processing.
3. For each tree hash in `reachable`, read the tree object, add all blob hashes and subtree hashes to `reachable`.
4. Walk all files in `.pes/objects/`. Any object whose hash is not in `reachable` is unreachable — delete it.

**Data structure:** A hash set (e.g., a C `uthash` table or a sorted array with binary search) for O(1) average lookup of reachable hashes.

**Estimate for 100,000 commits, 50 branches:** Assuming an average of 10 files per commit, each commit has 1 tree + ~10 blobs = ~11 objects. With deduplication across commits (unchanged files share blobs), a conservative estimate is visiting ~500,000–1,000,000 objects total across the mark phase.

### Q6.2: Race condition between GC and concurrent commit

**The race:**
1. A commit operation calls `tree_from_index()`, which writes new blob objects to the object store. At this point, the blobs exist but no commit or branch points to them yet.
2. GC runs its mark phase — it traverses all branch refs and finds these new blobs unreachable (no commit references them yet).
3. GC deletes the blobs.
4. The commit operation calls `object_write()` for the commit object and `head_update()` — but the blobs it references are now gone. The repository is corrupted.

**How Git avoids this:** Git's GC uses a "grace period" — it only deletes objects older than 2 weeks by default (`gc.pruneExpire`). New objects are always recent, so they survive GC even if temporarily unreferenced. Git also writes a `gc.pid` lock file to prevent concurrent GC runs, and the commit operation completes in milliseconds, well within the grace window.

