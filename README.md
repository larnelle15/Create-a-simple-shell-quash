Quash Shell — Design & Implementation

Author: Larnelle Ankunda
Course: Operating Systems
Primary file: shell.c
Language: C

:bookmark_tabs: Summary

Quash is a small UNIX-style shell that runs both built-in and external programs. It supports:

process creation via fork/execvp

foreground/background jobs

signal handling (SIGINT, SIGALRM)

10-second timeouts for foreground jobs

basic I/O redirection (<, >)

The project was built incrementally to showcase core OS ideas—process control, signals, file descriptors, and environment variables—while keeping the code modular and easy to read.

:dart: Goals

Modular structure — helpers for prompt, tokenization, built-ins, process control, signals, and I/O.

Readable first — straightforward, instructional code.

Robust — clear error messages and safe failure paths.

POSIX-only — standard system calls, no non-portable tricks.

Signal-safe — handlers avoid unsafe work; children restore defaults when needed.

:gear: High-Level Architecture
readline → tokenize → classify (built-in vs external)
        ↘ built-in: run handler and return
         ↘ external: fork → (child) execvp
                               ↳ (optional) set up < / > redirection
                     (parent) waitpid unless '&' → background

Main loop (overview)

Show prompt with getcwd() (format: /path/to/dir>).

fgets() a line and break it into argv-style tokens.

Expand $VAR tokens using getenv().

If command is built-in → run its handler.

Otherwise fork+exec; wait unless it’s a background job.

:rocket: Implemented Tasks (1–7)
Task 1 — Prompt & Built-ins

Prompt looks like:

/home/codio>


Built-ins:

Command	What it does
cd [dir]	Change directory (chdir)
pwd	Print current directory
echo [args]	Echo arguments; expands $VAR
setenv VAR VAL	Set an environment variable
env [VAR]	Show all or a specific variable
exit	Cleanly terminate the shell

Each built-in has its own function (e.g., builtin_cd, builtin_env) to keep things explicit.

Task 2 — Tokenization & $VAR Expansion

A dedicated tokenize() breaks the input into an argv vector.

Tokens beginning with $ are replaced by getenv() results, so you can do:

echo $HOME
cd $HOME
setenv greeting $USER

Task 3 — Running External Programs

Minimal pattern:

pid_t pid = fork();
if (pid == 0) {
    execvp(argv[0], argv);      // child: replace image
    perror("execvp");           // only runs if exec fails
    _exit(127);
} else if (pid > 0) {
    int status;
    waitpid(pid, &status, 0);   // parent: wait (unless background)
} else {
    perror("fork");
}


The child inherits descriptors; the parent reports failures (e.g., “No such file or directory”).

Task 4 — Background Jobs (&)

A trailing & runs the command without blocking:

/home/codio> ./a.out &
[background pid 1234]


The parent skips waitpid() so the prompt returns immediately.

Task 5 — Ctrl-C (SIGINT) Behavior

Problem: Ctrl-C used to kill both the running program and the shell.
Fix:

Install on_sigint() with sigaction.

Handler prints a newline and redraws the prompt; the shell keeps running.

Children restore default handlers so Ctrl-C still interrupts them.

Result:

/home/codio> ./a.out
...running...
^C
/home/codio>

Task 6 — Foreground Timeout (10s)

Parent sets alarm(10) before waitpid().

If time expires:

kill(pid, SIGTERM);
kill(pid, SIGKILL);


If the process ends early, alarm(0) cancels the timer.

Prevents runaway or hung jobs from blocking the shell.

Task 7 — I/O Redirection (<, >)

Handled in the child before execvp():

stdout to file (>):

cat shell.c > output.txt


stdin from file (<):

more < shell.c


Core idea:

int fd = open(filename, O_CREAT | O_TRUNC | O_WRONLY, 0644);
dup2(fd, STDOUT_FILENO);
close(fd);

:warning: Error Handling

Every system call is checked; errors are printed via perror().

Syntax issues like a missing filename after < or > are detected.

The shell survives failed commands and returns to the prompt.

Example messages:

syntax error: expected file after '>'
open: No such file or directory

:triangular_ruler: Design Notes

Small, focused helpers keep each feature isolated.

Easy to extend toward pipes (|) and job control.

Pedagogical: demonstrates classic OS primitives in a compact codebase.

:keyboard: Example Session
/home/codio> pwd
/home/codio
/home/codio> echo hello $USER
hello codio
/home/codio> ./task &
[background pid 4242]
/home/codio> cat shell.c > out.txt
/home/codio> more < out.txt
...

:framed_picture: Illustration Placeholder

Drop your diagram here (e.g., a flowchart of the main loop, or a state machine for signal/timeout handling).

:checkered_flag: Conclusion

Over the course of the project, Quash grew from a bare prompt into a functional shell that:

runs foreground and background jobs

responds sanely to Ctrl-C

enforces a 10-second limit on foreground tasks

redirects input/output with < and >

The emphasis on modular functions, POSIX calls, and clear control flow makes Quash a solid teaching tool and a good base for future upgrades (pipelines, job control, history, completion).

End of Report

If you want this as a pre-styled README.md file, say the word and I’ll generate the file for download.
