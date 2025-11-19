# 04: Managing Directories & Aliases

## 1. `pwd` – Print Working Directory

### Command

```
pwd

```

### Explanation

- Displays the **current working directory** (the folder where you are located in the filesystem).

### Example

```
$ pwd
/home/user/Documents

```

---

## 2. `cd` – Change Directory

### Commands

```
cd <path>      # Move into a specific directory
cd ~           # Go to the user’s home directory
cd -           # Go back to the previous directory
cd .           # Stay in the current directory
cd ..          # Move one level up (parent directory)

```

### Example Walkthrough

```
cd /etc
pwd           # Shows /etc
cd -          # Returns to previous directory
cd ..         # Goes up one level

```

---

## 3. Auto Path Completion

- Type the **first few letters** of a directory or file, then press **TAB** → Bash will auto-complete it.
- If multiple matches exist, press **TAB** twice to list possible completions.

---

## 4. `ls` – List Files & Directories

### Commands

```
ls        # List files
ls -a     # Show hidden files (starting with .)
ls -lh    # Long listing with human-readable sizes

```

### Example

```
ls -lh
-rw-r--r--  1 user user 1.2K Sep 28  date.txt
drwxr-xr-x  2 user user 4.0K Sep 28  folderA

```

---

## 5. `mkdir` – Make Directories

### Commands

```
mkdir folderA              # Create one directory
mkdir -p folderA/FolderB/FolderC   # Create nested directories

```

### Example

```
mkdir Projects
mkdir -p Projects/2025/DevOps

```

---

## 6. `rm` – Remove Files & Directories

### Commands

```
rm folderD             # Remove a file named folderD (⚠️ if it's a folder, it will fail)
rm -r folderA/FolderB/FolderC   # Remove directories recursively
rm -i date             # Interactive delete (asks before deleting)

```

### Example

```
mkdir testDir
rm -r testDir

```

⚠️ **Warning**: `rm -r` is **permanent**. Files are not sent to Trash. Use `-i` for safety.

---

## 7. Aliases – Custom Shortcuts

### Commands

```
alias                # Show all current aliases
alias rm='rm -i'     # Make rm always ask before deleting
unalias rm           # Remove the custom alias for rm

```

### Example Workflow

```
alias                # See existing aliases
cat > date           # Create a file named 'date'
cat date             # Show contents of 'date'
rm -i date           # Safe delete
unalias rm           # Remove rm alias
alias                # Verify rm alias is gone
rm date              # Now deletes without asking

```

---

## 8. Running Multiple Commands

### Syntax

```
command1 ; command2 ; command3

```

### Example

```
pwd ; ls ; date

```

- Prints current directory
- Lists files
- Shows system date

---

# ✅ Lab: Practice Directory Management

1. Navigate to home directory:
    
    ```
    cd ~
    
    ```
    
2. Create nested folders:
    
    ```
    mkdir -p Lab1/Task1/Subtask
    
    ```
    
3. List contents with details:
    
    ```
    ls -lh Lab1
    
    ```
    
4. Navigate inside, then go back:
    
    ```
    cd Lab1/Task1/Subtask
    cd -
    
    ```
    
5. Remove everything safely:
    
    ```
    rm -ri Lab1
    
    ```
    
6. Set an alias for safe delete:
    
    ```
    alias rm='rm -i'
    
    ```
    

---

# ⚠️ Tips & Warnings

- Always use `rm -i` or an alias for **safe deletion**.
- Use `mkdir -p` when making **nested folders**, so you don’t need to create each manually.
- `cd -` is very useful to switch between two locations quickly.

---