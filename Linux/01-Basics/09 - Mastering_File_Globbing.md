# 09: File Globbing (Wildcard Expansion) in Linux


## 1. What is Globbing?

### Explanation

**Globbing** (or **pathname expansion**) is the shell’s way of matching filenames using **wildcard characters**. The shell expands these patterns into matching filenames **before** passing them to commands.

> ⚠️ Important: Globbing is case-sensitive and depends on your system’s locale (language settings).
> 

---

## 2. Common Globbing Patterns

### Basic Wildcards

| Pattern | Meaning | Example |
| --- | --- | --- |
| `*` | Matches **zero or more** characters | `File*` → `FileA`, `File1`, `FileRR` |
| `?` | Matches **exactly one** character | `File?` → `FileA`, `File1` (not `File25`) |
| `[...]` | Matches **one character** from the set | `File[AB]` → `FileA`, `FileB` |

### Examples from Your Notes

```bash
# Create test files
touch fileA fileB fileC fileD FileA FileB FileC File1 File2 File3 File25 FileRR Filez5 Filef2

# Test patterns
ls File*          # All files starting with "File"
ls file*          # Only lowercase "file..." (case-sensitive!)
ls *ile*          # Any file containing "ile"
ls f*A            # Starts with "f", ends with "A"
ls File?          # "File" + 1 char: FileA, File1, etc.
ls File??         # "File" + 2 chars: File25, FileRR
ls File[5A]       # "File5" or "FileA"
ls File[BR][a2]   # e.g., FileBa, FileB2, FileRa, FileR2

```

### Lab Steps

1. Create test files:
    
    ```bash
    touch fileA fileB FileA File1 File25 FileRR Filez5
    
    ```
    
2. Run each pattern:
    
    ```bash
    ls File*
    ls File??
    ls File[1A]
    
    ```
    

### Tips & Warnings

- `File*` ≠ `file*` (Linux is case-sensitive).
- If **no matches**, the pattern is passed **literally** (e.g., `ls NoSuch*` → error).

---

## 3. Character Ranges and Classes

### Ranges

- `[a-z]`: Lowercase letters
- `[A-Z]`: Uppercase letters
- `[0-9]`: Digits

### Examples

```bash
ls File[a-z]*     # File + lowercase letter + anything: Filez5
ls file[a-z][a-z][0-9]  # "file" + 2 lowercase + 1 digit: filef2? (no match in your list)

```

> 🔍 Note: Your file filef2 matches file[a-z][0-9] (not 2 letters + digit).
> 

### Locale Matters!

- In some locales (e.g., `en_US.UTF-8`), `[a-z]` may include **uppercase** letters due to collation rules.
- Use `LANG=C` for **strict ASCII behavior** (recommended for scripts).

### Fix Locale for Predictable Globbing

```bash
echo $LANG        # Check current locale
LANG=C            # Set to POSIX/C locale
echo $LANG        # Now: C
ls File[a-z]*     # Now only matches true lowercase

```

### Lab Steps

1. Test with default locale:
    
    ```bash
    ls File[A-Z]*   # May include lowercase in some locales!
    
    ```
    
2. Switch to `C` locale:
    
    ```bash
    LANG=C
    ls File[A-Z]*   # Now only uppercase
    
    ```
    

---

## 4. Advanced Patterns

### Combining Patterns

```bash
# Match files like FileBa, FileB2, FileRa, FileR2
ls File[BR][a2]

```

### No Matches?

- If a glob pattern has **no matches**, the shell leaves it **unchanged**:
    
    ```bash
    ls NoSuch*       # Tries to list a file literally named "NoSuch*"
    
    ```
    

---

## 5. Globbing vs. Regular Expressions

### Key Difference

| Feature | Globbing | Regex |
| --- | --- | --- |
| Used by | Shell (for filenames) | Tools like `grep`, `sed` |
| `*` means | Any characters | Zero or more of previous char |
| `.` is literal | Yes | Special (matches any char) |

> ✅ Remember: Globbing is only for filenames—not for text content.
> 

---

## 6. Practical Examples & Common Mistakes

### ✅ Correct

```bash
# Backup all .txt files
cp *.txt backup/

# Remove all log files from 2023
rm *2023*.log

```

### ❌ Dangerous

```bash
rm *            # Deletes ALL files in current directory!
rm *.txt        # Safe only if you're sure what matches

```

### Safety Tips

1. **Test first** with `echo`:
    
    ```bash
    echo rm *.log   # See what would be deleted
    
    ```
    
2. Use `ls` to preview:
    
    ```bash
    ls File[0-9]*
    
    ```
    

---

## Summary Cheat Sheet

| Pattern | Matches | Example Matches |
| --- | --- | --- |
| `*` | Any characters | `File*` → `FileA`, `File25` |
| `?` | One character | `File?` → `FileA`, `File1` |
| `[abc]` | One of a, b, c | `File[1A]` → `File1`, `FileA` |
| `[a-z]` | Lowercase letter | `File[a-z]` → `Filez` |
| `[0-9]` | Digit | `File[0-9]` → `File1`, `File2` |

### Critical Settings

```bash
LANG=C      # Ensures [a-z] = only lowercase
echo *      # Shows all files (like ls)

```

> 💡 Golden Rule:
> 
> 
> Always test glob patterns with `echo` or `ls` before using them with `rm`, `mv`, or `cp`!
> 
> Set `LANG=C` in scripts for consistent behavior.
>