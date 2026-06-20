# Bash / Shell Scripting — Complete Notes (Basics → Advanced + Interview Prep)

---

## Table of Contents
1. [Introduction](#1-introduction)
2. [Shebang & Script Basics](#2-shebang--script-basics)
3. [Variables](#3-variables)
4. [Quoting Rules](#4-quoting-rules)
5. [Command Substitution & Arithmetic](#5-command-substitution--arithmetic)
6. [String Manipulation](#6-string-manipulation)
7. [Arrays](#7-arrays)
8. [Conditionals & Test Operators](#8-conditionals--test-operators)
9. [Loops](#9-loops)
10. [Case Statements](#10-case-statements)
11. [Functions](#11-functions)
12. [Script Arguments & getopts](#12-script-arguments--getopts)
13. [Input/Output & Redirection](#13-inputoutput--redirection)
14. [Pipes & Process Substitution](#14-pipes--process-substitution)
15. [Exit Codes & Error Handling](#15-exit-codes--error-handling)
16. [set Options (Strict Mode)](#16-set-options-strict-mode)
17. [Trap & Signal Handling](#17-trap--signal-handling)
18. [Here-Documents & Here-Strings](#18-here-documents--here-strings)
19. [Subshells vs Current Shell](#19-subshells-vs-current-shell)
20. [Regular Expressions in Bash](#20-regular-expressions-in-bash)
21. [File Testing & Globbing](#21-file-testing--globbing)
22. [Reading Files & Input](#22-reading-files--input)
23. [Common Utilities Used in Scripts](#23-common-utilities-used-in-scripts)
24. [Debugging Scripts](#24-debugging-scripts)
25. [sh vs bash vs zsh](#25-sh-vs-bash-vs-zsh)
26. [Common Patterns & Idioms](#26-common-patterns--idioms)
27. [Security Considerations](#27-security-considerations)
28. [Best Practices Summary](#28-best-practices-summary)
29. [Cheat Sheet](#29-cheat-sheet)
30. [Interview Questions & Answers](#30-interview-questions--answers)

---

## 1. Introduction

A **shell** is both an interactive command interpreter and a scripting language for automating command-line tasks. **Bash** (Bourne Again SHell) is the most common Linux/macOS default shell and scripting language — used everywhere from personal automation to CI/CD pipelines, Docker `ENTRYPOINT` scripts, and system administration.

**Why shell scripting matters for DevOps/SRE roles:**
- Glue logic in CI/CD pipelines (Jenkins, GitHub Actions steps, GitLab CI)
- Automating repetitive sysadmin tasks (backups, log rotation, deployments)
- Docker entrypoint/health-check scripts
- Quick text-processing and system diagnostics
- Universally available — no installation needed on virtually any Linux/Unix system

---

## 2. Shebang & Script Basics

```bash
#!/bin/bash
# This is a comment
echo "Hello, World!"
```

The **shebang** (`#!`) on the first line tells the OS which interpreter to use to run the script. `#!/bin/bash` explicitly uses bash; `#!/bin/sh` uses the system's default POSIX shell (often a more limited shell like `dash` on Debian/Ubuntu — fewer bash-specific features available).

```bash
chmod +x script.sh      # make executable
./script.sh               # run it directly (uses the shebang)
bash script.sh              # run explicitly with bash, ignoring the shebang
sh script.sh                  # run explicitly with sh
source script.sh                # run in the CURRENT shell (variables/functions persist after)
. script.sh                      # same as source (POSIX-compatible shorthand)
```

**`./script.sh` vs `source script.sh` (interview point):** Running a script normally (`./script.sh` or `bash script.sh`) executes it in a **new child process/subshell** — any variables or `cd` changes it makes don't affect your current shell. `source`-ing (or `.`-ing) a script runs it **within your current shell session**, so any variable assignments, function definitions, or directory changes persist afterward — this is exactly why you `source ~/.bashrc` rather than executing it.

---

## 3. Variables

```bash
NAME="Alice"               # no spaces around =
echo "$NAME"                  # always recommended to quote variable expansions
echo ${NAME}                    # braces — useful for disambiguating (e.g., ${NAME}_suffix)

readonly PI=3.14             # constant — cannot be reassigned
unset NAME                     # remove a variable

# Default values
echo "${VAR:-default}"     # use "default" if VAR is unset or empty (doesn't change VAR)
echo "${VAR:=default}"     # use "default" AND assign it to VAR if unset/empty
echo "${VAR:?error msg}"   # print error and exit if VAR is unset/empty
echo "${VAR:+alt}"         # use "alt" only if VAR IS set (inverse of :-)

# Local vs global (inside functions)
my_func() {
    local local_var="only visible inside this function"
    global_var="visible everywhere after this runs"
}
```

**Special/built-in variables:**
| Variable | Meaning |
|---|---|
| `$0` | Script name |
| `$1`, `$2`, ... | Positional arguments |
| `$#` | Number of arguments |
| `$@` | All arguments, as separate words (preserves individual quoting) |
| `$*` | All arguments, as a single combined string |
| `$?` | Exit code of the last command |
| `$$` | PID of the current shell/script |
| `$!` | PID of the last background process |
| `$_` | Last argument of the previous command |

**`$@` vs `$*` (very common interview question):** When quoted as `"$@"`, each argument is treated as a **separate, individually-quoted word** — correctly preserving arguments containing spaces. When quoted as `"$*"`, all arguments are joined into a **single string** (separated by the first character of `$IFS`, space by default). `"$@"` is almost always what you want when passing arguments through to another command.

---

## 4. Quoting Rules

```bash
echo "Hello $NAME"      # double quotes: variables/command substitution ARE expanded
echo 'Hello $NAME'        # single quotes: NOTHING is expanded — fully literal
echo "Cost: \$5"            # backslash escapes a special character even inside double quotes
echo "It's a test"           # single quote is fine inside double quotes
```

| Quote type | Variable expansion | Command substitution | Glob expansion |
|---|---|---|---|
| `'single'` | No | No | No |
| `"double"` | Yes | Yes | No |
| (none) | Yes | Yes | Yes (word splitting also happens) |

**Why always quote variables (very common gotcha/interview point):** Without quotes, `$VAR` undergoes **word splitting** (on spaces/`$IFS`) and **glob expansion** (`*`, `?` get expanded against filenames) — this is a classic source of bugs, especially with filenames containing spaces.
```bash
FILE="my file.txt"
rm $FILE        # BROKEN: tries to remove two files: "my" and "file.txt"
rm "$FILE"       # CORRECT: removes the single file "my file.txt"
```
**Rule of thumb: always double-quote variable expansions unless you specifically need word splitting/globbing.**

---

## 5. Command Substitution & Arithmetic

```bash
# Command substitution — capture a command's output into a variable
current_date=$(date +%Y-%m-%d)        # modern, preferred syntax
current_date=`date +%Y-%m-%d`           # legacy backtick syntax — avoid, hard to nest/read

echo "Today is $(date)"
file_count=$(ls | wc -l)

# Arithmetic
result=$((5 + 3))
echo $((10 / 3))            # integer division → 3 (bash has no native floating point)
echo $((10 % 3))             # modulo → 1
((count++))                    # increment (no $ needed inside double parens)
let "x = 5 + 3"                  # alternative arithmetic syntax

# Floating point (bash itself can't do it — delegate to a tool)
echo "scale=2; 10/3" | bc          # using bc for floating-point math
python3 -c "print(10/3)"             # or just shell out to python/awk
```

**`$()` vs backticks (interview point):** `$(command)` is the modern, POSIX-compliant syntax — it nests cleanly (`$(cmd1 $(cmd2))`) and is more readable. Backticks (`` `command` ``) are the legacy syntax, harder to nest (requires escaping inner backticks), and generally discouraged in new scripts.

---

## 6. String Manipulation

```bash
str="Hello World"

echo ${#str}                 # length → 11
echo ${str:0:5}                # substring from index 0, length 5 → "Hello"
echo ${str: -5}                  # last 5 characters → "World" (note the space before -5!)
echo ${str/World/Bash}             # replace first match → "Hello Bash"
echo ${str//o/0}                     # replace ALL matches → "Hell0 W0rld"
echo ${str^^}                          # uppercase entire string → "HELLO WORLD"
echo ${str,,}                            # lowercase entire string → "hello world"
echo ${str^}                               # uppercase first character only
echo ${str#Hello }                          # remove shortest match from the START → "World"
echo ${str%World}                             # remove shortest match from the END → "Hello "
echo ${str##*/}                                 # remove longest match from start — classic "basename" trick
echo ${str%%/*}                                   # remove longest match from end — classic "dirname" trick

# Concatenation
greeting="Hello"
full="$greeting, World!"

# Splitting a string into an array
IFS=',' read -ra parts <<< "a,b,c"
echo "${parts[1]}"     # "b"
```

**`#`/`##` vs `%`/`%%` (interview point):** `#` strips the **shortest** matching pattern from the **front**; `##` strips the **longest** matching pattern from the front. `%` strips the shortest match from the **end**; `%%` strips the longest match from the end. This is the basis for common idioms: `${path##*/}` extracts a filename from a path (strip everything up to and including the last `/`), and `${path%/*}` extracts the directory (strip everything from the last `/` onward).

---

## 7. Arrays

```bash
# Indexed array
fruits=("apple" "banana" "cherry")
echo "${fruits[0]}"             # "apple"
echo "${fruits[@]}"               # all elements: "apple banana cherry"
echo "${#fruits[@]}"                # number of elements → 3
fruits+=("date")                      # append an element
unset fruits[1]                         # remove element at index 1

for fruit in "${fruits[@]}"; do
    echo "$fruit"
done

# Associative array (bash 4+, requires explicit declare)
declare -A colors
colors[apple]="red"
colors[banana]="yellow"
echo "${colors[apple]}"            # "red"
echo "${!colors[@]}"                 # all keys
echo "${colors[@]}"                    # all values

for key in "${!colors[@]}"; do
    echo "$key -> ${colors[$key]}"
done
```

**`"${array[@]}"` vs `"${array[*]}"` (same distinction as `$@` vs `$*`):** `[@]` expands each element as a separate word (correctly handles elements with spaces when used in a `for` loop); `[*]` joins all elements into one string.

---

## 8. Conditionals & Test Operators

```bash
if [ "$x" -gt 10 ]; then
    echo "x is greater than 10"
elif [ "$x" -eq 10 ]; then
    echo "x is exactly 10"
else
    echo "x is less than 10"
fi

# [[ ]] is bash's improved test syntax (preferred over single [ ])
if [[ "$name" == "Alice" && "$age" -ge 18 ]]; then
    echo "Welcome, adult Alice"
fi
```

**Comparison operators:**
| Numeric | String | Meaning |
|---|---|---|
| `-eq` | `==` or `=` | equal |
| `-ne` | `!=` | not equal |
| `-gt` | `>` (inside `[[ ]]`) | greater than |
| `-lt` | `<` (inside `[[ ]]`) | less than |
| `-ge` | — | greater than or equal |
| `-le` | — | less than or equal |

**File test operators:**
```bash
[ -f file.txt ]      # is a regular file
[ -d dir ]             # is a directory
[ -e path ]              # exists (any type)
[ -r file.txt ]            # is readable
[ -w file.txt ]              # is writable
[ -x file.txt ]                # is executable
[ -s file.txt ]                  # exists and is non-empty
[ -L link ]                        # is a symbolic link
```

**Logical operators:**
```bash
[ "$a" -eq 1 ] && [ "$b" -eq 2 ]     # AND (separate test commands)
[[ "$a" -eq 1 && "$b" -eq 2 ]]         # AND (inside a single [[ ]])
[ "$a" -eq 1 ] || [ "$b" -eq 2 ]         # OR
command1 && command2                       # run command2 only if command1 succeeds (exit 0)
command1 || command2                         # run command2 only if command1 fails (exit non-zero)
```

**`[ ]` vs `[[ ]]` (very common interview question):** `[ ]` is the original POSIX `test` command — available in all shells, but more error-prone (e.g., unquoted empty variables can cause syntax errors, no native `&&`/`||` inside it, requires backslash-escaping `<`/`>` for string comparison, word-splits and glob-expands its arguments). `[[ ]]` is a bash (and zsh/ksh) keyword with safer parsing (no word-splitting/globbing issues with unquoted variables), supports `&&`/`||`/pattern matching/regex directly inside it, and is generally recommended in bash scripts when portability to other POSIX shells (`sh`/`dash`) isn't required.

---

## 9. Loops

```bash
# for loop — list of values
for i in 1 2 3 4 5; do
    echo "Number: $i"
done

# for loop — range (bash-specific brace expansion)
for i in {1..5}; do
    echo "$i"
done
for i in {1..10..2}; do     # step by 2
    echo "$i"
done

# C-style for loop
for ((i=0; i<5; i++)); do
    echo "$i"
done

# Looping over files
for file in /var/log/*.log; do
    echo "Processing: $file"
done

# while loop
i=0
while [ $i -lt 5 ]; do
    echo "$i"
    ((i++))
done

# until loop (opposite of while — runs while condition is FALSE)
i=0
until [ $i -ge 5 ]; do
    echo "$i"
    ((i++))
done

# Reading lines from a file
while IFS= read -r line; do
    echo "Line: $line"
done < file.txt

# break and continue
for i in {1..10}; do
    if [ $i -eq 3 ]; then continue; fi
    if [ $i -eq 7 ]; then break; fi
    echo "$i"
done
```

**Why `while IFS= read -r line` (very common interview "gotcha" question):** `IFS=` prevents leading/trailing whitespace from being stripped from each line. `-r` prevents backslash characters from being interpreted/escaped (treats them literally) — without both, you can silently corrupt lines containing backslashes or significant whitespace. This is the standard, correct idiom for reading a file line-by-line in bash.

---

## 10. Case Statements

```bash
read -p "Enter a fruit: " fruit

case "$fruit" in
    apple)
        echo "It's an apple"
        ;;
    banana|plantain)
        echo "It's a banana or plantain"
        ;;
    [Aa]vocado)
        echo "It's an avocado (case-insensitive first letter)"
        ;;
    *)
        echo "Unknown fruit"
        ;;
esac
```

Useful for cleaner multi-branch logic than a long `if/elif` chain, especially with pattern matching (`|` for OR, `*` as a wildcard/default case, glob-style patterns in each branch).

---

## 11. Functions

```bash
greet() {
    local name=$1            # always use 'local' for function-scoped variables
    echo "Hello, $name!"
    return 0                    # exit status (0-255), NOT a return value in the programming sense
}

greet "Alice"

# "Returning" a value — capture stdout via command substitution
get_sum() {
    local a=$1
    local b=$2
    echo $((a + b))             # function "returns" by printing to stdout
}
result=$(get_sum 5 3)
echo "Sum: $result"

# Functions can be recursive
factorial() {
    local n=$1
    if [ "$n" -le 1 ]; then
        echo 1
    else
        local prev=$(factorial $((n - 1)))
        echo $((n * prev))
    fi
}
```

**Why bash functions can't "return" arbitrary values (interview point):** The `return` keyword in bash only sets the function's **exit status** (an integer 0-255, like any command), not an arbitrary data value. To get an actual computed value out of a function, the conventional pattern is to `echo` the value and capture it with `$(function_name args)` from the caller, or to set a global variable inside the function.

---

## 12. Script Arguments & getopts

```bash
#!/bin/bash
echo "Script name: $0"
echo "First arg: $1"
echo "All args: $@"
echo "Number of args: $#"

shift          # shifts $2->$1, $3->$2, etc. (useful when processing args one at a time in a loop)
```

**Parsing flags with `getopts` (the standard bash way to handle `-x value` style options):**
```bash
#!/bin/bash
while getopts "n:a:vh" opt; do
    case $opt in
        n) name="$OPTARG" ;;
        a) age="$OPTARG" ;;
        v) verbose=true ;;
        h) echo "Usage: $0 -n name -a age [-v]"; exit 0 ;;
        \?) echo "Invalid option: -$OPTARG"; exit 1 ;;
    esac
done

echo "Name: $name, Age: $age, Verbose: $verbose"
# Usage: ./script.sh -n Alice -a 30 -v
```
A colon after a letter in the `getopts` string (e.g., `n:`) means that option **requires an argument** (captured in `$OPTARG`); no colon means it's a simple boolean flag.

---

## 13. Input/Output & Redirection

```bash
command > file.txt          # stdout, overwrite
command >> file.txt           # stdout, append
command 2> error.log            # stderr only
command > out.log 2>&1            # both stdout and stderr to the same file
command &> all.log                  # bash shorthand for the line above
command < input.txt                   # use a file as stdin
command > /dev/null 2>&1                # discard all output

read -p "Enter your name: " name      # prompt + capture input
read -s -p "Password: " password        # -s = silent (don't echo typed characters)
read -t 5 -p "Quick! " answer             # -t = timeout in seconds
```

**File descriptors recap:** `0` = stdin, `1` = stdout, `2` = stderr. `2>&1` means "redirect file descriptor 2 to wherever file descriptor 1 currently points" — order matters: `command > file.txt 2>&1` correctly sends both streams to the file, while `command 2>&1 > file.txt` does **not** (because at the moment `2>&1` executes, fd 1 still points to the terminal, not the file yet) — a classic interview gotcha about redirection ordering.

---

## 14. Pipes & Process Substitution

```bash
command1 | command2          # standard pipe — stdout of command1 becomes stdin of command2
ps aux | grep nginx | awk '{print $2}'

# Process substitution — treat a command's output as if it were a file
diff <(sort file1.txt) <(sort file2.txt)        # compare sorted versions without temp files
cat <(echo "first") <(echo "second")

# tee — write to a file AND pass through to stdout simultaneously
command | tee output.log
command | tee -a output.log     # append instead of overwrite
command | tee output.log | grep "error"    # save full output, but also filter
```

**Process substitution (`<(...)`) (interview point):** Creates a temporary named pipe (or `/dev/fd/N` file descriptor) representing a command's output, allowing that output to be used **as if it were a filename** — useful for commands like `diff` that expect file arguments, without needing to manually create and clean up temporary files.

---

## 15. Exit Codes & Error Handling

```bash
command
echo $?          # 0 = success, non-zero = failure (specific meaning varies by command)

if command; then
    echo "succeeded"
else
    echo "failed with exit code $?"
fi

exit 0           # exit the script successfully
exit 1            # exit with a generic failure code

# Custom exit codes for different failure reasons
if [ ! -f "$CONFIG_FILE" ]; then
    echo "Error: config file not found" >&2     # error messages should go to stderr
    exit 2
fi
```

**Convention:** `0` = success, `1-255` = various failure meanings (no universal standard beyond 0=success, though some specific codes like `126` "command found but not executable" and `127` "command not found" are conventional). Always send error messages to **stderr** (`>&2`), not stdout, so they can be filtered/redirected independently of normal output.

---

## 16. set Options (Strict Mode)

```bash
#!/bin/bash
set -e            # exit immediately if any command fails (non-zero exit)
set -u             # treat unset variables as an error (instead of silently expanding to empty)
set -o pipefail      # a pipeline's exit code is the rightmost failing command, not just the last one
set -x                # print each command before executing it (debugging trace)

# Commonly combined into "strict mode":
set -euo pipefail
```

**Why `set -euo pipefail` is recommended (very common interview/real-world topic):**
- **`-e`** — without it, a script silently continues even after a command fails, potentially cascading into worse problems; with it, the script stops immediately at the first failure (though there are well-known exceptions/gotchas, e.g., it doesn't trigger inside `if` conditions or for commands before `&&`/`||`).
- **`-u`** — catches typos in variable names (a misspelled `$VAIRABLE` would otherwise silently expand to an empty string instead of erroring).
- **`-o pipefail`** — without it, `false | true` reports success (exit 0) because only the *last* command's exit code matters by default in a pipeline; `pipefail` makes the pipeline fail if **any** command in it fails — critical for catching errors hidden in the middle of a pipe chain.

```bash
set +e      # turn OFF a previously set option (e.g., temporarily allow a command to fail)
some_command_that_might_fail
set -e       # turn it back on
```

---

## 17. Trap & Signal Handling

```bash
#!/bin/bash
cleanup() {
    echo "Cleaning up temporary files..."
    rm -f /tmp/myscript.lock
}
trap cleanup EXIT              # run cleanup whenever the script exits, for ANY reason (success, error, signal)

trap 'echo "Interrupted!"; exit 1' SIGINT     # custom handling for Ctrl+C

# Common pattern: lock file with guaranteed cleanup
LOCKFILE=/tmp/myscript.lock
if [ -f "$LOCKFILE" ]; then
    echo "Script already running"
    exit 1
fi
touch "$LOCKFILE"
trap 'rm -f "$LOCKFILE"' EXIT
# ... rest of script ...
# lock file is automatically removed even if the script crashes or is killed via SIGTERM
```

**`trap ... EXIT` (very useful, commonly tested pattern):** A trap on the special `EXIT` pseudo-signal runs the specified command whenever the script exits, **regardless of how** it exits (normal completion, `exit` call, or most signals) — the standard, reliable way to guarantee cleanup code (removing lock files, temp directories, restoring state) runs even if the script fails partway through or is interrupted.

---

## 18. Here-Documents & Here-Strings

```bash
# Here-document — multi-line string input
cat <<EOF
This is line 1
This is line 2, with a variable: $HOME
EOF

# Quoted delimiter — prevents variable expansion (treats content literally)
cat <<'EOF'
This $VAR will NOT be expanded
EOF

# Common use: generating a config file
cat <<EOF > /etc/myapp/config.conf
host=localhost
port=8080
env=$ENVIRONMENT
EOF

# Here-string — single-line input (shorter than echoing + piping)
grep "pattern" <<< "$some_variable"
```

`<<EOF ... EOF` feeds multi-line text directly as input — extremely common for generating config files, embedded SQL, or multi-line messages from within a script without needing a separate template file.

---

## 19. Subshells vs Current Shell

```bash
(cd /tmp && ls)        # runs in a SUBSHELL — the cd doesn't affect the parent shell's directory
pwd                       # still your original directory

cd /tmp; ls                # runs in the CURRENT shell — cd DOES persist after this line
pwd                          # now actually in /tmp

VAR=1
(VAR=2; echo "in subshell: $VAR")     # prints "in subshell: 2"
echo "outside: $VAR"                     # still prints "outside: 1" — subshell changes don't leak out
```

**When subshells happen implicitly:** Parentheses `( )` always spawn a subshell; pipelines (`cmd1 | cmd2`) run each command in a subshell in most shells (bash included, by default) — which is exactly why a `while read` loop piped from a command (`cmd | while read line; do VAR=$line; done`) **loses** any variable changes made inside it once the pipeline finishes (a classic bash gotcha — workaround: use process substitution `< <(cmd)` or redirect input instead of piping).

```bash
# Gotcha example:
count=0
cat file.txt | while read line; do ((count++)); done
echo "$count"     # prints 0! the while loop ran in a subshell, count++ didn't persist

# Fix using process substitution instead:
count=0
while read line; do ((count++)); done < <(cat file.txt)
echo "$count"     # correctly prints the actual count
```

---

## 20. Regular Expressions in Bash

```bash
# =~ operator inside [[ ]] — bash's regex match
if [[ "$email" =~ ^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$ ]]; then
    echo "Valid email format"
fi

# Capture groups via BASH_REMATCH
if [[ "2024-06-15" =~ ([0-9]{4})-([0-9]{2})-([0-9]{2}) ]]; then
    echo "Year: ${BASH_REMATCH[1]}"
    echo "Month: ${BASH_REMATCH[2]}"
    echo "Day: ${BASH_REMATCH[3]}"
fi
```

**Important:** The pattern on the right side of `=~` should generally be **unquoted** (or stored in a variable first) — quoting it can cause bash to treat it as a literal string match instead of a regex in some bash versions, a subtle and commonly-tested gotcha.

---

## 21. File Testing & Globbing

```bash
# Globbing (wildcard expansion, happens BEFORE the command runs, performed by the shell itself)
ls *.txt              # all .txt files
ls file?.txt            # file1.txt, file2.txt, etc. (? = exactly one character)
ls file[12].txt           # file1.txt or file2.txt
ls file[!12].txt            # any file NOT file1.txt or file2.txt

shopt -s globstar       # enable ** for recursive matching (bash 4+)
ls **/*.log                # all .log files recursively in subdirectories

shopt -s nullglob        # makes a non-matching glob expand to nothing instead of the literal pattern string
shopt -s dotglob           # include hidden (dotfiles) in glob matches
```

**Globbing vs Regex (important interview distinction):** Glob patterns (`*`, `?`, `[...]`) are expanded by the **shell itself** before the command even runs, and are used for matching **filenames**. Regular expressions (used by `grep`, `sed`, `awk`, or bash's `=~`) are a more powerful pattern language used for matching **text content**, evaluated by the *tool*, not the shell. `*` means "zero or more characters" in a glob, but means "zero or more of the *preceding character*" in regex — a very common point of confusion.

---

## 22. Reading Files & Input

```bash
# Line by line (the correct, robust way)
while IFS= read -r line; do
    echo "Processing: $line"
done < input.txt

# Reading a whole file into a variable
content=$(cat file.txt)
content=$(<file.txt)        # slightly more efficient bash-only shortcut, avoids spawning cat

# Reading into an array, one line per element
mapfile -t lines < file.txt          # bash 4+, modern preferred way
readarray -t lines < file.txt          # synonym for mapfile

# Reading CSV-like delimited data
while IFS=',' read -r col1 col2 col3; do
    echo "Col1: $col1, Col2: $col2, Col3: $col3"
done < data.csv
```

---

## 23. Common Utilities Used in Scripts

```bash
date +%Y-%m-%d              # format dates
date -d "2 days ago"           # relative date arithmetic (GNU date)
sleep 5                          # pause execution for 5 seconds
basename /path/to/file.txt         # → file.txt
dirname /path/to/file.txt            # → /path/to
realpath ./relative/path               # resolve to an absolute path
mktemp                                   # create a unique temp file safely
mktemp -d                                  # create a unique temp directory
xargs                                        # build/execute commands from stdin items
seq 1 10                                       # generate a sequence of numbers
column -t                                        # align columnar text output nicely
```

```bash
# Common xargs patterns
find . -name "*.tmp" -print0 | xargs -0 rm        # -print0/-0 safely handle filenames with spaces/special chars
echo "a b c" | xargs -n1 echo                         # run echo once per word
cat urls.txt | xargs -P 4 -I {} curl -O {}              # run 4 in parallel, {} = placeholder for each input line
```

---

## 24. Debugging Scripts

```bash
bash -x script.sh             # run with execution trace (prints each command with expansions before running it)
set -x                          # turn tracing on mid-script
set +x                            # turn tracing off mid-script

bash -n script.sh              # syntax check only, don't actually execute (dry run for parse errors)

PS4='+ ${BASH_SOURCE}:${LINENO}: '    # customize the trace prefix to include file/line number (very useful in larger scripts)
```

**Common debugging techniques:**
- Add `echo "DEBUG: variable=$variable"` statements at key points
- Use `set -x`/`set +x` to bracket just the suspicious section of a long script
- Use `shellcheck script.sh` — a static analysis linter that catches an enormous range of common bash mistakes (unquoted variables, wrong test operators, unreachable code) before you even run the script — **extremely commonly recommended in real-world workflows and worth mentioning in interviews**

---

## 25. sh vs bash vs zsh

| Shell | Notes |
|---|---|
| **sh** | The POSIX standard shell interface — on many Linux systems `/bin/sh` is actually a symlink to `dash` (Debian/Ubuntu) or `bash` running in POSIX-compatibility mode, not a distinct shell itself |
| **bash** | Bourne Again Shell — the most common default interactive/scripting shell on Linux, adds many conveniences beyond POSIX (`[[ ]]`, arrays, `{1..5}` brace expansion, `+=`, process substitution) |
| **dash** | A minimal, fast, POSIX-compliant shell — often used as `/bin/sh` for faster script execution (e.g., boot scripts), but lacks many bash-only features |
| **zsh** | A more feature-rich interactive shell (default on modern macOS), mostly bash-compatible for scripting but with some differences (e.g., array indexing starts at 1, not 0) |

**`#!/bin/sh` vs `#!/bin/bash` (interview point):** If a script uses bash-specific features (arrays, `[[ ]]`, `{1..5}`, `+=`, `local`) but has a `#!/bin/sh` shebang, it may fail or behave unexpectedly on systems where `/bin/sh` points to a stricter POSIX shell like `dash` — always match the shebang to the actual feature set you're using, and use `#!/bin/bash` explicitly whenever you rely on bash-specific syntax.

---

## 26. Common Patterns & Idioms

**Check if a command exists before using it:**
```bash
if command -v docker &> /dev/null; then
    echo "Docker is installed"
else
    echo "Docker not found"
fi
```

**Default argument values:**
```bash
ENVIRONMENT="${1:-dev}"      # use $1 if provided, else default to "dev"
```

**Retry logic:**
```bash
retry() {
    local n=0
    local max=5
    until [ "$n" -ge "$max" ]; do
        "$@" && break
        n=$((n+1))
        echo "Attempt $n failed, retrying..."
        sleep 2
    done
}
retry curl -sf https://example.com/health
```

**Script directory (reliable, even when called from elsewhere):**
```bash
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
```

**Confirm before a destructive action:**
```bash
read -p "Are you sure? (y/n) " confirm
if [[ "$confirm" != [yY] ]]; then
    echo "Aborted."
    exit 1
fi
```

**Logging with timestamps:**
```bash
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}
log "Starting deployment..."
```

**Parallel execution with wait:**
```bash
task1 &
task2 &
task3 &
wait              # block until ALL background jobs finish
echo "All tasks complete"
```

---

## 27. Security Considerations

1. **Never `eval` untrusted input** — `eval` executes a string as a full command, a classic injection vector if any part of that string comes from user/external input.
2. **Always quote variables**, especially when they hold filenames or come from external input (prevents word-splitting/glob-expansion exploits and unintended command injection in some contexts).
3. **Validate/sanitize input** before using it in destructive operations (`rm`, `mv`, database commands).
4. **Avoid `rm -rf $VAR`** without quotes/validation — an empty or unexpected `$VAR` (e.g., due to a typo or unset variable) can expand to something catastrophic like `rm -rf /` in the worst case. Combine with `set -u` to catch unset-variable cases.
5. **Use `mktemp`** instead of hardcoded predictable temp file paths (avoids race conditions/symlink attacks where another process could pre-create a malicious file at a predictable path).
6. **Don't hardcode secrets/passwords** directly in scripts — use environment variables, a secrets manager, or prompt securely (`read -s`).
7. **Be careful with scripts run via `sudo` or as root** — any vulnerability in the script becomes a privilege escalation vector.

```bash
# Dangerous
rm -rf $DIR/*

# Safer
if [ -n "$DIR" ] && [ -d "$DIR" ]; then
    rm -rf "${DIR:?}"/*
fi
```
(`${DIR:?}` causes an immediate error and script exit if `DIR` is unset/empty, rather than silently proceeding with a dangerously empty/wildcard path.)

---

## 28. Best Practices Summary

- Always start scripts with `#!/bin/bash` and `set -euo pipefail`
- Always quote variable expansions: `"$VAR"`, not `$VAR`
- Use `[[ ]]` over `[ ]` in bash-specific scripts for safer parsing
- Use `local` for all variables inside functions
- Use `"$@"` (not `$*`) when forwarding arguments to another command
- Use `mktemp`/`mktemp -d` for temporary files, with a `trap ... EXIT` to clean them up
- Run scripts through `shellcheck` before considering them done
- Prefer `$(...)` over backtick command substitution
- Use meaningful exit codes and send error messages to stderr (`>&2`)
- Add comments explaining *why*, not just *what*, for non-obvious logic
- Keep scripts focused — break large scripts into functions, or separate files/tools if they grow too complex
- Test with edge cases: empty input, missing files, special characters in filenames

---

## 29. Cheat Sheet

```bash
#!/bin/bash
set -euo pipefail

# Variables
VAR="value"
echo "${VAR:-default}"

# Conditionals
if [[ -f "$file" && "$count" -gt 0 ]]; then ... fi

# Loops
for i in {1..5}; do ... done
while IFS= read -r line; do ... done < file.txt

# Functions
my_func() { local x=$1; echo "$x"; }
result=$(my_func "arg")

# Arrays
arr=(a b c)
for item in "${arr[@]}"; do ... done

# String ops
${str:0:5}      ${str/old/new}      ${str^^}      ${str,,}

# Arithmetic
result=$((a + b))

# Error handling
trap cleanup EXIT
command || { echo "failed" >&2; exit 1; }

# Debug
bash -x script.sh
shellcheck script.sh
```

---

## 30. Interview Questions & Answers

**Q1: What's the difference between `$@` and `$*`?**
A: When quoted as `"$@"`, all positional arguments are expanded as **separate, individually-quoted words**, correctly preserving arguments that contain spaces. When quoted as `"$*"`, all arguments are joined into a **single string** separated by the first character of `$IFS` (space by default). `"$@"` is almost always the correct choice when forwarding arguments to another command.

**Q2: Why should you always quote variable expansions in bash?**
A: Without quotes, an unquoted `$VAR` undergoes word splitting (on whitespace/`$IFS`) and glob expansion (`*`/`?` matched against filenames) — this is a common source of bugs, especially with filenames or values containing spaces (e.g., `rm $FILE` could try to delete multiple unintended "files" if `$FILE` contains a space). Quoting (`"$VAR"`) prevents both, treating the variable's value as a single literal string.

**Q3: What's the difference between `[ ]` and `[[ ]]`?**
A: `[ ]` is the POSIX `test` command, available in all shells but more error-prone — it word-splits/glob-expands unquoted variables, requires escaping `<`/`>` for string comparisons, and has no native `&&`/`||` inside it. `[[ ]]` is a bash keyword with safer parsing (no word-splitting issues with unquoted variables inside it), native logical operators, and pattern/regex matching support — generally preferred in bash-specific scripts.

**Q4: Explain `set -e`, `set -u`, and `set -o pipefail`, and why combine them.**
A: `-e` makes the script exit immediately if any command fails (non-zero exit), preventing it from silently continuing after an error. `-u` treats references to unset variables as an error rather than silently expanding to an empty string, catching typos. `-o pipefail` makes a pipeline's exit status reflect the **rightmost failing command** rather than just the last command — without it, `false | true` reports success since only the last command's exit code matters by default. Combined as `set -euo pipefail`, these form a much safer "strict mode" baseline for production scripts.

**Q5: What's the difference between running a script normally versus `source`-ing it?**
A: Running a script (`./script.sh` or `bash script.sh`) executes it in a new child process — any variable assignments or `cd` changes it makes don't persist in your current shell. `source`-ing it (`source script.sh` or `. script.sh`) runs it **within your current shell session**, so any variables, functions, or directory changes it makes persist afterward — this is why shell config files like `.bashrc` are sourced, not executed.

**Q6: How do you read a file line by line correctly in bash, and what can go wrong if you don't do it carefully?**
A: The robust pattern is `while IFS= read -r line; do ... done < file.txt`. Without `IFS=`, leading/trailing whitespace on each line gets stripped; without `-r`, backslash characters in the line get interpreted/escaped instead of being treated literally — both can silently corrupt data depending on file content. Piping into the loop instead of redirecting (`cat file | while read ...`) also causes the loop to run in a subshell, so any variables modified inside the loop won't persist after it ends.

**Q7: What is process substitution and when would you use it?**
A: `<(command)` lets a command's output be treated as if it were a filename — useful for tools like `diff` that expect file arguments but where you want to compare command output directly (`diff <(sort a.txt) <(sort b.txt)`) without manually creating and cleaning up temporary files. It's implemented via a temporary named pipe or `/dev/fd/N` file descriptor under the hood.

**Q8: Why does a `while read` loop sometimes lose variable changes after it finishes?**
A: When the loop is fed via a pipe (`command | while read line; do ...; done`), bash runs the loop in a **subshell** (since pipelines spawn subshells for each stage), so any variables modified inside the loop are local to that subshell and disappear once the pipeline completes. The fix is to avoid the pipe and instead redirect input directly: `while read line; do ...; done < <(command)` (process substitution) or `< file.txt`, both of which run the loop in the current shell.

**Q9: How would you safely handle a destructive command like `rm -rf` involving a variable?**
A: Always quote the variable, validate it's non-empty and points to an expected location before the destructive operation, and consider using `${VAR:?}` to force an immediate error if the variable is unset/empty rather than silently expanding to something dangerous (e.g., `rm -rf "${DIR:?}"/*` errors out immediately if `DIR` is unset, instead of potentially executing `rm -rf /*`). Combining this with `set -u` adds another layer of protection against unset-variable mistakes.

**Q10: What's the difference between `$(command)` and backticks?**
A: Both perform command substitution (capturing a command's stdout into a string), but `$(command)` is the modern, POSIX-compliant syntax that nests cleanly (`$(cmd1 $(cmd2))`) and is generally more readable. Backticks (`` `command` ``) are the older, legacy syntax — harder to nest (requires escaping inner backticks) and generally discouraged in new scripts.

**Q11: How do you pass command-line flags/options to a bash script, and what does `getopts` do?**
A: Positional arguments are accessible via `$1`, `$2`, etc. For more structured flag-based options (like `-n value -v`), `getopts` is the standard built-in for parsing single-character flags in a loop, distinguishing flags that require an argument (marked with a trailing `:` in the option string, captured via `$OPTARG`) from simple boolean flags.

**Q12: What does `trap cleanup EXIT` do, and why is it useful?**
A: It registers the `cleanup` function (or command) to run automatically whenever the script exits, for virtually any reason — normal completion, an explicit `exit` call, or most signals/errors. It's the standard, reliable pattern for guaranteeing cleanup logic (removing lock files, temp directories) runs even if the script fails partway through or is interrupted, rather than relying on cleanup code only at the very end of the script (which would be skipped if the script exits early or crashes).

**Q13: What's the difference between a glob pattern and a regular expression?**
A: Glob patterns (`*`, `?`, `[...]`) are expanded by the **shell itself**, before the command even runs, and are used to match filenames in commands like `ls *.txt`. Regular expressions are a more powerful pattern-matching language interpreted by specific tools (`grep`, `sed`, `awk`, or bash's `=~` operator) for matching text content — critically, `*` means something different in each: "zero or more of the preceding character" in regex, versus "zero or more of any character" in a glob.

**Q14: How would you debug a misbehaving bash script?**
A: Run it with `bash -x script.sh` (or add `set -x`/`set +x` around the suspicious section) to print each command with its expanded variables as it executes, helping pinpoint where behavior diverges from expectations. Use `bash -n script.sh` for a syntax-only check without execution. Run `shellcheck script.sh` to catch common mistakes statically before even running it (unquoted variables, wrong test operators, etc.) — a very commonly recommended real-world practice.

**Q15: What's the difference between `local` and a regular variable assignment inside a function?**
A: A regular assignment inside a function (without `local`) creates or modifies a **global** variable, visible and persisting outside the function — often an unintended side effect. `local` scopes the variable strictly to that function's execution, automatically "unsetting" it once the function returns, preventing naming collisions and unintended state leakage between functions — best practice is to use `local` for essentially all variables defined inside a function unless you specifically intend to set something globally.

**Q16: How do you "return" a value from a bash function, given that `return` only sets an exit code?**
A: The conventional pattern is to have the function `echo` (print) the value to stdout, and have the caller capture it via command substitution: `result=$(my_function args)`. The `return` keyword itself only sets the function's numeric exit status (0-255, accessible via `$?`), used to indicate success/failure, not for returning arbitrary computed values.

**Q17: What's the difference between `2>&1 > file.txt` and `> file.txt 2>&1`, and why does the order matter?**
A: `> file.txt 2>&1` first redirects stdout (fd 1) to the file, then redirects stderr (fd 2) to wherever fd 1 *currently* points (the file) — correctly sending both streams to the file. `2>&1 > file.txt` first redirects stderr to wherever stdout *currently* points (still the terminal at that moment), and only *then* redirects stdout to the file — leaving stderr still going to the terminal while stdout goes to the file. Redirections are processed left to right, which is why the order matters here.

**Q18: What is ShellCheck and why would you use it?**
A: ShellCheck is a static analysis tool/linter for shell scripts that detects a very wide range of common mistakes — unquoted variables, incorrect test operators, unreachable code, subtle quoting issues, portability problems between `sh` and `bash` — before you even run the script. It's widely used in CI pipelines and considered a best practice for any non-trivial shell script, since many bash bugs are subtle and easy to miss in manual review.

**Q19: How would you make a script that runs multiple long-running tasks in parallel and waits for all of them to finish?**
A: Launch each task in the background with `&`, then use the `wait` builtin (with no arguments) to block until all background jobs launched by the current shell have completed: `task1 & task2 & task3 & wait`. For more control (capturing individual exit codes, limiting concurrency), tools like `xargs -P` or GNU `parallel` are often used for more complex parallel workloads.

**Q20: What's the difference between `#!/bin/sh` and `#!/bin/bash`, and why does it matter?**
A: `#!/bin/bash` explicitly invokes bash, giving you access to bash-specific features (arrays, `[[ ]]`, brace expansion `{1..5}`, `local`, `+=`). `#!/bin/sh` invokes whatever the system's default POSIX shell is — on many Linux distributions (notably Debian/Ubuntu) this is actually `dash`, a more minimal POSIX-compliant shell that does **not** support bash-specific syntax. A script using bash features but shebanged with `#!/bin/sh` may fail or behave unexpectedly on such systems — always match the shebang to the actual feature set the script relies on.

---

### Final interview tip
Be ready to **write a script live** that takes command-line arguments, reads a file line-by-line correctly, includes proper error handling (`set -euo pipefail`, a `trap` for cleanup), and explain **why each defensive choice matters** as you write it. Also expect at least one "what's wrong with this script" debugging question — commonly featuring an unquoted variable, a `while read` piped into a subshell losing state, or a missing `set -e`/`pipefail` masking a silent failure.
