# PES-VCS Lab Report

## Repository

GitHub Repository: `https://github.com/shashankd06/OS--Orange`

## Screenshot Checklist

Take the following screenshots from the repository root in WSL or Ubuntu after building the binaries.

### Phase 1

Screenshot 1A

```bash
gcc -Wall -Wextra -O2 -o test_objects test_objects.c object.c -lcrypto
./test_objects
```

Screenshot 1B

```bash
find .pes/objects -type f
```

### Phase 2

Screenshot 2A

```bash
gcc -Wall -Wextra -O2 -o test_tree test_tree.c object.c tree.c index.c -lcrypto
./test_tree
```

Screenshot 2B

```bash
find .pes/objects -type f
xxd .pes/objects/XX/YYY... | head -20
```

Replace `XX/YYY...` with any tree object path from the previous command.

### Phase 3

Screenshot 3A

```bash
gcc -Wall -Wextra -O2 -o pes object.c tree.c index.c commit.c pes.c -lcrypto
rm -rf .pes
./pes init
echo "hello" > file1.txt
echo "world" > file2.txt
./pes add file1.txt file2.txt
./pes status
```

Screenshot 3B

```bash
cat .pes/index
```

### Phase 4

Screenshot 4A

```bash
rm -rf .pes
./pes init
echo "Hello" > hello.txt
./pes add hello.txt
./pes commit -m "Initial commit"
echo "World" >> hello.txt
./pes add hello.txt
./pes commit -m "Add world"
echo "Goodbye" > bye.txt
./pes add bye.txt
./pes commit -m "Add farewell"
./pes log
```

Screenshot 4B

```bash
find .pes -type f | sort
```

Screenshot 4C

```bash
cat .pes/refs/heads/main
cat .pes/HEAD
```

### Final Integration Test

```bash
bash test_sequence.sh
```

If your checkout has CRLF line endings and Bash complains, run:

```bash
python3 - <<'PY'
from pathlib import Path
src = Path("test_sequence.sh").read_text().replace("\r\n", "\n")
Path(".test_sequence_lf.sh").write_text(src)
PY
bash .test_sequence_lf.sh
```

## Analysis Questions

### Q5.1

A branch checkout needs to change both repository metadata and the working tree. In `.pes/`, `HEAD` must either be updated to `ref: refs/heads/<branch>` or already point there, and the target branch file in `.pes/refs/heads/<branch>` must be read to get the target commit hash. From that commit, PES would read the commit object, then the root tree, then recursively reconstruct every tracked file into the working directory. Files that exist in the current branch but not in the target branch would need to be deleted, files that differ would need to be overwritten, and the index would need to be rewritten to match the checked-out snapshot.

The operation is complex because checkout is not just metadata switching. It must compare the current working tree, index, and target tree carefully to avoid overwriting user work. Nested directories have to be created or removed in the right order, executable bits have to be restored, and uncommitted changes may make some paths unsafe to replace.

### Q5.2

To detect a dirty-working-directory conflict using only the index and object store, I would start from the current index entry for each tracked file. The index already stores the staged blob hash plus the file's size and modification time when it was staged. First, compare each working-directory file's current metadata against the index. If the size or mtime changed, the file is a candidate for re-hashing. For those candidates, read the file, compute its blob hash using the same object format as `pes add`, and compare it to the index hash. If the hash differs, the file has uncommitted working-tree changes.

Then compare the target branch's tree to the current branch's tracked paths. If a file is both dirty relative to the index and would be changed, deleted, or replaced by the target branch, checkout must refuse. That is the dangerous case: the target checkout would overwrite content the user has not committed.

### Q5.3

In detached HEAD state, `HEAD` contains a commit hash directly instead of referring to a branch file. New commits still work, but each new commit updates `HEAD` itself rather than moving a named branch. That means the commits exist in the object store and are reachable only from the detached HEAD position. If the user later checks out a normal branch, those commits may become unreachable because no branch name points to them anymore.

The user can recover them as long as they still know the commit hash or have not garbage-collected the repo. The simplest recovery is to create a new branch or ref pointing to that detached commit, for example by writing its hash into `.pes/refs/heads/recovered` and then making `HEAD` refer to that branch. Conceptually, recovery means "give this orphaned chain a permanent name."

### Q6.1

The right algorithm is mark-and-sweep. In the mark phase, begin from every root reference: all branch files in `.pes/refs/heads/` and possibly `HEAD` if detached. For each reachable commit, mark the commit hash, then mark its tree hash, then recursively traverse tree entries and mark every subtree and blob they reference, and finally follow the parent commit link until the root. A hash set is the right data structure for the reachable set because membership checks must be fast and duplicate visits are common.

In the sweep phase, walk every file in `.pes/objects/`. Convert each object path back to its full hash and delete any object whose hash is not in the reachable set.

For 100,000 commits and 50 branches, GC does not necessarily visit 5,000,000 distinct commits because branch histories overlap heavily. In the worst case, if all branches are disjoint, it would visit roughly all commits reachable from those branches plus the trees and blobs they reference. In a more realistic repository with shared history, it would still be on the order of the total unique reachable objects in the DAG: roughly 100,000 commits, around 100,000 root trees plus many subtree objects, and however many unique blobs are still referenced.

### Q6.2

Concurrent GC is dangerous because commit creation is not instantaneous. A commit operation may write new blob objects first, then write tree objects, then finally write the commit object and update the branch ref. If GC runs in the middle, it may scan the refs, fail to see the not-yet-linked new tree or blob objects as reachable, and delete them. The concurrent commit would then finish by creating a commit that points to objects that GC has already removed.

One race looks like this:

1. `pes add` or `pes commit` writes a new blob object.
2. GC starts from branch refs and marks reachable objects.
3. Because the new commit object and branch update do not exist yet, the new blob is not marked.
4. GC sweeps and deletes that blob.
5. Commit creation finishes and writes a tree and commit that reference the now-missing blob.

Real Git avoids this with a combination of locking, temporary object handling, reflogs, grace periods, and conservative reachability rules. The general idea is that objects being written are protected until references are updated, and GC does not aggressively delete very recent or in-progress objects.
