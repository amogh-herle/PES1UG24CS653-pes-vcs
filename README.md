# PES-VCS Lab Report

## 1. Screenshots

### Phase 1: Object Storage Foundation
* **1A: `./test_objects` output**
  ![Screenshot 1A - test_objects](Screenshots/Screenshot%201A.png)
* **1B: `find .pes/objects -type f` output**
  ![Screenshot 1B - sharded directory](Screenshots/Screenshot%201B.png)

### Phase 2: Tree Objects
* **2A: `./test_tree` output**
  ![Screenshot 2A - test_tree](Screenshots/Screenshot%202A.png)
* **2B: `xxd` of a raw tree object**
  ![Screenshot 2B - tree object hex](Screenshots/Screenshot%202B.png)

### Phase 3: The Index (Staging Area)
* **3A: `pes init` → `pes add` → `pes status` sequence**
  ![Screenshot 3A - add sequence](Screenshots/Screenshot%203A.png)
* **3B: `cat .pes/index` output**
  ![Screenshot 3B - index file](Screenshots/Screenshot%203B.png)

### Phase 4: Commits and History
* **4A: `pes log` output**
  ![Screenshot 4A - pes log](Screenshots/Screenshot%204A.png)
* **4B: Object store growth**
  ![Screenshot 4B - object growth](Screenshots/Screenshot%204B.png)
* **4C: Reference chain**
  ![Screenshot 4C - refs and HEAD](Screenshots/Screenshot%204C.png)

### Final Integration Test
* **Full integration test (`make test-integration`)**
  ![Screenshot Final - integration test](Screenshots/Screenshot%20Final.png)

---

## 2. Analysis Questions

### Q5.1: Branching and Checkout Implementation
* Modify `.pes/HEAD` to contain `ref: refs/heads/<branch>`.
* Read the target branch's commit hash from `.pes/refs/heads/<branch>`.
* Traverse the target commit's tree to update the working directory files.
* Complexity stems from preventing the overwrite of unsaved user changes during the transition.

### Q5.2: Detecting a Dirty Working Directory
* Check working directory files against index metadata (size, mtime) to find local modifications.
* Hash the working file and compare it to the index hash if metadata differs.
* Compare the current index hashes against the target branch's root tree hashes.
* Refuse checkout if a modified file has different hashes in the current index versus the target tree.

### Q5.3: Detached HEAD Operations
* Commits update the `.pes/HEAD` file directly with a new hash instead of updating a branch reference file.
* These commits become unreachable once `HEAD` is pointed to another branch.
* Recover them by creating a new branch reference file in `.pes/refs/heads/` containing the detached commit's hash.

### Q6.1: Garbage Collection and Space Reclamation Algorithm
* Use a Mark-and-Sweep algorithm.
* Start from all references in `.pes/refs/heads/` and `.pes/HEAD`.
* Traverse and mark all reachable commits, their trees, and corresponding blobs.
* Delete any files in `.pes/objects/` that are not marked.
* Use a Hash Set to track reachable object IDs efficiently.
* Visiting 100,000 commits and 50 branches requires checking hundreds of thousands to millions of associated tree and blob objects.

### Q6.2: Garbage Collection Race Conditions
* A race condition occurs if a user stages a file (creating a blob) while GC is running.
* GC might scan the new blob before it is linked to a commit or reference, marking it as unreachable.
* GC deletes the newly staged blob, corrupting the subsequent commit.
* Git prevents this by enforcing a time-based grace period (e.g., pruning only unreachable objects older than 2 weeks).