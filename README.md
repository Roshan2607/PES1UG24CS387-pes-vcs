# Building PES-VCS — A Version Control System from Scratch

**Objective:** Build a local version control system that tracks file changes, stores snapshots efficiently, and supports commit history. Every component maps directly to operating system and filesystem concepts.

**Platform:** Ubuntu 22.04

---

## Getting Started

### Prerequisites

```bash
sudo apt update && sudo apt install -y gcc build-essential libssl-dev
```

### Using This Repository

This is a **template repository**. Do **not** fork it.

1. Click **"Use this template"** → **"Create a new repository"** on GitHub
2. Name your repository (e.g., `SRN-pes-vcs`) and set it to **public**. Replace `SRN` with your actual SRN, e.g., `PESXUG24CSYYY-pes-vcs`
3. Clone this repository to your local machine and do all your lab work inside this directory.
4.  **Important:** Remember to commit frequently as you progress. You are required to have a minimum of 5 detailed commits per phase. Refer to [Submission Requirements](#submission-requirements) for more details.
5. Clone your new repository and start working

The repository contains skeleton source files with `// TODO` markers where you need to write code. Functions marked `// PROVIDED` are complete — do not modify them.

### Building

```bash
make          # Build the pes binary
make all      # Build pes + test binaries
make clean    # Remove all build artifacts
```

### Author Configuration

PES-VCS reads the author name from the `PES_AUTHOR` environment variable:

```bash
export PES_AUTHOR="Your Name <PESXUG24CS042>"
```

If unset, it defaults to `"PES User <pes@localhost>"`.

### File Inventory

| File               | Role                                 | Your Task                                          |
| ------------------ | ------------------------------------ | -------------------------------------------------- |
| `pes.h`            | Core data structures and constants   | Do not modify                                      |
| `object.c`         | Content-addressable object store     | Implement `object_write`, `object_read`            |
| `tree.h`           | Tree object interface                | Do not modify                                      |
| `tree.c`           | Tree serialization and construction  | Implement `tree_from_index`                        |
| `index.h`          | Staging area interface               | Do not modify                                      |
| `index.c`          | Staging area (text-based index file) | Implement `index_load`, `index_save`, `index_add`  |
| `commit.h`         | Commit object interface              | Do not modify                                      |
| `commit.c`         | Commit creation and history          | Implement `commit_create`                          |
| `pes.c`            | CLI entry point and command dispatch | Do not modify                                      |
| `test_objects.c`   | Phase 1 test program                 | Do not modify                                      |
| `test_tree.c`      | Phase 2 test program                 | Do not modify                                      |
| `test_sequence.sh` | End-to-end integration test          | Do not modify                                      |
| `Makefile`         | Build system                         | Do not modify                                      |

---

## Understanding Git: What You're Building

Before writing code, understand how Git works under the hood. Git is a content-addressable filesystem with a few clever data structures on top. Everything in this lab is based on Git's real design.

### The Big Picture

When you run `git commit`, Git doesn't store "changes" or "diffs." It stores **complete snapshots** of your entire project. Git uses two tricks to make this efficient:

1. **Content-addressable storage:** Every file is stored by the SHA hash of its contents. Same content = same hash = stored only once.
2. **Tree structures:** Directories are stored as "tree" objects that point to file contents, so unchanged files are just pointers to existing data.

```
Your project at commit A:          Your project at commit B:
                                   (only README changed)

    root/                              root/
    ├── README.md  ─────┐              ├── README.md  ─────┐
    ├── src/            │              ├── src/            │
    │   └── main.c ─────┼─┐            │   └── main.c ─────┼─┐
    └── Makefile ───────┼─┼─┐          └── Makefile ───────┼─┼─┐
                        │ │ │                              │ │ │
                        ▼ ▼ ▼                              ▼ ▼ ▼
    Object Store:       ┌─────────────────────────────────────────────┐
                        │  a1b2c3 (README v1)    ← only this is new   │
                        │  d4e5f6 (README v2)                         │
                        │  789abc (main.c)       ← shared by both!    │
                        │  fedcba (Makefile)     ← shared by both!    │
                        └─────────────────────────────────────────────┘
```

### The Three Object Types

#### 1. Blob (Binary Large Object)

A blob is just file contents. No filename, no permissions — just the raw bytes.

```
blob 16\0Hello, World!\n
     ↑    ↑
     │    └── The actual file content
     └─────── Size in bytes
```

The blob is stored at a path determined by its SHA-256 hash. If two files have identical contents, they share one blob.

#### 2. Tree

A tree represents a directory. It's a list of entries, each pointing to a blob (file) or another tree (subdirectory).

```
100644 blob a1b2c3d4... README.md
100755 blob e5f6a7b8... build.sh        ← executable file
040000 tree 9c0d1e2f... src             ← subdirectory
       ↑    ↑           ↑
       │    │           └── name
       │    └── hash of the object
       └─────── mode (permissions + type)
```

Mode values:
- `100644` — regular file, not executable
- `100755` — regular file, executable
- `040000` — directory (tree)

#### 3. Commit

A commit ties everything together. It points to a tree (the project snapshot) and contains metadata.

```
tree 9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d
parent a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
author Alice <alice@example.com> 1699900000
committer Alice <alice@example.com> 1699900000

Add new feature
```

The parent pointer creates a linked list of history:

```
    C3 ──────► C2 ──────► C1 ──────► (no parent)
    │          │          │
    ▼          ▼          ▼
  Tree3      Tree2      Tree1
```

### How Objects Connect

```
                    ┌─────────────────────────────────┐
                    │           COMMIT                │
                    │  tree: 7a3f...                  │
                    │  parent: 4b2e...                │
                    │  author: Alice                  │
                    │  message: "Add feature"         │
                    └─────────────┬───────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────────────┐
                    │         TREE (root)             │
                    │  100644 blob f1a2... README.md  │
                    │  040000 tree 8b3c... src        │
                    │  100644 blob 9d4e... Makefile   │
                    └──────┬──────────┬───────────────┘
                           │          │
              ┌────────────┘          └────────────┐
              ▼                                    ▼
┌─────────────────────────┐          ┌─────────────────────────┐
│      TREE (src)         │          │     BLOB (README.md)    │
│ 100644 blob a5f6 main.c │          │  # My Project           │
└───────────┬─────────────┘          └─────────────────────────┘
            ▼
       ┌────────┐
       │ BLOB   │
       │main.c  │
       └────────┘
```

### References and HEAD

References are files that map human-readable names to commit hashes:

```
.pes/
├── HEAD                    # "ref: refs/heads/main"
└── refs/
    └── heads/
        └── main            # Contains: a1b2c3d4e5f6...
```

**HEAD** points to a branch name. The branch file contains the latest commit hash. When you commit:

1. Git creates the new commit object (pointing to parent)
2. Updates the branch file to contain the new commit's hash
3. HEAD still points to the branch, so it "follows" automatically

```
Before commit:                    After commit:

HEAD ─► main ─► C2 ─► C1         HEAD ─► main ─► C3 ─► C2 ─► C1
```

### The Index (Staging Area)

The index is the "preparation area" for the next commit. It tracks which files are staged.

```
Working Directory          Index               Repository (HEAD)
─────────────────         ─────────           ─────────────────
README.md (modified) ──── pes add ──► README.md (staged)
src/main.c                            src/main.c          ──► Last commit's
Makefile                               Makefile                snapshot
```

The workflow:

1. `pes add file.txt` → computes blob hash, stores blob, updates index
2. `pes commit -m "msg"` → builds tree from index, creates commit, updates branch ref

### Content-Addressable Storage

Objects are named by their content's hash:

```python
# Pseudocode
def store_object(content):
    hash = sha256(content)
    path = f".pes/objects/{hash[0:2]}/{hash[2:]}"
    write_file(path, content)
    return hash
```

This gives us:
- **Deduplication:** Identical files stored once
- **Integrity:** Hash verifies data isn't corrupted
- **Immutability:** Changing content = different hash = different object

Objects are sharded by the first two hex characters to avoid huge directories:

```
.pes/objects/
├── 2f/
│   └── 8a3b5c7d9e...
├── a1/
│   ├── 9c4e6f8a0b...
│   └── b2d4f6a8c0...
└── ff/
    └── 1234567890...
```

### Exploring a Real Git Repository

You can inspect Git's internals yourself:

```bash
mkdir test-repo && cd test-repo && git init
echo "Hello" > hello.txt
git add hello.txt && git commit -m "First commit"

find .git/objects -type f          # See stored objects
git cat-file -t <hash>            # Show type: blob, tree, or commit
git cat-file -p <hash>            # Show contents
cat .git/HEAD                     # See what HEAD points to
cat .git/refs/heads/main          # See branch pointer
```
---

# LAB REPORT

## Analysis Questions

### Phase 5: Branching and Checkout

**Q5.1: Given this, how would you implement `pes checkout <branch>` — what files need to change in `.pes/`, and what must happen to the working directory? What makes this operation complex?**
*   **`.pes/` Changes:** We need to update `.pes/HEAD` to point to the new branch using standard `open()`, `write()`, and `close()` system calls. The staging area (`.pes/index`) also needs to be updated to match the new branch's tree.
*   **Working Directory:** We have to traverse the target commit's tree, much like parsing a directory structure, and update the physical files in our workspace to match.
*   **Complexity:** The real complexity lies in file protection. If the user has an open file they are actively modifying, doing a blind checkout would overwrite their work. We have to carefully use the `stat()` system call to check file metadata before replacing anything, ensuring we don't accidentally corrupt the user's unsaved workspace.

**Q5.2: Describe how you would detect this "dirty working directory" conflict using only the index and the object store.**
*   This concept is very similar to how a dirty bit works in page replacement algorithms. 
*   We can use the `stat()` or `lstat()` system calls on every file in the working directory to retrieve its attributes (like modification time and size). 
*   By comparing these attributes to the metadata stored in our `.pes/index`, we can tell if the file has been modified since it was last staged. If the file is dirty, and checking out the new branch would overwrite those local modifications, the system must block the checkout operation to prevent data loss.

**Q5.3: "Detached HEAD" means HEAD contains a commit hash directly instead of a branch reference. What happens if you make commits in this state? How could a user recover those commits?**
*   **What Happens:** The commit is created in the object store, but no branch file is updated to point to it. If you switch away to another branch, it essentially becomes an orphan process. In file system terms, it's like a file inode that has a link count of 0 because no directory entry points to it anymore. It's lost in the void.
*   **Recovery:** To recover it, the user needs to create a new branch pointing to that orphaned commit's hash. This is basically applying the `link()` system call concept to create a new hard link to the commit, bringing its reference count back up so it's reachable again.

### Phase 6: Garbage Collection and Space Reclamation

**Q6.1: Describe an algorithm to find and delete these objects. What data structure would you use to track "reachable" hashes efficiently? For a repository with 100,000 commits and 50 branches, estimate how many objects you'd need to visit.**
*   **Algorithm:** We can use a Mark-and-Sweep approach to manage our mass-storage space. First, we start at the branch files and traverse the pointers down through all the commits, trees, and blobs, keeping track of every hash we find in main memory. Then, we use directory-reading system calls to scan `.pes/objects`. If a file isn't in our reachable list, we delete it to free up disk space.
*   **Data Structure:** A Hash Table in main memory is best for fast lookups. However, if the data structure gets too large for physical memory, the OS will resort to demand paging. If we aren't careful with memory allocation while scanning huge directories, it could lead to thrashing.
*   **Estimate:** We have to visit all 100,000 commit objects, plus at least 100,000 root trees, and all historically unique blobs. We'd easily be visiting hundreds of thousands of files.

**Q6.2: Why is it dangerous to run garbage collection concurrently with a commit operation? Describe a race condition where GC could delete an object that a concurrent commit is about to reference. How does Git's real GC avoid this?**
*   **Race Condition:** This creates a classic critical-section problem. Suppose the commit process creates a new blob on disk, but before it can link it to a tree, a context switch happens. The GC thread runs, sees that the new blob has no references, and deletes it. When the commit process resumes, it saves a tree pointing to a deleted file, resulting in a corrupted file system.
*   **How Git avoids this:** Normally, we'd use mutex locks or semaphores to ensure mutual exclusion while writing objects. However, locking the whole repository would create massive bottlenecks. Instead, real Git avoids this concurrency issue by using a grace period—it simply refuses to garbage collect any files created within the last 14 days, naturally side-stepping the synchronization problem.

---

## Screenshots

### 1A: `./test_objects` output
<img width="940" height="289" alt="image" src="https://github.com/user-attachments/assets/a0767b4b-4cc2-4d18-8006-69e749fbcd68" />


### 1B: `find .pes/objects` output
<img width="940" height="257" alt="image" src="https://github.com/user-attachments/assets/8b5896c8-779a-4cae-bc07-a79d692fc7ce" />


### 2A: `./test_tree` output
<img width="940" height="346" alt="image" src="https://github.com/user-attachments/assets/874e6b68-7065-4e9c-ba35-6e24e1fdd1eb" />


### 2B: `xxd` of a raw tree object
<img width="940" height="136" alt="image" src="https://github.com/user-attachments/assets/80c1f4ab-4735-4502-9518-1750d38e529c" />


### 3A: `pes init` → `pes add` → `pes status`
<img width="940" height="749" alt="image" src="https://github.com/user-attachments/assets/dbff9988-4f8a-40ec-8ace-03a6e21fcd83" />



### 3B: `cat .pes/index`
<img width="940" height="201" alt="image" src="https://github.com/user-attachments/assets/72d40ce8-4a6a-4182-a39c-7bb6d93ab908" />



### 4A: `pes log`
<img width="940" height="469" alt="image" src="https://github.com/user-attachments/assets/5132914b-1da4-4bfe-a65c-75ce49af875c" />



### 4B: `find .pes -type f | sort`
<img width="940" height="430" alt="image" src="https://github.com/user-attachments/assets/daeed45b-13ad-4909-8139-9d0f57406703" />

<img width="940" height="489" alt="image" src="https://github.com/user-attachments/assets/69f11104-4cf7-4a3b-a39d-e1c7ee0bab63" />


### 4C: `cat .pes/refs/heads/main` and `cat .pes/HEAD`
<img width="1783" height="522" alt="image" src="https://github.com/user-attachments/assets/738d98a9-1745-4a63-959a-efdfc34f2c22" />


### Final: Full integration test
<img width="1914" height="2277" alt="image" src="https://github.com/user-attachments/assets/68db2986-8dcc-400b-8aef-a0ffc3e5e4ec" />

