# 08: Shell Variables, Embedding, Options & History in Linux

## 1. Types of Shell Variables

### Explanation

- **System-defined variables**: Pre-set by the system (usually uppercase). Examples: `$USER`, `$HOME`, `$SHELL`.
- **User-defined variables**: Created by the user (case-sensitive, often lowercase).

### Commands

```bash
# System variables
echo $USER
echo $SHELL
echo $PWD
echo $HOSTNAME

# View all variables
set

# User-defined variable
var1=555
echo $var1
unset var1        # Delete variable
echo $var1        # Now empty

```

### Lab Steps

1. Check system info:
    
    ```bash
    echo "User: $USER, Shell: $SHELL"
    
    ```
    
2. Create and delete a variable:
    
    ```bash
    myvar=hello
    echo $myvar
    unset myvar
    echo $myvar
    
    ```
    

### Tips & Warnings

- Always use `$` to **reference** a variable.
- Variable names are case-sensitive: `VAR ≠ var`.

---

## 2. The `PATH` Variable

### Explanation

`PATH` is a colon-separated list of directories where the shell looks for **executable commands**. When you type `ls`, the shell searches each directory in `PATH` for a file named `ls`.

### Commands

```bash
echo $PATH
which ls          # Shows full path: /usr/bin/ls

```

### Lab Steps

1. View your `PATH`:
    
    ```bash
    echo $PATH
    
    ```
    
2. Try running a command by full path:
    
    ```bash
    /usr/bin/ls
    
    ```
    

### Tips & Warnings

- Add custom directories to `PATH` in `~/.bashrc`:
    
    ```bash
    export PATH=$PATH:/my/custom/bin
    
    ```
    

---

## 3. Environment Variables & `export`

### Explanation

- **Local variables** exist only in the current shell.
- **Environment variables** (exported) are passed to **child processes** (e.g., new shells, scripts).

### Commands

```bash
var1=one
var2=two
export var2       # Make var2 available to child shells

zsh               # Start new shell
echo $var1 $var2  # Only var2 appears
exit

export var1       # Now export var1
zsh
echo $var1 $var2  # Both appear

```

### View Environment Variables

```bash
env               # List all exported variables
env -i bash       # Start clean shell (no env vars)

```

### Lab Steps

1. Test variable inheritance:
    
    ```bash
    testvar=local
    export testenv=global
    bash -c 'echo "Local: $testvar, Env: $testenv"'
    
    ```
    

---

## 4. Variable Expansion & Delimiting

### Explanation

Use `${var}` to **delimit** variable names when adjacent to text.

### Examples

```bash
prefix=Super
echo Hello $prefixman        # Fails (looks for $prefixman)
echo Hello ${prefix}man      # Correct: "Hello Superman"

```

### Lab Steps

```bash
animal=cat
echo The${animal}s           # Output: Thecats

```

---

## 5. Handling Unset Variables (`set -u`)

### Explanation

- By default, referencing an unset variable returns **empty**.
- `set -u` makes the shell **exit with error** on unset variables (good for debugging scripts).

### Commands

```bash
echo $MayVar      # Empty (no error)
set -u
echo $MayVar      # Error: "MayVar: unbound variable"
MayVar=22
echo $MayVar      # Works
set +u            # Disable strict mode

```

### Tips & Warnings

- Always use `set -u` in production scripts to catch typos.

---

## 6. Command Substitution: `$()` and Backticks

### Explanation

Run a command inside `$()` or ``...`` and **capture its output** as text.

### Syntax

```bash
# Modern (preferred)
output=$(command)

# Legacy (backticks)
output=`command`

```

### Examples

```bash
echo $(date)
echo `date`

# Nested substitution
A=shell
echo $(B=sub; echo $A $B)

```

### Lab Steps

1. Capture directory listing:
    
    ```bash
    files=$(ls /etc/pass*)
    echo $files
    
    ```
    

### Tips & Warnings

- **Prefer `$()`**—it supports nesting; backticks do not.
- Avoid backticks in new scripts.

---

## 7. Shell Options (`set`)

### Explanation

Shell options control behavior. View active options with `echo $-`.

### Common Options

| Option | Effect |
| --- | --- |
| `set -x` | Enable command tracing (debug mode) |
| `set +x` | Disable tracing |
| `set -u` | Exit on unset variables |
| `set -C` | Prevent overwriting files with `>` |

### Examples

```bash
echo $-        # View current options (e.g., "himBHs")
set -x
echo "Debug on"
set +x

```

---

## 8. Shell History

### Explanation

The shell records commands in a **history list** for reuse.

### Commands

```bash
history          # Show full history
history 10       # Show last 10 commands
!!               # Re-run last command
!3               # Re-run command #3 from history
!to              # Re-run last command starting with "to" (e.g., "touch")

# Search interactively
# Press Ctrl+R, then type part of command (e.g., "cal")

```

### History Variables

```bash
echo $HISTSIZE        # Max commands in memory (default: 500–1000)
echo $HISTFILE        # File where history is saved (~/.bash_history)
echo $HISTFILESIZE    # Max lines in history file

```

### Exclude Commands from History

- Prefix command with a **space**:
    
    ```bash
     secret_command    # Won't appear in history
    
    ```
    

### Clear History

```bash
history -c    # Clear current session history

```

### Lab Steps

1. Run a few commands:
    
    ```bash
    date
    pwd
    ls
    
    ```
    
2. Re-run `date` with `!d`.
3. Search with **Ctrl+R** → type "ls".

---

## Summary Cheat Sheet

| Topic | Key Command |
| --- | --- |
| View system vars | `echo $USER $SHELL` |
| Create variable | `var=value` |
| Export variable | `export var` |
| Safe expansion | `echo ${var}text` |
| Strict mode | `set -u` |
| Command substitution | `out=$(command)` |
| Debug mode | `set -x` |
| Re-run last command | `!!` |
| Search history | **Ctrl+R** |
| Hide from history | `space_command` |

> 💡 Golden Rule:
> 
> - Use `export` to share variables with child processes.
> - Use `${var}` to avoid ambiguity.
> - Use `set -u` and `set -x` for safer, debuggable scripts.