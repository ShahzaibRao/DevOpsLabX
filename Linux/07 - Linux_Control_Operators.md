# 07: Control Operators in Linux Shell

## 1. Sequential Execution with `;`

### Explanation

The semicolon (`;`) allows you to **run multiple commands one after another**, regardless of whether each command succeeds or fails. Commands execute **sequentially** (one at a time).

### Syntax

```bash
command1; command2; command3

```

### Examples

```bash
date; pwd; cal
echo Hello; echo World

```

### Lab Steps

1. Open a terminal.
2. Run:
    
    ```bash
    date; pwd; whoami
    
    ```
    
3. Observe that all three commands run in order, even if one fails.

### Tips & Warnings

- Use `;` when you want commands to run **independently**.
- Unlike `&&`, it **does not check** if the previous command succeeded.

---

## 2. Background Execution with `&`

### Explanation

The ampersand (`&`) runs a command in the **background**, allowing the next command to start **immediately** (in parallel). This is useful for long-running tasks.

### Syntax

```bash
command1 & command2

```

### Examples

```bash
sleep 10 & date          # 'date' runs immediately while 'sleep' runs in background
sleep 20 &               # Runs sleep in background and returns prompt

```

### Lab Steps

1. Run a background task:
    
    ```bash
    sleep 15 &
    
    ```
    
2. Note the job ID and PID (e.g., `[1] 1234`).
3. Run another command immediately:
    
    ```bash
    date
    
    ```
    
4. Wait for background job to finish (you’ll see a notification like `[1]+ Done`).

### Tips & Warnings

- Background jobs continue running even if you close the terminal (unless you use `nohup` or `disown`).
- Use `jobs` to list background jobs, `fg` to bring one to foreground.
- **Don’t confuse** `&` (background) with `&&` (logical AND).

---

## 3. Exit Status and `$?`

### Explanation

Every command returns an **exit status** (a number from 0 to 255):

- **`0`** = Success
- **Non-zero** = Failure (specific number may indicate error type)

The special variable `$?` holds the exit status of the **last executed command**.

### Commands

```bash
echo $USER      # Valid command → exit status 0
echo $?         # Outputs: 0

zacall          # Invalid command → exit status non-zero
echo $?         # Outputs: 127 (or similar)

```

### Lab Steps

1. Run a valid command:
    
    ```bash
    ls /tmp
    echo $?   # Should be 0
    
    ```
    
2. Run an invalid command:
    
    ```bash
    fakecommand
    echo $?   # Non-zero (e.g., 127)
    
    ```
    

### Tips & Warnings

- Always check `$?` **immediately** after the command—it gets overwritten by the next command.
- Useful in scripts for error handling.

---

## 4. Logical AND with `&&`

### Explanation

`command1 && command2` means:

**"Run `command2` ONLY if `command1` succeeds (exit status 0)."**

### Examples

```bash
echo Hello && echo World        # Both run (first succeeds)
zcho Hello && echo World        # Only 'zcho' runs (fails), 'echo World' is skipped

```

### Lab Steps

1. Test success:
    
    ```bash
    mkdir testdir && echo "Directory created"
    
    ```
    
2. Test failure:
    
    ```bash
    mkdir /root/test && echo "This won't print"  # Fails (permission denied)
    
    ```
    

### Tips & Warnings

- Great for **conditional execution** (e.g., "download && install").
- Safer than `;` when commands depend on each other.

---

## 5. Logical OR with `||`

### Explanation

`command1 || command2` means:

**"Run `command2` ONLY if `command1` fails (non-zero exit status)."**

### Examples

```bash
echo Hello || echo World        # Only 'echo Hello' runs (succeeds)
zcho Hello || echo World        # 'zcho' fails → 'echo World' runs

```

### Lab Steps

1. Test failure case:
    
    ```bash
    ls /nonexistent || echo "Path not found"
    
    ```
    
2. Combine with `&&`:
    
    ```bash
    rm file1 && echo "Deleted" || echo "Failed to delete"
    
    ```
    

### Tips & Warnings

- Often used for **fallback actions** or error messages.
- Be careful with complex chains—use parentheses if needed.

---

## 6. Combining `&&` and `||` for Conditional Logic

### Explanation

You can chain `&&` and `||` to create **if-then-else** logic in one line.

### Example

```bash
touch file1
ls -l file1
rm file1 && echo "It worked" || echo "It failed!"
rm file1 && echo "It worked" || echo "It failed!"  # Second run: file doesn't exist

```

### Lab Steps

1. Create a file:
    
    ```bash
    touch myfile.txt
    
    ```
    
2. Delete and report:
    
    ```bash
    rm myfile.txt && echo "Success" || echo "Failure"
    
    ```
    
3. Run again (file is gone):
    
    ```bash
    rm myfile.txt && echo "Success" || echo "Failure"  # Now shows "Failure"
    
    ```
    

### Tips & Warnings

- This mimics: `if command; then ... else ... fi`
- **Caution**: If the "success" command fails, the `||` part will run!
Example: `cmd && echo "OK" || echo "BAD"` → if `echo` fails (rare), "BAD" prints.

---

## 7. Comments with `#`

### Explanation

The hash (`#`) marks the rest of the line as a **comment**. The shell ignores it.

### Examples

```bash
echo hello world > file.txt        # Writes to file
echo "#hello world" > file.txt     # Writes literal #hello world to file
# sh file.txt                      # This line is ignored

```

> Note: Only works at the start of a token. Inside quotes, # is literal.
> 

### Lab Steps

1. Create a script file:
    
    ```bash
    echo '#!/bin/bash' > test.sh
    echo '# This is a comment' >> test.sh
    echo 'echo "Hello"' >> test.sh
    
    ```
    
2. Run it:
    
    ```bash
    bash test.sh
    
    ```
    

---

## 8. Line Continuation with `\\`

### Explanation

The backslash (`\\`) at the end of a line **continues the command on the next line**. Useful for long commands.

### Examples

```bash
echo This is Linux \\
> class \\
> start new world

apt-get update \\
&& apt-get upgrade \\
&& apt-get install python3

```

### Lab Steps

1. Run a multi-line echo:
    
    ```bash
    echo Welcome to \\
    Linux \\
    class
    
    ```
    
2. Simulate a package install (don’t run if not needed):
    
    ```bash
    echo "Update" && \\
    echo "Upgrade" && \\
    echo "Install"
    
    ```
    

### Tips & Warnings

- **No space** after `\\`—it must be the very last character on the line.
- Common in scripts for readability.

---

## Summary Cheat Sheet

| Operator | Purpose | Example |
| --- | --- | --- |
| `;` | Run commands sequentially | `date; pwd` |
| `&` | Run command in background | `sleep 10 &` |
| `$?` | Get exit status of last command | `echo $?` |
| `&&` | Run next **only if success** | `mkdir dir && cd dir` |
| ` |  | ` |
| `#` | Comment (ignored by shell) | `# This is a note` |
| `\\` | Continue command on next line | `apt update \\ && apt upgrade` |

> 💡 Golden Rule:
> 
> - Use `&&` for **dependent** commands.
> - Use `;` for **independent** commands.
> - Always check `$?` after critical operations in scripts.