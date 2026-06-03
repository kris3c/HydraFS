# HydraFS — Distributed Network File System

A fault-tolerant distributed file system built in C, featuring a central Naming Server, multiple Storage Servers, and concurrent client support over TCP sockets.

---

## Architecture

```
                        ┌─────────────┐
                        │             │
          ┌─────────────│ Naming Server│─────────────┐
          │             │  (Port 2000) │             │
          │             └──────┬──────┘             │
          │                    │                    │
    ┌─────▼──────┐      ┌──────▼──────┐     ┌──────▼──────┐
    │  Storage   │      │   Storage   │     │   Storage   │
    │  Server 1  │      │   Server 2  │     │   Server 3  │
    │ NFS: 1500  │      │  NFS: 2500  │     │  NFS: 3500  │
    │ CLT: 1501  │      │  CLT: 2501  │     │  CLT: 3501  │
    └────────────┘      └─────────────┘     └─────────────┘
          ▲                    ▲                    ▲
          └────────────────────┼────────────────────┘
                               │
                        ┌──────▼──────┐
                        │   Client    │
                        └─────────────┘
```

**Naming Server (NS)** — Central coordinator. Handles path resolution, Storage Server registration, backups, and client routing. Listens on port `2000`.

**Storage Server (SS)** — Stores and serves actual files. Each SS registers with the NS on startup and exposes two ports: one for NS communication and one for direct client access.

**Client** — Sends file operation requests to the NS, which either handles them directly or routes to the appropriate SS.

---

## Features

- **Multi-client concurrency** — up to 100 simultaneous clients via pthreads
- **Fault tolerance** — if an SS goes down, its files remain accessible (read-only) via backup SSes
- **Automatic replication** — NS backs up each SS's data across two other SSes
- **LRU caching** — frequently accessed paths are cached at the NS (cache size: 20)
- **Trie-based path indexing** — efficient O(path length) lookup for accessible paths
- **Dynamic path updates** — SS detects newly added/removed files and notifies the NS
- **TCP socket communication** — all components communicate over TCP
- **Comprehensive logging** — all NS activity is logged; press `Ctrl+Z` to print logs
- **Error codes** — structured error handling (`404`, `200`, `600`-range, etc.)

---

## Supported Client Operations

| Command | Description |
|---|---|
| `READ <path>` | Read contents of a file |
| `WRITE <path> <data>` | Write data to a file |
| `APPEND <path> <data>` | Append data to a file |
| `CREATE FILE <path>` | Create a new file |
| `CREATE FOLDER <path>` | Create a new folder |
| `DELETE FILE <path>` | Delete a file |
| `DELETE FOLDER <path>` | Delete a folder |
| `COPY <src> <dest>` | Copy a file/folder between SSes |
| `LIST` | List all accessible paths |
| `INFO <path>` | Get file metadata (size, permissions, etc.) |
| `MAN` | Show full command reference |

---

## Prerequisites

```bash
sudo apt update
sudo apt install gcc make python3 gnome-terminal
```

---

## Building & Running

### 1. Clone the repository

```bash
git clone https://github.com/yourusername/HydraFS.git
cd HydraFS
```

### 2. Build the Naming Server

```bash
cd NS
make
cd ..
```

### 3. Build the Client

```bash
cd client
make
cd ..
```

### 4. Set up and launch Storage Servers

From the project root, run with your desired number of SSes (e.g. 3):

```bash
make n=3
```

This will:
- Create `SS1/`, `SS2/`, `SS3/` directories
- Assign each SS unique ports
- Compile each SS binary
- Launch each SS in a separate terminal window

> Each SSi gets ports: NFS = `1000*(i+1)+500`, Client = `1000*(i+1)+501`

### 5. Start the Naming Server

```bash
cd NS
./ns.o
```

### 6. Run the Client

```bash
cd client
./client.o
```

Type `MAN` at the client prompt to see all available commands.

---

## Project Structure

```
HydraFS/
├── NS/                     # Naming Server
│   ├── ns.c                # Main NS entry point
│   ├── threads.c           # Thread handlers (receive, backup, server)
│   ├── request_handler.c   # Client/SS request processing
│   ├── basic_ops.c         # Read, write, append operations
│   ├── create_delete.c     # Create and delete operations
│   ├── copy_handler.c      # Copy/paste across SSes
│   ├── tries.c             # Trie data structure for path indexing
│   ├── cache.c             # LRU cache implementation
│   ├── locks.c             # Path-level locking
│   ├── book_keeping.c      # Logging and monitoring
│   ├── headers.h           # Shared types, macros, constants
│   └── Makefile
├── SS/                     # Storage Server template
│   ├── ss.c                # Main SS entry point
│   ├── threads.c           # Thread handlers
│   ├── utils.c             # Utility functions
│   ├── headers.h           # SS-specific constants
│   └── makefile
├── client/                 # Client
│   ├── client.c            # Main client entry point
│   ├── read.c              # Read operation
│   ├── write_append.c      # Write/append operations
│   ├── create.c            # Create operation
│   ├── delete.c            # Delete operation
│   ├── copy.c              # Copy operation
│   ├── info.c              # File info operation
│   ├── list.c              # List paths
│   ├── connection.c        # NS connection setup
│   ├── man.c               # MAN page
│   ├── headers.h           # Client constants
│   └── makefile
├── setup_ss.py             # Creates and compiles SS instances
├── start_ss.py             # Launches SS instances in new terminals
└── makefile                # Root makefile (runs both scripts)
```

---

## Configuration

Key constants in `NS/headers.h`:

| Macro | Default | Description |
|---|---|---|
| `NS_PORT` | `2000` | Naming Server port |
| `MAX_CONNECTIONS` | `100` | Max concurrent clients |
| `CACHE_SIZE` | `20` | LRU cache size |
| `MAX_FILES` | `50` | Max paths per SS |

---

## Cleanup

To remove all generated SS folders and compiled binaries:

```bash
make clean n=3   # Use the same n as setup
```

---

## Known Limitations

- Only `.txt` (text) files are supported
- All file/folder names must be unique across the entire system
- Max data per write request is bounded by packet size
- If all SSes holding a file (original + backups) go down simultaneously, the file is inaccessible

---

## Tech Stack

- **Language:** C
- **Concurrency:** POSIX threads (`pthreads`), semaphores, mutexes
- **Networking:** TCP sockets (`sys/socket.h`)
- **Data Structures:** Trie (path indexing), LRU Cache, Linked Lists
- **Build:** GCC, Make, Python 3

---

## Bugs Fixed During Development

- Corrected `pthread_create` function signatures (`void*()` → `void*(void*)`)
- Fixed `accept()` call using wrong type for `socklen_t` argument
- Fixed SS binary name mismatch in launch script (`ss.o` → `ss9.o`)
- Fixed OS target in `start_ss.py` for Linux compatibility
