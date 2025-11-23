# 012: Managing Processes in Linux

## 1. Viewing Processes

### `pstree`

Shows processes in a **tree hierarchy** (parent → child relationships).

```bash
pstree

```

### `ps`

Lists current processes. Common options:

```bash
ps fx          # Full-format listing with hierarchy
ps -C sleep    # Show all processes named "sleep"

```

### `top`

Interactive real-time process monitor.

- Press **`k`** → enter PID → **Enter** to kill a process.
- Press **`q`** to quit.

### `jobs`

Shows **background jobs** in the current shell (not system-wide processes).

```bash
jobs           # List jobs: [1]+ Running sleep 200 &

```

---

## 2. Sending Signals to Processes

### Understanding Signals

Processes communicate via **signals**. Common ones:

| Signal | Number | Purpose |
| --- | --- | --- |
| `SIGHUP` | 1 | Reload config (graceful restart) |
| `SIGTERM` | 15 | Request graceful shutdown (default for `kill`) |
| `SIGKILL` | 9 | Force kill (cannot be ignored) |
| `SIGSTOP` | 19 | Pause process |
| `SIGCONT` | 18 | Resume paused process |

### View All Signals

```bash
kill -l        # List all signal names/numbers

```

---

## 3. Killing Processes

### Basic Kill

```bash
sleep 100 &    # Start background process → note PID (e.g., 1234)
kill 1234      # Send SIGTERM (graceful shutdown)
kill -15 1234  # Explicit SIGTERM

```

### Force Kill (Unstoppable)

```bash
sleep 120 &
kill -9 1234   # SIGKILL: immediate termination (use only if SIGTERM fails)

```

### Kill by Name

```bash
sleep 130 &
pkill sleep    # Kill all processes named "sleep"
killall sleep  # Alternative to pkill

```

> ⚠️ Warning: killall kills all processes with that name (use cautiously!).
> 

---

## 4. Pausing and Resuming Processes

### Stop (Pause) a Process

```bash
sleep 140 &
kill -19 1234  # SIGSTOP: pauses process (appears as "T" in ps)
jobs           # Shows: [1]+ Stopped sleep 140

```

### Resume a Process

```bash
kill -18 1234  # SIGCONT: resumes paused process
# OR
fg 1           # Bring job #1 to foreground (resumes automatically)

```

### Lab Steps

1. Start a background job:
    
    ```bash
    sleep 160 &
    
    ```
    
2. Pause it:
    
    ```bash
    kill -19 %1    # %1 = job #1
    jobs           # Verify: "Stopped"
    
    ```
    
3. Resume:
    
    ```bash
    kill -18 %1    # Or: fg 1
    
    ```
    

---

## 5. Advanced Process Inspection

### Trace System Calls

Monitor a process’s kernel interactions:

```bash
sleep 120 &
strace -p 1234   # Attach to PID 1234
# Press Ctrl+C to stop tracing

```

> 🔍 Note: strace shows low-level system calls (e.g., file opens, network calls).
> 

---

## 6. Job Control (Foreground/Background)

### Key Commands

| Command | Purpose |
| --- | --- |
| `command &` | Run in background |
| `jobs` | List background jobs |
| `fg %1` | Bring job #1 to foreground |
| `Ctrl+Z` | Pause foreground job |
| `bg %1` | Resume job #1 in background |

### Example Workflow

```bash
sleep 130 &    # [1] 1234
sleep 120      # Run in foreground
# Press Ctrl+Z → [2]+ Stopped
jobs           # [1] Running, [2] Stopped
fg 1           # Resume job #1 in foreground

```

---

## Summary Cheat Sheet

| Task | Command |
| --- | --- |
| List processes | `ps fx`, `top`, `pstree` |
| List background jobs | `jobs` |
| Graceful kill | `kill PID` or `kill -15 PID` |
| Force kill | `kill -9 PID` |
| Kill by name | `pkill process_name`, `killall process_name` |
| Pause process | `kill -19 PID` |
| Resume process | `kill -18 PID` or `fg %job_number` |
| Trace system calls | `strace -p PID` |
| View signals | `kill -l` |

> 💡 Golden Rules:
> 
> - Always try `SIGTERM` (`kill PID`) before `SIGKILL` (`kill -9`).
> - Use `pkill`/`killall` **carefully**—they affect all matching processes.
> - `jobs` only works for processes started in **your current shell session**.
> - Paused processes (`SIGSTOP`) still consume memory—resume or kill them!