# 06: Commands and Arguments in Linux Shell


## 1. Understanding Shell Expansion and Quoting

### Explanation

The shell processes (or "expands") commands before executing them. By default, **multiple spaces between words are collapsed into a single space**. To preserve spacing or special characters, you must use **quoting**.

### Commands and Examples

```bash
echo Hello world                 # Output: Hello world
echo Hello           world       # Output: Hello world (spaces collapsed)
echo 'Hello           world'    # Output: Hello           world (single quotes preserve everything)
echo "Hello           world"    # Output: Hello           world (double quotes also preserve spaces)

```

### Lab Steps

1. Run each of the above `echo` commands.
2. Observe how unquoted spaces are collapsed, but quoted ones are preserved.

### Tips & Warnings

- **Single quotes (`'...'`)**: Preserve **everything literally**—no variable expansion or escape sequences.
- **Double quotes (`"..."`)**: Allow **variable expansion** and **escape sequences** (like `\\n`, `\\t`), but preserve spaces.
- Always quote variables: `echo "$var"` instead of `echo $var` to avoid word splitting.

---

## 2. Shell Variables: System vs. User-Defined

### Explanation

- **System (Environment) Variables**: Predefined by the system (e.g., `$SHELL`, `$HOME`). Usually uppercase.
- **User-Defined Variables**: Created by the user. Not automatically available to child processes unless exported.

### Commands

```bash
echo $SHELL        # Shows your current shell (e.g., /bin/bash)

# Create a user variable
var1=100
echo $var1         # Correct: outputs 100
echo var1          # Wrong: outputs literal "var1"
echo 'var1'        # Outputs literal "var1" (no expansion in single quotes)
echo "var1"        # Outputs literal "var1" (no $, so no expansion)

```

### Lab Steps

1. Check your shell:
    
    ```bash
    echo $SHELL
    
    ```
    
2. Create and test a variable:
    
    ```bash
    myvar=42
    echo $myvar
    echo myvar
    
    ```
    
3. Try quoting:
    
    ```bash
    echo "$myvar"    # Works
    echo '$myvar'    # Outputs literal $myvar
    
    ```
    

### Tips & Warnings

- Always use `$` to **reference** a variable (e.g., `$var1`).
- To make a user variable available to child processes, use `export var1`.

---

## 3. Escape Sequences with `echo -e`

### Explanation

The `-e` option enables interpretation of **backslash escape sequences** like:

- `\\n` = newline
- `\\t` = tab
- `\\\\` = literal backslash

### Command

```bash
echo -e "Welcome \\n to \\t Linux"

```

### Output

```
Welcome
 to 	Linux

```

### Lab Steps

1. Run:
    
    ```bash
    echo -e "Line 1\\nLine 2\\tIndented"
    
    ```
    
2. Compare with without `e`:
    
    ```bash
    echo "Line 1\\nLine 2\\tIndented"  # Prints \\n and \\t literally
    
    ```
    

### Tips & Warnings

- Not all shells support `echo -e` (though Bash does). For portability, consider `printf`:
    
    ```bash
    printf "Welcome \\n to \\t Linux\\n"
    
    ```
    

---

## 4. Built-in vs. External Commands

### Explanation

- **Built-in Commands**: Part of the shell itself (e.g., `cd`, `export`, `alias`). Faster and can modify shell state.
- **External Commands**: Separate executable files stored in directories like `/bin`, `/usr/bin`, or `/sbin`.

### Commands to Identify Command Type

```bash
type cd          # Output: cd is a shell builtin
type ls          # Output: ls is /usr/bin/ls
which ls         # Output: /usr/bin/ls (path to external command)

```

### Why It Matters

- Built-ins like `cd` **must** be built-in—external programs can’t change the parent shell’s working directory.
- External commands require their directory to be in your `PATH` environment variable.

### Lab Steps

1. Check command types:
    
    ```bash
    type pwd
    type echo
    which grep
    
    ```
    
2. Run an external command by full path:
    
    ```bash
    /usr/bin/ls /tmp
    
    ```
    

### Tips & Warnings

- If a command isn’t found, check if it’s installed and if its path is in `PATH`:
    
    ```bash
    echo $PATH
    
    ```
    
- Use `type` instead of `which`—it works for both built-ins and externals.

---

## 5. Using Aliases

### Explanation

An **alias** creates a shortcut for a command. Useful for long or frequently used commands.

### Commands

```bash
alias cat=dog                # Creates alias (but 'dog' isn't a real command!)
dog filename                 # Fails unless 'dog' exists
alias ll='ls -l'             # Useful example
alias                            # List all aliases
unalias dog                  # Remove the 'dog' alias
unalias ll                   # Remove 'll' alias

```

### Lab Steps

1. Create a safe alias:
    
    ```bash
    alias ll='ls -l'
    ll /tmp
    
    ```
    
2. List aliases:
    
    ```bash
    alias
    
    ```
    
3. Remove it:
    
    ```bash
    unalias ll
    
    ```
    

### Tips & Warnings

- Aliases are **not** inherited by subshells or scripts.
- Avoid overriding critical commands (e.g., `alias rm='rm -i'` is common, but can be dangerous if you rely on it).
- Typos like `alias cat=dog` won’t break anything—but `dog filename` will fail.

---

## 6. Debugging Shell Expansion with `set -x`

### Explanation

Use `set -x` to **enable debug mode**, which shows exactly how the shell expands and executes your commands. Use `set +x` to disable it.

### Commands

```bash
date             # Normal output
set -x           # Enable tracing
date             # Now shows: + date
set +x           # Disable tracing

```

### Lab Steps

1. Enable tracing:
    
    ```bash
    set -x
    echo $SHELL
    ls /tmp
    
    ```
    
2. Observe the `+` prefix showing expanded commands.
3. Turn it off:
    
    ```bash
    set +x
    
    ```
    

### Tips & Warnings

- Extremely useful for debugging scripts.
- Output can be verbose—use selectively.
- In scripts, add `set -x` at the top to trace the entire script.

---

## Summary Cheat Sheet

| Concept | Command | Purpose |
| --- | --- | --- |
| Preserve spaces | `echo 'text'` or `echo "text"` | Prevent shell word splitting |
| Use variable | `echo $var` | Reference variable value |
| Escape sequences | `echo -e "Hello\\nWorld"` | Interpret `\\n`, `\\t`, etc. |
| Check command type | `type command` | Built-in or external? |
| Find external path | `which command` | Full path to executable |
| Create alias | `alias ll='ls -l'` | Shortcut for commands |
| Remove alias | `unalias ll` | Delete an alias |
| Debug shell | `set -x` / `set +x` | Trace command expansion |

> 💡 Golden Rule: When in doubt about how the shell interprets your command, use set -x to see the real expansion. And always quote your variables!
>