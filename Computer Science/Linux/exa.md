---
memory: to_finish
tags:
  - learned
language:
  - Linux
review-date:
last-reviewed: 2025-09-22
scheda: done
visit-count: 2
confidence-level: 2
consecutive-correct: 2
---

```dataviewjs
const currentPage = dv.current();
let visitCount = currentPage.file.frontmatter["visit-count"] || 0;
let confidence = currentPage.file.frontmatter["confidence-level"] || 1;
let streak = currentPage.file.frontmatter["consecutive-correct"] || 0;

const container = this.container.createEl('div');
container.style.cssText = `
    background: #2a2a2a; border: 1px solid #404040; border-radius: 6px;
    padding: 12px; margin: 10px 0; display: inline-block;
`;

// Status display
const status = container.createEl('div');
status.innerHTML = `
    <strong>Learning Progress</strong><br>
    Reviews: ${visitCount} | Confidence: ${confidence}/5 | Streak: ${streak}
`;
status.style.cssText = 'margin-bottom: 10px; font-size: 13px; color: #cccccc;';

// Quick feedback buttons
const buttonContainer = container.createEl('div');
['Got it! ✅', 'Struggled ⚠️', 'Failed ❌'].forEach((label, index) => {
    const btn = buttonContainer.createEl('button');
    btn.textContent = label;
    btn.style.cssText = `
        margin-right: 8px; padding: 4px 8px; border: none; border-radius: 3px;
        cursor: pointer; font-size: 11px;
        background: ${['#28a745', '#ffc107', '#dc3545'][index]};
        color: ${index === 1 ? '#000' : '#fff'};
    `;
    
    btn.addEventListener('click', async () => {
        visitCount++;
        if (index === 0) { // Got it
            confidence = Math.min(5, confidence + 0.5);
            streak++;
        } else { // Struggled or failed
            confidence = Math.max(1, confidence - 0.5);
            streak = 0;
        }
        
        const file = app.vault.getAbstractFileByPath(currentPage.file.path);
        await app.fileManager.processFrontMatter(file, (fm) => {
            fm["visit-count"] = visitCount;
            fm["confidence-level"] = confidence;
            fm["consecutive-correct"] = streak;
            fm["last-reviewed"] = new Date().toISOString().split('T')[0];
            if (index > 0) fm["last-struggle-date"] = new Date().toISOString().split('T')[0];
        });
        
        status.innerHTML = `
            <strong>Learning Progress</strong><br>
            Reviews: ${visitCount} | Confidence: ${confidence}/5 | Streak: ${streak}
        `;
    });
});
```

# **Purpose/Why**:
---

The `exa` command solves the problem of ==**modernizing and enhancing directory listing** in Unix-like operating systems, traditionally handled by the venerable `ls` command==. While `ls` is a powerful and ubiquitous tool, ==it dates back to the early days of Unix== and can be verbose or require many flags to get desirable output.

`exa` aims to be a **modern replacement for `ls`**, offering:

1. **Improved Readability:** It uses colors to distinguish file types, permissions, owners, and dates, making the output much easier to parse at a glance.
2. **More Information by Default:** It provides useful information like Git status, file sizes in human-readable format, and a tree view without needing a complex combination of flags.
3. **Better Defaults:** Many common `ls` options (like `-l` for long listing) are either default or easily accessible with simpler flags.
4. **Enhanced Features:** Features like recursive listing with a tree-like output, Git status integration, and better handling of extended attributes are built-in.


It's important in computer science (specifically in systems administration, development, and general Linux usage) because efficient and clear file system navigation is a fundamental task. `exa` enhances productivity by providing more intuitive and visually rich information about files and directories, reducing the mental effort required to understand command output. For developers, seeing Git status directly in the directory listing is a significant convenience. For anyone interacting with the command line frequently, `exa` offers a significant quality-of-life improvement over `ls`.

# **Core Explanation:**
---

`exa` is a modern, feature-rich command-line utility written in Rust, designed to be a more user-friendly and powerful alternative to the standard `ls` command for listing directory contents.

**Key Characteristics:**

- **Color-coding:** `exa` heavily utilizes color to make its output more readable and informative. Different file types (directories, executables, images, archives), permissions, ownership, and dates are all color-coded.
- **Human-readable file sizes:** By default, file sizes are displayed in human-readable units (e.g., KB, MB, GB), similar to `ls -h`.
- **Git integration:** If you are in a Git repository, `exa` can show the Git status of files and directories (modified, new, ignored, etc.) directly in its output.
- **Tree view:** `exa` can display directory contents recursively in a tree-like structure, making it easy to visualize directory hierarchies.
- **Improved long listing:** Its default long listing provides more information and is better formatted than `ls -l`.
- **Symlink following:** It can resolve symlinks to show their target.
- **Cross-platform:** While most commonly used on Linux and macOS, it's designed to be cross-platform.


**How it Works:**

`exa` works by interacting with the operating system's file system APIs (like `stat` and `readlink`) to gather metadata about files and directories. It then processes this information and formats it according to its internal logic and the command-line options provided by the user.

For example, when you run `exa -l`, it:

1. Retrieves file permissions, owner/group IDs, file size, last modified time, and file name for each entry.
2. Looks up user and group names from their IDs.
3. Calculates file sizes in human-readable format.
4. Determines file types (directory, regular file, symlink, etc.) to apply appropriate colors.
5. If Git integration is active (e.g., with `--git` option), it queries the Git repository for the status of each file.
6. Presents all this information in a structured, color-coded table.

Unlike `ls`, `exa`'s developers have chosen better defaults and richer output, minimizing the need for complex alias chains (like `alias ls='ls -lah --color=auto'`) that are often seen with `ls`.

# exa Command Line Flags

`-1, --oneline` - Display one entry per line
`-l, --long` - Display extended file metadata as a table
`-G, --grid` - Display entries as a grid (default)
`-x, --across` - Sort the grid across, rather than downwards
`-T, --tree` - Recurse into directories as a tree
`-R, --recurse` - Recurse into directories
`-b, --binary` - List file sizes with binary prefixes
`-B, --bytes` - List file sizes in bytes, without prefixes
`-g, --group` - List each file's group
`-h, --header` - Add a header row to each column
`-H, --links` - List each file's number of hard links
`-i, --inode` - List each file's inode number
`-L, --level DEPTH` - Limit the depth of recursion
`-m, --modified` - Use modified timestamp field
`-S, --blocks` - Show number of file system blocks
`-t, --time FIELD` - Which timestamp field to list
`-u, --accessed` - Use accessed timestamp field
`-U, --created` - Use created timestamp field
`--time-style` - How to format timestamps
`--git` - List each file's Git status, if tracked
`-@, --extended` - List file's extended attributes
`--color, --colour` - When to use terminal colors
`--icons` - Display icons
`--no-permissions` - Suppress the permissions field
`--sort FIELD` - Sort by field (name, size, modifie

# **Related Concepts:**

---
- **`ls` command:** This is the direct predecessor and the traditional Unix command for listing directory contents.
 - **Connection:** `exa` is designed as a drop-in replacement for `ls`. They both serve the same core purpose.
 - **Difference:** `exa` offers more modern features, better default output formatting, and enhanced readability (especially with color), often requiring fewer flags to achieve desired results compared to `ls`. `ls` is universally available; `exa` needs to be installed.
- **Shell Aliases:** Shell aliases allow you to create shortcuts or customize existing commands. Many users alias `ls` to `ls -lah --color=auto` to get more useful output by default.
 - **Connection:** `exa` aims to provide much of this desired output by default, reducing the need for extensive `ls` aliases. Users often alias `ls` to `exa` directly after installing `exa`.
 - **Difference:** Aliases are user-specific customizations, while `exa` is a distinct binary with built-in features that go beyond what simple `ls` flags can provide (e.g., tree view, Git integration).
- **File System (and inodes):** The file system is the hierarchical structure used to organize files and directories on a storage device. Both `ls` and `exa` retrieve their information by querying the file system metadata (e.g., permissions, size, modification times) stored in **inodes**.

# **Examples:**
---
```bash

# This is a bash script to demonstrate exa commands.
# You need to have 'exa' installed on your system for these commands to work.
# Installation: .com/ogham/exa
# installation

echo "
---
Basic exa usage (like 'ls')
---
"

# Simply running 'exa' lists the contents of the current directory.
# It uses color by default to differentiate file types (directories, executables, regular files).
exa

echo "\n
---

Long listing format ('exa -l' vs 'ls -l')
---
"

# '-l' (long) option provides detailed information:
# permissions, number of links, owner, group, size, last modified date/time, and name.
# Notice the colors for permissions and file types.
exa -l

echo "\n
---
Show hidden files ('exa -a' vs 'ls -a')
---
"

# '-a' (all) option shows hidden files (those starting with a dot).
exa -a

echo "\n
---
Human-readable sizes ('exa -h' is often default or implied with -l)
---
"

# '-h' (human-readable) shows file sizes in KB, MB, GB etc.
# Exa often does this by default with '-l'.
exa -lh

# Combines long listing and human-readable sizes
echo "\n
---
Displaying directory tree ('exa -T')
---
"

# '-T' (tree) option shows contents recursively in a tree format.
# You can limit the depth with '--level'.
mkdir -p my_project/src/models my_project/src/views
touch my_project/src/main.cpp my_project/src/models/user.h my_project/src/views/dashboard.cpp
touch my_project/README.md
exa -T my_project

# Shows a directory tree of 'my_project'
echo "\n
---
Displaying Git status ('exa --git')
---
"

# '--git' option shows the Git status of files.
# Requires being in a Git repository.
# Let's simulate a tiny Git repo.
mkdir -p git_test
cd git_test
git init > /dev/null 2>&1

# Initialize a Git repository silently
echo "Initial content" > file1.txt
git add file1.txt
git commit -m "Initial commit" > /dev/null 2>&1
echo "Modified content" > file1.txt

# Modify an existing file
echo "New file content" > file2.txt

# Create a new file
echo "Ignored file" > .gitignore_me

# Create a file that would typically be ignored
# Now run exa with --git
# 'M' for modified, 'U' for untracked (or '?' in some Git versions), 'I' for ignored
exa -l --git

# Shows long listing with Git status (M, U, I etc.)
cd ..

# Go back to original directory
rm -rf git_test my_project

# Clean up created directories
```