# Creating a UNIX Shell — The Foundations

**Milestones 1–3 (Weeks 1–3) · C programming and the Unix system-call interface**

In Weeks 4–6 you will assemble a working Unix shell, [mysh](mysh.zip), and extend it with your own OS features.
Before that you need the parts. This assignment is those parts: **three milestones, one per week, each
a set of small modules with a self-checking test suite.** Nothing you write here is throw-away. Every
function is something the final shell calls, so the quality of your Week 1–3 code decides how smooth
Weeks 4–6 will be.

```
  what you type                                             what the OS does
       │
       ▼
  ┌──────────┐   ┌───────────┐   ┌───────────┐   ┌──────────────────────────────┐
  │ input.c  │──►│ lexer.c   │──►│ parser.c  │──►│ executor.c                   │
  │ read_line│   │ tokenize  │   │ parse_    │   │ fork · exec · wait · pipe ·  │──► programs run
  │ str_trim │   │           │   │ pipeline  │   │ dup2 · signals               │
  └──────────┘   └─────┬─────┘   └─────┬─────┘   └──────────────┬───────────────┘
                       │               │                        │
   Milestone 2 ────────┤  strvec.c     │  builds                │ uses
   (mystring.c, joblist.c)  token list │  command.c  ◄──────────┘ iohelp.c
                                       ▼  (Milestone 1: the data model)
                                 struct pipeline { struct command cmds[], background }

   shell_state.c (exit status, prompt)      builtin.c (cd, exit, ...)      joblist.c (background jobs)
```

| Milestone | Week | You write | Main ideas | Tests |
|---|---|---|---|---|
| **1** | 1 | `command.c` `shell_state.c` `builtin.c` `input.c` `main.c` | types, control flow, structs, arrays, strings, `snprintf` | `make test1` — 36 unit + 7 integration |
| **2** | 2 | `mystring.c` `strvec.c` `lexer.c` `parser.c` `joblist.c` | pointers, `malloc`/`realloc`/`free`, ownership, linked lists, parsing | `make test2` — 72 tests |
| **3** | 3 | `iohelp.c` `executor.c` | file descriptors, `read`/`write`, `fork`/`exec`/`wait`, `pipe`/`dup2`, signals | `make test3` — 52 tests |

Each milestone has its own due date (end of its week). Milestone 2 depends on Milestone 1 (the parser is built
from your `command_*` functions) and Milestone 3 depends on it too, so **do not fall behind**: a bug in Week 1
code will surface again in Weeks 2 and 3.

---

## 0. Ground rules

**Environment.** You work inside a **development container**: a small Ubuntu Linux system (with `gcc`, `make`,
`gdb` and `valgrind` already installed) that runs in Docker on your own computer, while you edit the files in
VS Code as usual. Everyone gets the identical toolchain, so "it works on my machine" problems disappear, and
it works on Windows, macOS (Intel and Apple Silicon) and Linux. Setup is in Section 1.1. The tests rely on
Linux features (they make `malloc` fail on purpose with a GNU linker option), so do not try to run them
natively on macOS.

**Compiler flags.** Everything is compiled with `-std=gnu11 -Wall -Wextra -Werror`. **Warnings are
errors.** Treat each one as a bug report from the compiler; don't silence it with a cast until you
understand it.

**What you may edit.** Only the files in `src/`. Do not edit `include/` (those headers are the
specification), `tests/`, or the `Makefile`. If you think a header or test is wrong, tell the instructor.

**Restrictions per file** (checked by `make lint`). These exist so the exercise teaches what it is meant to:

| File | You may NOT | Why |
|---|---|---|
| `mystring.c` | include `<string.h>`; call any libc string/memory function | you are writing them |
| `lexer.c` | use `strtok`, `strsep`, `sscanf` | do the splitting yourself |
| `iohelp.c` | use stdio (`<stdio.h>`, `printf`, `fgets`, `FILE *` ...) | use `read` and `write` directly |
| `executor.c` | use `system()` or `popen()` | do the `fork`/`exec` yourself |

The lint check is a plain text search, so don't write those names followed by `(` even inside a comment.
Everywhere else the normal C library is fine.

**Your own work.** The tests are there so *you* can check your work. Write your own code, and
you may discuss ideas and debugging strategies with classmates, but do not share or copy code. At grading
time we may run extra tests, run your code under `valgrind`, and read it. Code that passes the visible
tests but handles nothing else correctly will lose points.

---

## 1. Getting started

### 1.1 One-time setup: VS Code and a development container

You need three free programs:

1. **Docker Desktop** (Windows, macOS) or Docker Engine (Linux): <https://www.docker.com/products/docker-desktop/>.
   On Windows, use the default *WSL 2* backend. Start Docker Desktop and leave it running.
2. **Visual Studio Code**: <https://code.visualstudio.com/>.
3. The VS Code extension **Dev Containers** (publisher: Microsoft; search for `ms-vscode-remote.remote-containers`
   in the Extensions view).

Then:

1. Unzip the assignment. **Windows:** put the folder somewhere inside your normal user folder or, for much
   faster builds, inside your WSL home folder; avoid network drives and OneDrive-synced folders.
2. In VS Code choose **File > Open Folder...** and open the `mysh` folder (the one that contains `Makefile`).
3. VS Code shows *"Folder contains a Dev Container configuration file. Reopen in Container"*. Click it. (If you
   miss the pop-up: press `F1` or `Ctrl+Shift+P`, type `Dev Containers: Reopen in Container`, press Enter.)
4. The first time, Docker downloads Ubuntu and installs the tools. This takes a few minutes and needs internet.
   Later starts take seconds. Wait until the bottom-left corner of VS Code says **Dev Container: mysh**.
5. Open a terminal inside VS Code (**Terminal > New Terminal**). The prompt should look like
   `ubuntu@...:/workspaces/mysh$`. Check that it works:

   ```
   make test1
   ```

   The unfinished starter code fails most tests, with a summary line like `-- 2/36 tests passed --` and no
   crashes. That is the correct starting point.

Things worth knowing:

* **Your files stay on your computer.** The `mysh` folder is shared into the container, so everything you
  edit in VS Code is saved in the real folder, survives if you delete the container, and is what you submit.
  Everything *outside* that folder inside the container (for example, anything you install with `apt`) is
  thrown away when the container is rebuilt.
* The terminal, build, tests, debugger and the IntelliSense squiggles all run inside the container. Do not
  run `make` in a terminal outside the container.
* You are the ordinary user `ubuntu`, not root. This is on purpose: one test checks that a file without
  execute permission cannot be run, and root is allowed to run it anyway.
* If Docker is not running you will get an error such as *"Cannot connect to the Docker daemon"*: start
  Docker Desktop and try again. After the instructor updates the course files, use
  **Dev Containers: Rebuild Container**.
* **Not allowed to install Docker** (some lab or school computers)? Ask the instructor about the course
  server, or use the free **GitHub Codespaces** (open the repository and choose *Code > Codespaces*), which
  uses this same configuration.

### 1.2 What is in the folder

```
mysh/
├── include/      the headers: THE SPECIFICATION of every function (read them first, don't edit)
├── src/          your code: one .c file per module, with TODO comments (edit these)
├── tests/        the test suites (don't edit)
├── .devcontainer/ the container recipe (don't edit)
├── .vscode/      VS Code tasks and debugger settings (don't edit)
├── Makefile
└── ASSIGNMENT.md this file
```

### 1.3 Commands

Run these in the VS Code terminal (you start in the right folder). Instead of typing, you can also press
`Ctrl+Shift+P`, choose **Tasks: Run Task**, and pick `test1`, `test2`, `test3`, `test (all)`, `asan1`, ...
Errors from the compiler then show up in the *Problems* panel and click through to the line.

**Commands** (run from the `mysh/` folder):

| Command | What it does |
|---|---|
| `make test1` / `make test2` / `make test3` | build and run that milestone's tests |
| `make test` | `lint` + all three milestones |
| `make lint` | check the per-file restrictions above |
| `make asan1` / `asan2` / `asan3` | the same tests built with AddressSanitizer + UBSan (finds buffer overflows, use-after-free, leaks) |
| `make valgrind1` / `valgrind2` / `valgrind3` | the same tests under valgrind (slow, but thorough) |
| `make` | build the shell itself, `build/mysh` (from Milestone 4 on) |
| `make build/mysh_m1` | build the Milestone 1 shell, run it with `./build/mysh_m1` |
| `make clean` | delete the build folders |

**Reading the test output.**

```
  [PASS] test_add_arg_basic
  [FAIL] test_add_arg_length_limit
         tests/test_m1.c:83: command_add_arg(&c, longest)
             expected 0, got -1
```

Each test runs in its own process, so a crash or an endless loop in one function does not hide the others:
you will see `CRASHED: Segmentation fault` or `TIMEOUT` next to that test instead. `TIMEOUT` after 10 seconds
means your code is stuck. To run only some tests: `TEST=strvec ./build/test_m2` runs every test whose name
contains `strvec`. Test names are descriptive on purpose: the name usually tells you what rule you broke.

**How to attack a milestone.** For each function: (1) read its contract in the header, (2) write it,
(3) run the tests, (4) read the *first* failure, fix it, repeat. Fix failures in the order they are
printed. The tests are ordered so that basic behaviour comes before edge cases. When `make testN` shows all
tests passing, run `make asanN`; **a milestone is not finished until it is also clean under the sanitizer.**

**Use version control.** `git init`, commit when a module passes its tests. When you break something in
Week 5, `git diff` will thank you.

### 1.4 Debugging in VS Code

Printing with `fprintf(stderr, ...)` always works, but the debugger saves time on pointer and memory bugs.

1. Open **Run and Debug** (`Ctrl+Shift+D`) and pick **Debug one test: milestone N** (N = 1, 2 or 3).
2. Click in the left margin of a source line to set a breakpoint (a red dot), for example the first line of the
   function that fails.
3. Press `F5`. VS Code asks which test to run: type part of its name, such as `lexer_quotes` (this is the same
   `TEST=` filter described above). Leave it empty to run all tests.
4. Use `F10` (step over), `F11` (step into), `Shift+F11` (step out), and look at the *Variables* panel and the
   *Watch* box. The debug console also accepts gdb commands after `-exec`, e.g. `-exec print *v`.

The test programs run every test in a child process, and the debugger is set to follow it, so your breakpoints
are hit inside the test as expected. Debugging runs without the 10-second timeout. The launch entry
**Debug the Milestone 1 shell** runs the interactive shell under the debugger.

If the debugger says `ptrace: Operation not permitted`, the container was not created from the course
configuration: use **Dev Containers: Rebuild Container**.
