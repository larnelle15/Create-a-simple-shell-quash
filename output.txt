#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>
#include <fcntl.h>
#include <signal.h>
#include <errno.h>
#include <limits.h>
#include <unistd.h>
#include <sys/stat.h>

#define MAX_COMMAND_LINE_LEN 1024
#define MAX_COMMAND_LINE_ARGS 128
#define MAX_CMDS_IN_PIPE 16

char prompt[] = "> ";
char delimiters[] = " \t\r\n";
extern char **environ;

static volatile sig_atomic_t got_sigint = 0;
static volatile sig_atomic_t timeout_fired = 0;

/*----------- HELPER FUNCTIONS -----------*/ 

static void on_sigint(int signo) {
    (void)signo;
    got_sigint = 1;
    // async-signal-safe: print a newline to move to a clean line
    write(STDOUT_FILENO, "\n", 1);
}

static void on_sigalrm(int signo) {
    (void)signo;
    timeout_fired = 1;   // just set a flag; keep handlers async-signal-safe
}

static void handle_redirection(char *arguments[], int *argc) {
  int i;
  for (i = 0; i < *argc; i++) {
    // Output redirection
    if (strcmp(arguments[i], ">") == 0) {
      if (i + 1 >= *argc) {
        fprintf(stderr, "syntax error: expected file after '>'\n");
        return;
      }
      int fd = open(arguments[i + 1], O_CREAT | O_TRUNC | O_WRONLY, 0644);
      if (fd < 0) {
        perror("open");
        return;
      }
      if (dup2(fd, STDOUT_FILENO) < 0) {
        perror("dup2");
        close(fd);
        return;
      }
      close(fd);

      arguments[i] = NULL;  // cut off args here
      *argc = i;
      break;
    }

    // Input redirection
    if (strcmp(arguments[i], "<") == 0) {
      if (i + 1 >= *argc) {
        fprintf(stderr, "syntax error: expected file after '<'\n");
        return;
      }
      int fd = open(arguments[i + 1], O_RDONLY);
      if (fd < 0) {
        perror("open");
        return;
      }
      if (dup2(fd, STDIN_FILENO) < 0) {
        perror("dup2");
        close(fd);
        return;
      }
      close(fd);

      arguments[i] = NULL;  // cut off args here
      *argc = i;
      break;
    }
  }
}

static void print_prompt(void) {
    char cwd[PATH_MAX];
    if (getcwd(cwd, sizeof(cwd)) != NULL) {
        // Format: /full/path>
        printf("%s> ", cwd);
    } else {
        // Fallback if getcwd fails
        printf("> ");
    }
    fflush(stdout);
}

// Tokenize in-place, returns argc
static int tokenize(char *line, char *arguments[], int max_args) {
    int argc = 0;
    char *tok = strtok(line, delimiters);
    while (tok && argc < max_args - 1) {
        arguments[argc++] = tok;
        tok = strtok(NULL, delimiters);
    }
    arguments[argc] = NULL;
    return argc;
}

static int builtin_cd(int argc, char *argv[]) {
    const char *target = NULL;

    if (argc < 2 || strcmp(argv[1], "~") == 0) {
        target = getenv("HOME");
        if (!target) target = "/";
    } else {
        target = argv[1];
    }
    if (chdir(target) != 0) {
        perror("cd");
        return 1;
    }
    return 0;
}

static int builtin_pwd(void) {
    char cwd[PATH_MAX];
    if (getcwd(cwd, sizeof(cwd)) == NULL) {
        perror("pwd");
        return 1;
    }
    printf("%s\n", cwd);
    return 0;
}

static int echo_one(const char *s) {
    // If begins with '$', expand environment variable (no braces support here)
    if (s[0] == '$') {
        const char *name = s + 1;
        if (*name == '\0') {
            // Plain '$' -> print nothing (or '$' if you prefer)
            return 0;
        }
        const char *val = getenv(name);
        if (val) fputs(val, stdout);
        return 0;
    } else {
        fputs(s, stdout);
        return 0;
    }
}

static int builtin_echo(int argc, char *argv[]) {
    int i;
    for (i = 1; i < argc; i++) {
        echo_one(argv[i]);
        if (i + 1 < argc) fputc(' ', stdout);
    }
    fputc('\n', stdout);
    return 0;
}

static int builtin_env(int argc, char *argv[]) {
    if (argc == 1) {
        char **e;  /* C89: declare before the loop */
        for (e = environ; *e != NULL; ++e) {
            puts(*e);
        }
        return 0;
    } else {
        int i;  /* C89: declare before the loop */
        for (i = 1; i < argc; i++) {
            const char *name = argv[i];
            const char *val = getenv(name);
            if (val) {
                puts(val);
            }
        }
        return 0;
    }
}

static int parse_key_value(const char *arg, char **out_key, char **out_val) {
    // Parses KEY=VALUE (in-place copy).
    char *eq = strchr(arg, '=');
    if (!eq) return 0;
    size_t klen = (size_t)(eq - arg);
    char *k = (char *)malloc(klen + 1);
    char *v = strdup(eq + 1);
    if (!k || !v) { free(k); free(v); return 0; }
    memcpy(k, arg, klen);
    k[klen] = '\0';
    *out_key = k;
    *out_val = v;
    return 1;
}

static int builtin_setenv(int argc, char *argv[]) {
    // Accept either: setenv KEY=VALUE  OR  setenv KEY VALUE
    if (argc < 2) {
        fprintf(stderr, "setenv usage: setenv KEY=VALUE  or  setenv KEY VALUE\n");
        return 1;
    }

    int rc = 0;

    if (argc == 2) {
        char *key = NULL, *val = NULL;
        if (!parse_key_value(argv[1], &key, &val)) {
            fprintf(stderr, "setenv: expected KEY=VALUE\n");
            return 1;
        }
        if (setenv(key, val, 1) != 0) {
            perror("setenv");
            rc = 1;
        }
        free(key);
        free(val);
        return rc;
    }

    // argc >= 3: treat as setenv KEY VALUE (ignore any extra tokens)
    const char *key = argv[1];
    const char *val = argv[2];
    if (setenv(key, val ? val : "", 1) != 0) {
        perror("setenv");
        return 1;
    }
    return 0;
}

// Declare and run builtin commands
static bool is_builtin(const char *cmd) {
    return cmd &&
           (strcmp(cmd, "cd") == 0 ||
            strcmp(cmd, "pwd") == 0 ||
            strcmp(cmd, "echo") == 0 ||
            strcmp(cmd, "exit") == 0 ||
            strcmp(cmd, "env") == 0 ||
            strcmp(cmd, "setenv") == 0);
}

static int run_builtin(int argc, char *argv[]) {
    if (argc == 0) return 0;

    if (strcmp(argv[0], "cd") == 0)        return builtin_cd(argc, argv);
    if (strcmp(argv[0], "pwd") == 0)       return builtin_pwd();
    if (strcmp(argv[0], "echo") == 0)      return builtin_echo(argc, argv);
    if (strcmp(argv[0], "env") == 0)       return builtin_env(argc, argv);
    if (strcmp(argv[0], "setenv") == 0)    return builtin_setenv(argc, argv);
    if (strcmp(argv[0], "exit") == 0)      exit(0);

    return 0;
}

static void run_external(int argc, char *argv[], bool background) {
  pid_t pid = fork();
  if (pid < 0) {
    perror("fork");
    return;
  }

  if (pid == 0) {
    // ---- Child process ----
    signal(SIGINT, SIG_DFL);   // keep child interruptible
    signal(SIGQUIT, SIG_DFL);

    // Handle redirection (< or >)
    handle_redirection(argv, &argc);

    // execute command  
    execvp(argv[0], argv);

    // exec failed
    fprintf(stderr, "execvp() failed: %s\n", strerror(errno));
    fprintf(stderr, "An error occurred.\n");
    _exit(127);
  }

  // ---- Parent process ----
  if (background) {
    printf("[Started background pid %d]\n", pid);
    fflush(stdout);
    return; // no waiting, no timer
  }

  // Foreground: set 10s timeout
  timeout_fired = 0;
  alarm(10);

  int status = 0;
  for (;;) {
    pid_t w = waitpid(pid, &status, 0);
    if (w == pid) {
      // Child finished before timeout
      alarm(0);       
      break;
    }
    if (w == -1) {
      if (errno == EINTR) {
        if (timeout_fired) {
          // Time's up: try TERM, then escalate to KILL if needed
          fprintf(stderr, "[Timeout] Killing pid %d after 10s\n", pid);
          (void)kill(pid, SIGTERM);

          // Wait up to ~1s for graceful exit
          int i;
          for (i = 0; i < 10; i++) {
            pid_t w2 = waitpid(pid, &status, WNOHANG);
            if (w2 == pid) {
              alarm(0); 
              goto done_waiting;
            }
            usleep(100000);
          }
          (void)kill(pid, SIGKILL);
          (void)waitpid(pid, &status, 0);
          alarm(0);
          goto done_waiting;
        }
        continue;
      } else {
        // Real waitpid error
        perror("waitpid");
        alarm(0);
        break;
      }
    }
  }
  done_waiting:
    if (WIFSIGNALED(status)) {
      fprintf(stderr, "Process terminated by signal %d\n", WTERMSIG(status));
    }
}



/*----------- MAIN -----------*/ 
int main() {
  // Stores the string typed into the command line.
  char command_line[MAX_COMMAND_LINE_LEN];
  char cmd_bak[MAX_COMMAND_LINE_LEN];
  
  // Stores the tokenized command line input.
  char *arguments[MAX_COMMAND_LINE_ARGS];
    
  // SIGINT handler
  struct sigaction sa;
  sigemptyset(&sa.sa_mask);
  sa.sa_handler = on_sigint;
  sa.sa_flags = SA_RESTART; 
  if (sigaction(SIGINT, &sa, NULL) < 0) {
    perror("sigaction");
    return 1;
  }

  // SIGALRM handler
  struct sigaction sa_alrm;
  sigemptyset(&sa_alrm.sa_mask);
  sa_alrm.sa_handler = on_sigalrm;
  sa_alrm.sa_flags = 0;              // do NOT use SA_RESTART here
  if (sigaction(SIGALRM, &sa_alrm, NULL) < 0) {
    perror("sigaction(SIGALRM)");
    return 1;
  }

  while (true) {
    /* prompt + fgets + tokenize + builtins */
    if (got_sigint) {
      got_sigint = 0;
    }
    print_prompt();

    // Read input from stdin, if error, exit immediately
    if ((fgets(command_line, MAX_COMMAND_LINE_LEN, stdin) == NULL) && ferror(stdin)) {
      fprintf(stderr, "fgets error");
      exit(0);
    }

    // while just ENTER pressed
    if (command_line[0] == '\n') continue;
    // command_line[strlen(command_line) - 1] = '\0';
  
    // Strip trailing newline
    size_t len = strlen(command_line);
    if (len > 0 && command_line[len - 1] == '\n') command_line[len - 1] = '\0';

    // If the user input was EOF (ctrl+d), exit the shell.
   if (feof(stdin)) {
      printf("\n");
      fflush(stdout);
      fflush(stderr);
      return 0;
    }	  
  
    // 1. Tokenize the command line input (split it on whitespace)
    int argc = tokenize(command_line, arguments, MAX_COMMAND_LINE_ARGS);
    if (argc == 0) continue;

    // Check for background process
    bool background = false;
    if (strcmp(arguments[argc - 1], "&") == 0) {
      background = true;
      arguments[argc - 1] = NULL;  // remove '&' from argument list
      argc--;
    }
  
    // 2. Implement Built-In Commands
    if (is_builtin(arguments[0])) {
      (void)run_builtin(argc, arguments);
      continue;
    }

    // 3. Create a child process which will execute the command line input
    run_external(argc, arguments, background);

    // 4. The parent process should wait for the child to complete unless its a background process
  
  
    // Hints (put these into Google):
    // man fork
    // man execvp
    // man wait
    // man strtok
    // man environ
    // man signals
    
    // Extra Credit
    // man dup2
    // man open
    // man pipes
  }
  // This should never be reached.
  return -1;
}
