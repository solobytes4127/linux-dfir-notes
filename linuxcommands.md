## 1. Time and System State

### date

What it does: prints current date and time\
Format: day, date, month, year, time, timezone\
DFIR use: timestamp your actions; confirm system timezone before
trusting log timestamps\
: always note the timezone — IST vs UTC matters in reports

### uptime

Output: 11:32:13 up 1:25, 1 user, load average: 0.13, 0.20, 0.20\
\
What each part means:\
- 11:32:13 → current time\
- up 1:25 → system has been running for 1 hour 25 minutes\
- 1 user → one user logged in\
- 0.13, 0.20, 0.20 → load average over last 1, 5, 15 minutes\
\
DFIR use:\
- "up" tells you if the system rebooted recently (a reboot wipes
memory)\
- load average spike can indicate a process consuming resources
(possible malware)\
- number of users can flag unexpected logins\
\
: load average above 1.0 per CPU core means the system is overloaded

## 2. Disk and Partitions

### df

What it does: shows disk space usage for all mounted filesystems\
\
Output columns:\
- Filesystem → the device or virtual fs\
- 1K-blocks → total size in 1K blocks\
- Used → space used\
- Available → space free\
- Use% → percentage used\
- Mounted on → where it's mounted in the directory tree\
\
DFIR use:\
- Check if a disk is nearly full — full disks can cause logging to stop
(evidence loss)\
- Identify unusual mounts (e.g. a tmpfs with high usage could hide
volatile data)\
- Confirm which partitions exist before imaging\
- Notice separate partitions for /var and /home — logs live in /var,
user data in /home\
\
: "Use%" can hit 100% on tmpfs without real trouble (it's RAM-backed).

### df -h

What it does: same as df but sizes in human-readable units (G, M, K)\
\
Difference from df:\
- df → sizes in 1K blocks (precise, script-friendly)\
- df -h → sizes in GB/MB (readable, report-friendly)\
\
DFIR use:\
- Use df -h in reports for readability\
- Use plain df when scripting\
\
: "Avail" is not always "Size minus Used" — filesystems reserve ~5% for
root.

### lsblk

What it does: lists block devices in a tree — shows which physical disk
has which partitions\
\
Output columns:\
- NAME → device name (nvme0n1, nvme0n1p1, etc.)\
- MAJ:MIN → major:minor device numbers\
- RM → removable (1 = yes, 0 = no)\
- SIZE → partition size\
- RO → read-only (1 = yes, 0 = no)\
- TYPE → disk vs part vs rom\
- MOUNTPOINTS → where it's mounted (blank if not mounted)\
\
DFIR use:\
- CRITICAL before imaging: tells you the device path to image\
- Shows unmounted partitions — could hold evidence you'd miss with df\
- Shows swap partition — swap can contain memory remnants\
\
: imaging the whole disk (/dev/nvme0n1) captures everything including
deleted\
partitions. Imaging a single partition misses other evidence.

**The forensic lesson:** df shows you what's **mounted**. lsblk shows
you what **exists**. Those are different questions. If you only run df,
you miss unmounted partitions — a common way to lose evidence.

## 3. Memory

### free

What it does: shows RAM and swap usage\
\
Output columns:\
- total → total RAM installed\
- used → currently in use\
- free → completely unused\
- shared → memory used by tmpfs and shared segments\
- buff/cache → memory used for disk caching (reclaimable)\
- available → best estimate of what's actually available to new apps\
\
DFIR use:\
- "available" is the number that matters, not "free"\
- High buff/cache is normal and good — Linux uses free RAM for disk
cache\
- If swap is in use, it may contain memory remnants\
- Low available + high load = possible resource exhaustion\
\
: "free" is misleadingly small on Linux. Use "available" instead.

**Why it matters for Month 4:** Memory dumps need free disk space
roughly equal to RAM size. If swap is in use, swap is also evidence.

## 4. Navigation and Listing

### pwd

What it does: prints the current working directory (absolute path)\
\
DFIR use:\
- Confirm where you are before running commands\
- In reports, always note the exact path where evidence was found\
\
: pwd tells you WHERE you are; ls tells you WHAT's there.

### cd

What it does: changes the current working directory\
\
Forms:\
- cd /usr/bin → go to absolute path\
- cd .. → go up one level\
- cd ~ → go to home directory\
- cd → also goes to home directory\
- cd - → go to previous directory\
\
DFIR use:\
- Navigate to evidence locations before running tools\
- \`cd -\` is useful when bouncing between two directories\
\
: cd with no argument goes HOME, not to the filesystem root.

### ls

What it does: lists directory contents\
\
Key flags:\
- ls -l → long format (permissions, owner, size, timestamp)\
- ls -a → show hidden files (dotfiles)\
- ls -la → both\
- ls -lh → human-readable sizes\
- ls -lt → sort by modification time (newest first)\
- ls -ltr → sort by time, oldest first\
- ls -li → show inode numbers\
- ls -R → recursive\
- ls -A → hidden files but NOT . and ..\
- ls -s → show allocated size in blocks\
- ls -f → unsorted, shows everything including . and ..\
- ls -d → show the directory itself, not its contents\
- ls -r → reverse order\
\
DFIR use:\
- \`ls -la\` reveals hidden files — malware often hides as .something\
- \`ls -lt\` shows most recently modified files — "what changed?"\
- \`ls -li\` shows inodes — useful for detecting hardlinks and deleted
file remnants\
- Timestamps on files are evidence: mtime, atime, ctime\
\
: ls doesn't show files starting with "." unless you use -a. Always use
-la in\
forensic work. Also: \`ls -h\` alone looks identical to \`ls\` — -h only
affects\
size display, and plain ls doesn't show sizes.

**ls -lh is your default listing.** It's what you run first on any
directory in an investigation. Look for outliers by size, type, and
time.

## 5. Reading Files

### less

What it does: view a file one page at a time, scrollable, without
modifying it\
\
Navigation:\
- Space / Page Down → next page\
- b / Page Up → previous page\
- g → go to start\
- G → go to end\
- /pattern → search forward\
- ?pattern → search backward\
- n → next match\
- N → previous match\
- q → quit\
\
DFIR use:\
- Read files too long for cat (logs, /etc/passwd, config files)\
- Safe: less does NOT write to the file. Evidence stays untouched.\
- Search inside a log for "Failed", "Accepted", an IP, or a username\
\
: less doesn't modify files. Do NOT confuse with editors like vi/nano\
(which CAN modify).

### Which reader when?

cat → short file, print all at once\
less → long file, read-only, searchable ← default for DFIR\
head → first N lines (head -20 file)\
tail → last N lines (tail -50 file)\
tail -f → follow a live log (Ctrl+C to stop)

## 6. User Accounts

### The file: /etc/passwd

Format: each line has 7 colon-separated fields:\
username:x:UID:GID:comment:home:shell\
(this is a FORMAT, not a command — don't run it)\
\
Example real line:\
baalaje:x:1000:1000:Baalaje:/home/baalaje:/bin/zsh\
\
DFIR use:\
- UID 0 = root. A second UID-0 account = backdoor.\
- shell = /bin/bash or /bin/zsh → can log in. /usr/sbin/nologin →
cannot.\
- Look for accounts you didn't expect to exist.

### UID ranges to remember

0 → root (only one should exist)\
1-999 → system/service accounts\
1000+ → human users

### Related files

| File         | What it holds             | Access         |
|--------------|---------------------------|----------------|
| /etc/passwd  | User accounts, UID, shell | World-readable |
| /etc/shadow  | Password hashes, expiry   | Root only      |
| /etc/group   | Groups and members        | World-readable |
| /etc/sudoers | Who can run sudo          | Root only      |

### grep nologin /etc/passwd

Shows service accounts (can't log in). Compare with grep bash
/etc/passwd (can log in). The difference between the two lists is what
matters in an investigation.

**What to scan for:**

- A service account with /bin/bash → red flag

- A new account with UID 1000+ → possible backdoor

- An account with UID 0 other than root → backdoor

- A name that looks out of place

## 7. The Root Directory — Forensic Map

| Directory | Why a forensic analyst cares |
|----|----|
| /etc | Configs. passwd, shadow, sudoers, crontab = who/where/how |
| /var/log | **Logs.** auth.log, syslog, kern.log = who did what when |
| /home | User data. Browser history, shell history, documents, downloads |
| /tmp | Transient. Malware often drops payloads here (world-writable) |
| /dev/shm | RAM-backed. Malware sometimes hides here — vanishes on reboot |
| /proc | Live system state. Real-time process info — **key for live forensics** |
| /root | Root's home. Root shell history, scripts, keys |
| /bin, /usr/bin | Binaries. Malware may replace or add binaries here |
| /boot | Kernel. Bootkits live here — rare, but catastrophic if present |

Command: cp \<source\> \<destination\>::

What it does: copies files and directories to a new location or name.

Source stays; a duplicate appears at the destination.

syntax examples:

\- cp notes.txt backup.txt → same dir, new name

\- cp notes.txt /home/baalaje/Documents/ → same name, new location

\- cp notes.txt /backup/notes-copy.txt → new location AND new name

\- cp -r myfolder/ /backup/ → copy a whole folder (needs -r)

Output columns: none — cp is silent unless -v is used.

Key flags:

\- cp -r → recursive (required for directories)

\- cp -a → archive mode (preserves permissions, timestamps, links) —
best for evidence

\- cp -p → preserve timestamps, permissions, ownership

\- cp -u → update: copy only if source is newer than destination

\- cp -v → verbose (show each file copied)

\- cp -i → interactive (prompt before overwrite)

\- cp -n → never overwrite existing files

DFIR use:

\- Working copies of configs, logs, artifacts for analysis

\- Incremental collection with -u (only copy what changed)

\- NEVER for evidence acquisition — use dd / FTK Imager / dc3dd instead

\- Pair with sha256sum to verify copy matches source

\- Directories need -r or -a. Plain cp fails on folders.

\- Trailing slash on destination means "directory":

cp file.txt /backup → may create a FILE named backup

cp file.txt /backup/ → puts file.txt INSIDE the directory

\- Overwrites silently by default. No undo.

\- Does NOT preserve timestamps unless -p or -a is used.

\- -u compares modification time only, not content. mtime can be faked.

\- cp is not forensically safe for evidence: mtime of copy = now (unless
-p),

atime on source may change just from reading, sparse files expand.

Practical rule: for evidence-adjacent work, use

cp -avp \<source\> \<destination\>

archive + verbose + preserve.

Wildcards:

\- \*.html expands BEFORE cp sees it. Confirm with \`ls \*.html\` first.

\- If nothing matches, the shell passes literal \*.html and cp errors.

Command: mkdir \<directory\>::

What it does: creates one or more new directories.

Forms:

\- mkdir dir1 → create one directory

\- mkdir dir1 dir2 dir3 → create several at once

\- mkdir -p parent/child/grand → create nested dirs, no error if parents
exist

\- mkdir -v dir1 → verbose: print each directory created

\- mkdir -m 700 dir1 → set permissions at creation (rwx------)

Output columns: none — silent unless -v is used.

Key flags:

\- -p / --parents → make parent directories as needed; no error if dir
exists

\- -v / --verbose → print a message for each created directory

\- -m / --mode → set file mode (permissions) at creation

DFIR use:

\- Set up a clean evidence directory before collecting:

mkdir -p /evidence/{logs,disk,memory,network,notes}

\- -p is safe to re-run — won't error if the directory already exists

\- Avoid creating evidence dirs inside the suspect system; use external
storage

\- -m 700 on sensitive evidence dirs keeps them root-only

\- Without -p, mkdir fails if the parent doesn't exist:

mkdir a/b/c → "No such file or directory" if a doesn't exist

mkdir -p a/b/c → creates a, a/b, a/b/c

\- Without -p, mkdir also fails if the directory already exists:

mkdir evidence → "File exists"

mkdir -p evidence → silent, no error

\- Default permissions follow your umask. Use -m for explicit control.

mv → move/rename. Source gone. Silent overwrite. Directory behavior
differs from cp.

ln → hard link (same inode). ln -s → symlink (path pointer).

ls -li → show inode numbers. Same inode = same file.

rm → delete. No undo. rm -r for dirs. Preview wildcards first.

rmdir → safe: removes empty directories only.

Command: type \<command\>

What it does: tells you HOW the shell interprets a command name.

Possible outputs:

\- "X is a shell builtin" → built into the shell itself (cd, echo, pwd)

\- "X is /usr/bin/X" → external program on disk

\- "X is aliased to \`...\`" → user-defined shortcut

\- "X is a function" → shell function

\- "X is a shell keyword" → reserved word (if, while, for)

Useful flags:

\- type -a \<command\> → show ALL locations, not just the first

\- type -t \<command\> → print just the type word
(builtin/file/alias/keyword/function)

\- type -p \<command\> → print only the path (like \`which\`)

DFIR use:

\- Verify a command is the REAL binary, not an alias or function
overriding it

\- If \`ls\` is aliased to something malicious, \`type ls\` reveals it

\- On a suspect system, \`type -a \<cmd\>\` shows every version in PATH

\- \`type -t \<cmd\>\` is script-friendly for checking before execution

\- Builtins don't have a file on disk — \`type cd\` says "shell
builtin",

\`which cd\` often finds nothing

\- Aliases can silently override standard commands. \`type\` is how you
find out.

\- Example attack: alias sudo='sudo ' (trailing space) — or worse, an
alias that

logs your password. \`type sudo\` catches it.

Command: which \<command\>

What it does: shows which executable file in PATH would run for that
command.

Example:

which python3 → /usr/bin/python3

which ls → /usr/bin/ls

Useful flags:

\- which -a \<command\> → show ALL matches in PATH, not just the first

DFIR use:

\- Confirm the exact binary that will run (path matters in forensics)

\- Find alternate versions of a tool (e.g. python vs python3 vs
python3.11)

\- If \`which python\` returns /tmp/python, that's suspicious — PATH
hijack or

malicious binary staged in /tmp

:

\- \`which\` only finds EXTERNAL programs — not shell builtins (cd,
echo)

or aliases. Use \`type\` for those.

\- Doesn't tell you if it's an alias — only \`type\` does.

\- On some systems, \`which\` is a shell function or alias itself (Zsh
on Kali).

Use \`type -a which\` to see.

\- Different shells have different \`which\` behavior. \`command -v
\<cmd\>\` is more

portable across shells.

Command: help \[builtin\]

What it does: shows documentation for SHELL BUILTINS only.

Examples:

\- help → list all builtins

\- help cd → help for the cd builtin

\- help echo → help for the echo builtin

DFIR use:

\- Understand builtin behavior when scripting collection/analysis tools

\- Builtins differ between shells (bash vs zsh) — \`help\` confirms your
shell's behavior

\- Quick reference during a live investigation without leaving the
terminal

:

\- \`help\` does NOT work for external programs (ls, cp, cat).

Those need \`man\` or \`--help\`.

\- Bash builtin. In zsh, \`help\` behaves differently (or isn't
available).

\- If \`help ls\` shows "no help topic for ls", that's because ls is
external.

Command: man \<command\>

What it does: opens the manual page for a command.

Navigation: same as less (it uses less as the pager)

\- Space / b → page down/up

\- /pattern → search

\- n / N → next/previous match

\- q → quit

Useful flags:

\- man -k \<keyword\> → same as apropos: search descriptions

\- man \<section\> \<cmd\> → read a specific section (see below)

\- man -f \<command\> → same as whatis: one-line description

Sections:

1 → user commands

2 → system calls (kernel)

3 → library functions

5 → file formats and conventions (e.g. man 5 passwd)

8 → system administration commands

DFIR use:

\- The authoritative reference for tool flags — faster and more reliable
than

a web search during an investigation

\- man 5 passwd → shows the FORMAT of /etc/passwd (useful for parsing)

\- man 8 tcpdump → admin-level flags for packet capture

\- Section numbers let you disambiguate: \`man 1 passwd\` is the
command,

\`man 5 passwd\` is the file format

:

\- Not every command has a man page — some use \`--help\` or \`info\`

\- Builtins may have a man page in section 1 that differs from
shell-specific

behavior — trust \`help\` for builtins

\- Manual pages can be out of date with the installed version —
cross-check with

\`\<command\> --help\` or \`--version\`

Command: apropos \<keyword\>

What it does: searches manual page NAMES and DESCRIPTIONS for a keyword.

Finds commands when you know what you want to do but not the name.

Examples:

\- apropos copy → every command whose description mentions "copy"

\- apropos "disk image" → quoted multi-word search

\- apropos forensic → commands related to forensics

Useful flags:

\- apropos -e \<term\> → exact word match only

DFIR use:

\- Discover tools when you know the task but not the command

apropos "memory image"

apropos "packet capture"

apropos "file system"

\- Inventory available forensic tools on a system quickly

\- Find the right man page when you only remember the description

:

\- Requires the man-db / mandb database to be built (usually already
done)

\- Case-insensitive by default on most systems

\- Might not find commands whose descriptions don't mention your keyword

\- Same as \`man -k\`

Command: info \<command\>

What it does: opens a GNU info page — a more detailed, hyperlinked
manual.

Navigation:

\- Space / Backspace → page through nodes

\- Tab → jump to next hyperlink

\- Enter → follow a link

\- n / p / u → next / previous / up one node

\- q → quit

DFIR use:

\- More thorough than man for GNU tools (coreutils, bash, find, grep)

\- Often includes examples and tutorials the man page omits

\- \`info find\` or \`info grep\` — genuinely more detailed than the man
page

:

\- Only exists for GNU tools. Many non-GNU commands have no info page.

\- Uses its own pager (not less) — navigation is different

\- Many people never use it — man + --help is usually enough

\- On minimal systems, \`info\` may not be installed

Command: whatis \<command\>

What it does: shows the ONE-LINE description from the man page.

Quick lookup without opening the full manual.

Examples:

\- whatis ls → ls (1) - list directory contents

\- whatis cp → cp (1) - copy files and directories

\- whatis passwd → passwd (1) - change user password

passwd (5) - the password file

Useful flags:

\- whatis -w \<pattern\> → wildcard search on command names

DFIR use:

\- Quick tool identification when you're not sure what a binary does

\- Confirm the purpose of an unfamiliar command on a suspect system

\- Fast reference while working in a terminal — no pager to close

:

\- Only prints the first line — no syntax, no flags, no examples

\- Same as \`man -f\`

\- Multiple sections may print (e.g. passwd has entries in sections 1
and 5)

\- If the man database isn't built, returns "nothing appropriate"

Command: alias ='' What it does: creates a shortcut — a new name that
expands to another command.

Examples:

- alias → list all current aliases

- alias ll='ls -l' → define shortcut "ll"

- alias la='ls -la' → define "la"

- unalias ll → remove the alias

- \ls → bypass the alias and run the real command

DFIR use:

- ⚠️ DETECTIVE ANGLE: aliases can override standard commands alias
  ls='ls -la --color' → benign, common alias sudo='read -p "pass: " x;
  echo \$x \| sudo -S ...' → password steal alias with no args lists
  everything — check it on a suspect system

- Confirm standard commands aren't shadowed: type -a ls

- To bypass any alias, prefix the command with \\ or use the full
  path: \ls /usr/bin/ls

:

- Aliases only exist in the current shell session — they don't persist
  unless written to ~/.bashrc, ~/.zshrc, or /etc/profile.d/\*

- Check config files for suspicious aliases: cat ~/.bashrc ~/.zshrc
  /etc/profile /etc/bash.bashrc

- Aliases take precedence over builtins and external commands — a
  malicious alias can silently hijack a trusted command

- To see what an alias actually expands to: type \<name\> or alias
  \<name\>

- \command runs the un-aliased version; /full/path also bypasses
