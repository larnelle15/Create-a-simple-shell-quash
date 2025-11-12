# Quash Shell — Design & Implementation

**Author:** Larnelle Ankunda  
**Course:** Operating Systems  
**Primary File:** `shell.c`  
**Language:** C

---

## 📑 Overview

**Quash** is a lightweight, UNIX-style shell. It can run built-in commands as well as external programs and supports:

- Process creation with `fork` / `execvp`
- Foreground and background execution
- Signal handling (`SIGINT`, `SIGALRM`)
- A 10‑second timeout for foreground jobs
- Basic I/O redirection (`<`, `>`)

This project was developed incrementally to highlight key OS concepts—processes, signals, file descriptors, and environment variables—while keeping the codebase clear and modular.

---

## 🎯 Project Goals

- **Modularity:** Separate helpers for the prompt, tokenization, built-ins, process control, signals, and I/O.  
- **Clarity:** Prefer straightforward, instructional code over clever tricks.  
- **Robustness:** Defensive error handling and graceful recovery paths.  
- **POSIX Compliance:** Stick to standard interfaces for portability.  
- **Signal Safety:** Use safe handlers; children reset to defaults where appropriate.

---

## ⚙️ Architecture at a Glance

```text
readline
  └─→ tokenize
        └─→ classify (built-in vs external)
               ├─→ built-in: run handler and return
               └─→ external:
                     ├─ child:  [optional <, > redirection] → execvp
                     └─ parent: waitpid (unless '&' → run in background)
```

### Main Loop (summary)

1. Render prompt with `getcwd()` (format: `/path/to/dir>`).  
2. Read a line via `fgets()` and split into argv-style tokens.  
3. Expand `$VAR` tokens using `getenv()`.  
4. Dispatch to a built-in **or** fork/exec an external program.  
5. If not a background job, `waitpid()` for completion.

---

## 🚀 Implemented Tasks

### 1) Prompt & Built-ins

Prompt example:

```text
/home/codio>
```

Built-ins:

| Command            | Description                                 |
|--------------------|---------------------------------------------|
| `cd [dir]`         | Change the working directory (`chdir`)      |
| `pwd`              | Print the current directory                  |
| `echo [args]`      | Print arguments; expands `$VAR`              |
| `setenv VAR VALUE` | Set an environment variable                  |
| `env [VAR]`        | Show all env vars or a specific one          |
| `exit`             | Exit the shell cleanly                       |

> Each built-in has its own handler (e.g., `builtin_cd`, `builtin_env`) to keep responsibilities clear.

---

### 2) Tokenization & `$VAR` Expansion

- A dedicated `tokenize()` produces an argv-like vector.  
- Tokens beginning with `$` are replaced using `getenv()`:

```sh
echo $HOME
cd $HOME
setenv greeting $USER
```

---

### 3) Executing External Programs

Typical flow:

```c
pid_t pid = fork();
if (pid == 0) {
    execvp(argv[0], argv);      // child: replace image
    perror("execvp");           // only executes on failure
    _exit(127);
} else if (pid > 0) {
    int status;
    waitpid(pid, &status, 0);   // parent: wait (unless background)
} else {
    perror("fork");
}
```

- The child inherits file descriptors; the parent reports failures (e.g., “No such file or directory”).

---

### 4) Background Jobs (`&`)

A trailing `&` prevents the shell from blocking:

```text
/home/codio> ./a.out &
[background pid 1234]
```

- The parent skips `waitpid()` and returns to the prompt immediately.

---

### 5) `Ctrl-C` (SIGINT) Behavior

**Initial issue:** `Ctrl-C` terminated both the shell and the running job.  
**Resolution:**

- Install `on_sigint()` with `sigaction`.  
- Handler prints a newline and redraws the prompt without exiting.  
- Child processes restore default handlers so `Ctrl-C` still interrupts them.

Example:

```text
/home/codio> ./a.out
...running...
^C
/home/codio>
```

---

### 6) Foreground Timeout (10s)

- The parent sets `alarm(10)` before `waitpid()`.  
- If the timer expires:

```c
kill(pid, SIGTERM);
kill(pid, SIGKILL);
```

- If the child exits early, call `alarm(0)` to cancel.  
- Prevents hung or runaway tasks from blocking the shell.

---

### 7) I/O Redirection (`<`, `>`)

Handled in the **child** process before `execvp()`.

**stdout to file (`>`):**
```sh
cat shell.c > output.txt
```

**stdin from file (`<`):**
```sh
more < shell.c
```

Core steps:

```c
int fd = open(filename, O_CREAT | O_TRUNC | O_WRONLY, 0644);
dup2(fd, STDOUT_FILENO);
close(fd);
```

---

## ⚠️ Error Handling

- Every system call is checked and surfaced via `perror()`.  
- Syntax mistakes (e.g., missing filename after `<` or `>`) are detected.  
- The shell stays alive and returns to the prompt after failures.

Example messages:

```text
syntax error: expected file after '>'
open: No such file or directory
```

---

## 📐 Design Notes

- **Focused helpers** keep features decoupled and testable.  
- **Extension-friendly:** natural next steps include pipelines (`|`) and job control.  
- **Teaching-friendly:** demonstrates essential OS primitives with minimal surface area.

---

## ⌨️ Example Session

```text
/home/codio> pwd
/home/codio
/home/codio> echo hello $USER
hello codio
/home/codio> ./task &
[background pid 4242]
/home/codio> cat shell.c > out.txt
/home/codio> more < out.txt
...
```

---

## 🖼️ Illustration Placeholder

> **Insert your diagram here** — e.g., a flowchart of the main loop or a small state machine for timeout/signal handling.

---

## 🏁 Conclusion

Across iterations, Quash evolved from a simple prompt into a practical shell that:

- runs foreground and background jobs
- handles `Ctrl-C` sensibly
- enforces a 10‑second cap on foreground tasks
- supports `<` / `>` redirection

The emphasis on modularity, POSIX calls, and transparent control flow makes Quash both an effective learning tool and a solid foundation for future upgrades (pipelines, job control, history, completion).

---

_End of Report_
