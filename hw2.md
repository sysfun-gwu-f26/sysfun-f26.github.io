---
layout: default
title: Homework 2
subtitle: Creating a UNIX Shell (Weeks 1–3)
permalink: /hw2
---

## Milestone 1 (Week 1): the data model, state, built-ins and input

The goal of this week is to get comfortable with the toolchain and with C's types, control flow, structs and arrays — and,
at the same time, define the vocabulary of the whole shell. No dynamic memory yet: everything here uses
fixed-size arrays, which is why the limits (`MAX_ARGS`, `MAX_ARG_LEN`, ...) are constants in `command.h`.

The concepts you will be implementing this week include: integer types and bit masks, `char` arrays and the terminating `'\0'`, off-by-one
reasoning, `struct`s (including arrays inside structs), `enum`s, tables of structs, functions that report
errors through return values, `snprintf`-style truncation.

> **Read the header before each function.** The header before each function lives in the .h of the file.
> Here you will find the documentation stating what each function will return and its behavior.
> Read these headers to understand the purpose of each function before implementation. 

### 2.1 `command.c` — what is a command? (7 functions)

A line such as `cat < in.txt | sort > out.txt &` becomes one `struct pipeline` holding two
`struct command`s. A `struct command` stores its words (`argv`), its optional input file, its optional
output file, and whether the output is appended.

| Function | What it must do | Watch out for |
|---|---|---|
| `command_init` | set *every* field to "empty" | `argv[i]` must be `""` for **all** `i`, not just the used ones. The test fills the struct with garbage first. |
| `command_add_arg` | append a copy of a word | The array holds `MAX_ARG_LEN + 1` chars: room for the `'\0'`. A word of exactly `MAX_ARG_LEN` characters **fits**; one more does not. A failed call must not change the command at all. `""` is a legal word. |
| `command_set_infile` | set the `<` file | reject `NULL`, `""` and too-long paths, leaving the old value alone |
| `command_set_outfile` | set the `>` / `>>` file | store `append` as exactly 0 or 1. A *failed* call must not change `append` either. |
| `command_argv_ptrs` | build the `char *[]` that `execvp` wants | `out` has `MAX_ARGS + 1` slots precisely so that `out[argc]` can be `NULL`, even for a full command. The pointers point **into** the struct (nothing is copied). |
| `command_format` | render `"cat < a > b"` as text | see below |
| `pipeline_init` | empty pipeline | initialise all `MAX_CMDS` commands |

**`command_format` and `snprintf` semantics.** Many C functions that fill a buffer follow the contract of
`snprintf`: *never write past `size`; always terminate with `'\0'` when `size > 0`; return the length the
output **would** have had if the buffer were big enough.* Callers use the return value to detect truncation
or to allocate exactly the right amount. This function, `shell_state_format_prompt` and `joblist_format` all
use it. Example, for the command `hello world` (11 characters):

| call | buffer afterwards | returns |
|---|---|---|
| `command_format(&c, buf, 32)` | `hello world` | 11 |
| `command_format(&c, buf, 12)` | `hello world` | 11 |
| `command_format(&c, buf, 11)` | `hello worl` | 11 |
| `command_format(&c, buf, 5)` | `hell` | 11 |
| `command_format(&c, NULL, 0)` | (nothing written) | 11 |

*Hint:* write one small helper that appends a string at an offset, always advancing the offset by the full
length of the string but copying only what still fits. Then every piece (word, `<`, file name, ...) goes
through that helper and the truncation logic exists in exactly one place.

### 2.2 `shell_state.c` — status and prompt (3 functions)

A process's exit status is a number from 0 to 255 (only the low 8 bits survive).
`shell_state_set_status` must reduce any `int` to that range: `256 → 0`, `257 → 1`, **`-1 → 255`**.
Think about which operator gives the right answer for negative numbers (the test checks `-1`).
The prompt is `mysh> ` normally and `[127] mysh> ` after a command that exited with status 127.

### 2.3 `builtin.c` — recognising built-ins (2 functions)

`cd`, `exit` and friends cannot be run by a forked child (a child that changes *its* directory changes nothing
for the shell), so the shell recognises them itself. For now you only identify them.
Use a **static table** of `struct { const char *name; enum builtin_id id; }` and loop over it, instead
of a chain of `if`s. Matching is exact: `"exit "` and `"Exit"` and `"ex"` are not built-ins.

### 2.4 `input.c` — reading a line (2 functions)

`read_line` is the part of Week 1 with the most edge cases. All of these are tested:

| Input (buffer size 8) | Result |
|---|---|
| `abc\n` | returns 3, buffer `abc` (no newline stored) |
| `\n` | returns 0 |
| `1234567\n` (7 chars) | returns 7: it **fits** (7 chars + `'\0'` = 8) |
| `12345678\n` (8 chars) | `READ_TOO_LONG`, **and the rest of that line is thrown away**, so the next call returns the next line |
| `last` then end of file (no `\n`) | returns 4; the following call returns `READ_EOF` |
| nothing at all | `READ_EOF` |

*Hint:* `fgets` does most of the work. If the buffer does not end with `'\n'` you must find out why: the line
exactly filled the buffer? it was the last line of the file? or it was too long? Peeking at the next character
with `fgetc` tells you which.
`str_trim` removes leading and trailing whitespace **in place** and returns a pointer to the first
non-blank character. (`isspace` wants an `unsigned char`: cast.)

### 2.5 `main.c` — the loop

Write the shell's main loop. Behaviour (checked by `tests/integration_m1.sh`):

1. Format the prompt with `shell_state_format_prompt`, print it, and **`fflush(stdout)`** (the prompt has no
   newline, so it would otherwise stay in the buffer while the shell waits for input).
2. `read_line` into a 1024-byte buffer. On `READ_EOF` (Ctrl-D) print a single `\n` and quit. On
   `READ_TOO_LONG` print `mysh: line too long` to **stderr**, set the status to 1, and loop again.
3. `str_trim` the line; a blank line just prompts again.
4. If the line is the built-in `exit`, clear `st.running` and stop.
5. Otherwise print `You said: <line>` and set the status to 0.

```
$ make build/mysh_m1 && ./build/mysh_m1
mysh> hello world
You said: hello world
mysh>
mysh>    exit
$
```

In Week 4 step 5 becomes "parse and run the command"; everything else in this loop stays.

### 2.6 Checking Milestone 1

```
make test1      # expect: 36/36 unit tests and 7/7 integration tests
make asan1      # must also be clean
```

## 3. Milestone 2 (Week 2): strings, dynamic memory, tokenizer, parser, jobs

**Goal.** Build the front end of the shell (the code that turns the text you typed into a `struct pipeline`)
and the data structures behind it. This is the week of **pointers and ownership**: for every piece of
memory you create, you must be able to say who owns it and who frees it.

**Skills practised:** pointer arithmetic, `malloc`/`realloc`/`free`, ownership and lifetime, `NULL` checks,
pointer-to-pointer, linked lists, a hand-written scanner, grammar-driven validation.

> **Milestone 1 must pass first.** The parser calls `command_add_arg`, `command_set_infile` and friends.

### 3.1 `mystring.c` — seven `<string.h>` functions

Each must behave **exactly** like its libc namesake; the tests compare against libc. You may not include
`<string.h>` or call any libc string/memory function in this file (`malloc` is allowed). Read the
manual pages (`man 3 strncpy`) — some of these have surprising rules.

| Function | The subtle part |
|---|---|
| `my_strlen` | try writing it with a pointer that walks to the `'\0'`, then subtracting pointers |
| `my_strcmp`, `my_strncmp` | compare as **`unsigned char`**: `"\xff"` is *greater* than `"a"`. Only the sign of the result matters. `n == 0` compares nothing. |
| `my_strcpy` | returns `dst`; copies the `'\0'` |
| `my_strncpy` | writes **exactly `n` bytes**: pads with `'\0'` if `src` is shorter, and does **not** terminate if `src` has `n` or more characters |
| `my_strchr` | the argument is an `int` converted to `char`; searching for `'\0'` finds the terminator |
| `my_strdup` | `malloc(length + 1)`; return `NULL` for a `NULL` argument **or** a failed `malloc`; check before copying |

### 3.2 `strvec.c` — a growable array of strings

```
struct strvec            items ──► ┌────┬────┬────┬────┐   capacity (cap) = 4 slots
  len = 3                          │ ●  │ ●  │ ●  │    │   length (len)   = 3 in use
  cap = 4                          └─┬──┴─┬──┴─┬──┴────┘
                                     ▼    ▼    ▼
                                   "ls" "-l" "/tmp"      <- each string is its own malloc'ed copy
```

* **Ownership.** The vector owns its strings. `strvec_push` stores a *copy* (so the caller may change or
  free the original). `strvec_get` lends you a pointer: do not free it. `strvec_pop` is the one function
  that **transfers ownership to you**: you must `free` what it returns.
* **Growth rule** (tested exactly): the first push allocates room for **4**; whenever the array is full the
  capacity **doubles** (4, 8, 16, 32, ...). Doubling makes `n` pushes cost O(n) total; growing by one slot
  each time costs O(n²).
* **The `realloc` trap.** `v->items = realloc(v->items, n)` is a bug: if `realloc` fails it returns `NULL`
  and you have just overwritten the only pointer to the old block (a leak, and the vector is destroyed).
  Assign to a temporary, check it, and only then update `v->items`.
* **Failure must be clean.** The tests make `malloc` fail on purpose at every possible point. After a
  failed push the vector must contain exactly what it did before, and nothing may leak (if you made the
  copy and then the array allocation fails, free the copy).
* `strvec_clear` frees the strings but keeps the array; `strvec_free` frees everything and resets the vector to
  the `strvec_init` state, so calling it twice is harmless.

### 3.3 `lexer.c` — `tokenize`

`tokenize("ls -l | wc > out", &v)` fills `v` with `ls`, `-l`, `|`, `wc`, `>`, `out`. The complete rules are
in `include/lexer.h`; the essentials:

| Line | Tokens |
|---|---|
| `ls   -l\t/tmp` | `ls` `-l` `/tmp` (any whitespace separates; repeated whitespace makes no empty tokens) |
| `ls>out`  `a\|b`  `sleep 5&` | operators `\| < > >> &` are **always separate tokens**, even without spaces |
| `a>>b`  `a>>>b` | `a` `>>` `b`  and  `a` `>>` `>` `b` |
| `echo "hello world"` | `echo` `hello world` (quotes group text and are removed) |
| `he"llo wo"rld` | `hello world` (one token: text touching a quoted part stays in the same token) |
| `echo ""` | `echo` and an **empty token** `""` |
| `"it's"`  `'say "hi"'` | `it's`  and  `say "hi"` (the other kind of quote is plain text) |
| `echo "a\|b"` | `echo` `a\|b` (operators inside quotes are plain text) |
| `echo "oops` | error `LEX_ERR_QUOTE`, and `out` is left **empty** |

*Design hint.* Walk the line with an index. Allocate **one** scratch buffer as long as the whole line plus
one (no token can be longer than the line) and build each token in it. At each step you are in one of three
cases: whitespace (skip), operator (emit it), or the start of a word (keep going until unquoted whitespace
or an operator, copying quoted text without its quote characters). Every exit path must `free` the scratch
buffer, including the error paths; the tests check that.

> *Known limitation (think about it for your reflection):* the result is only strings, so a quoted `">"`
> cannot later be told apart from a real redirection operator. How would you fix that in the data model?

### 3.4 `parser.c` — `parse_pipeline`

Tokens in, `struct pipeline` out. The grammar (full rules in `include/parser.h`):

```
line     := (empty) | pipeline [ "&" ]
pipeline := command ( "|" command )*
command  := ( WORD | redirect )+                 at least one WORD
redirect := ( "<" | ">" | ">>" ) WORD
```

| Tokens | Result |
|---|---|
| (none) | `PARSE_EMPTY` |
| `ls -l` | 1 command, `argv = {ls, -l}` |
| `cat < in > out` | 1 command, `infile = in`, `outfile = out` |
| `> out ls -l` | `argv = {ls, -l}`, `outfile = out` (redirections may appear anywhere) |
| `cat < in \| sort \| uniq > out` | 3 commands; `in` on the first, `out` on the last |
| `ls \| wc &` | 2 commands, `background = 1` |
| `ls \|`   `\| ls`   `ls \| \| wc` | `PARSE_ERR_SYNTAX` (every command needs a word) |
| `ls > a \| wc` | `PARSE_ERR_SYNTAX` (`>` only on the **last** command; `<` only on the **first**) |
| `cat < a < b` | `PARSE_ERR_SYNTAX` (at most one of each per command) |
| `ls >`   `ls > \| wc`   `cat > ""` | `PARSE_ERR_SYNTAX` (a redirection needs a real file name) |
| `ls & ls`   `&` | `PARSE_ERR_SYNTAX` (`&` only at the very end, after a complete pipeline) |
| 17 words | `PARSE_ERR_TOO_MANY_ARGS` (`MAX_ARGS` is 16) |
| 9 piped commands | `PARSE_ERR_TOO_MANY_CMDS` (`MAX_CMDS` is 8) |
| a 65-character word | `PARSE_ERR_TOO_LONG` |

Use `my_strcmp` (never `==`) to compare token strings, call `pipeline_init(out)` first so that old data
can never leak through, and reuse your Milestone 1 functions to store words and file names. *Hint:* keep a
pointer `cur` to the command you are filling; `|` validates it and moves `cur` forward.

### 3.5 `joblist.c` — background jobs as a linked list

Nothing runs in the background yet. In Week 5 every `&` command will add a node here, and every
finished process will remove one. The list is ordered oldest-first, ids count up from 1 and are
**never reused**, and `joblist_format` produces what the `jobs` built-in will print:

```
[1] 4021 sleep 10
[2] 4030 ls | wc
```

*Hint.* A `struct job **link` that points at either `jl->head` or some node's `next` field lets one loop
handle the first node, a middle node and the last node with the same code — no special case for the head.
In `joblist_add` this walks you to the end of the list; in `joblist_remove_pid` it lets you unlink a node with
`*link = node->next`. If the node allocation succeeds but copying the command text fails, free the node. A
failed `add` must not consume an id.

### 3.6 Memory discipline (read this before you debug)

* **Every `malloc` has an owner and a matching `free`.** Write the owner in a comment next to the allocation.
* **Check every allocation.** The tests simulate "out of memory" at *every* allocation your function makes,
  one at a time; the function must report failure, leave its data structure valid, and leak nothing.
* **`realloc` into a temporary.** (See 3.2.)
* **Free in the right order**: read `node->next` *before* you free `node`.
* **After `free`, the pointer is dangling.** Do not use it; set it to `NULL` if it lives on.

Run `make asan2`. AddressSanitizer stops at the first memory error and prints a report like

```
ERROR: AddressSanitizer: heap-buffer-overflow on address 0x602000000014 ...
WRITE of size 1 at ... #0 my_strcpy  src/mystring.c:42   <-- the line that did it
... located 0 bytes after 4-byte region ...                  <-- allocated 4, wrote the 5th: forgot the '\0'
allocated by ... #1 my_strdup src/mystring.c:61              <-- where the block came from
```

Read it bottom-up: *who allocated*, then *who misused it*. At exit it also lists leaks. `make valgrind2` gives
similar information (slower, no recompilation).

### 3.7 Checking Milestone 2

```
make test2      # expect 72/72
make asan2      # must also be clean (leaks count as failures)
make lint       # mystring.c / lexer.c restrictions

## 4. Milestone 3 (Week 3): file descriptors, processes, pipes, signals

**Goal.** Meet the operating system. By the end of this week you can start a program, wire its input and
output to files or to another program, and get its exit status back — the entire engine of a shell. These are
the functions you will call from your main loop in Week 4.

**Skills practised:** file descriptors, `open`/`read`/`write`/`close`, `dup2`, `fork`, `execvp`, `waitpid`,
`pipe`, exit statuses, signals (`sigaction`), `errno`, and careful resource cleanup.

### 4.1 The mental model

* A **file descriptor** is a small integer naming an open file, pipe or terminal in the process's
  descriptor table. Every process starts with `0` = stdin, `1` = stdout, `2` = stderr.
* `open` returns the **lowest unused** number. `close(fd)` frees it. `dup2(a, b)` makes `b` another name for
  whatever `a` refers to (closing `b` first). **This is how redirection works**: open the file, `dup2` it onto
  1, `close` the original.
* `fork` creates a copy of the process, *including its descriptor table and its unflushed stdio buffers*.
* `exec` replaces the program but **keeps** open descriptors and **ignored signals**. Anything you do not
  clean up before `exec` is inherited by the new program.
* A pipe's read end reports end-of-file only when **every** copy of its write end, in **every** process, has
  been closed. One forgotten copy and the reader waits forever.

### 4.2 `iohelp.c` — two helpers built on `read` and `write`

No stdio in this file (`make lint` enforces it). The two system calls have behaviour that stdio normally hides
from you:

* `write(fd, buf, n)` may write **fewer than `n` bytes** (a *short write*), e.g. when a pipe fills up and a
  signal arrives. `write_all` must keep going until everything is written.
* `read` and `write` may fail with `errno == EINTR` when a signal handler interrupts them. That is **not** an
  error: try again. Any other failure: return `-1` (and do not touch `errno` afterwards: the caller reads it).

`copy_fd(in, out)` copies until end-of-file through a fixed buffer, using `write_all`, and returns the number of
bytes copied (or `-1`). It must **not** close either descriptor, and a short `read` is *not* the end of the
file: only `read` returning `0` is.

The tests exercise all of this with real signals and real pipes: a writer that is interrupted mid-write, a
slow producer that sends data in two bursts, a signal that arrives while `read` is blocked, a copy through a
pipe larger than the pipe buffer.

### 4.3 `executor.c` — running commands

**Exit-status convention** (identical to a real shell, and used by every function below):

| What happened | Status |
|---|---|
| the program exited with code `c` | `c` |
| the program was killed by signal `S` | `128 + S` (so Ctrl-C = `SIGINT` = 2 → **130**; `SIGTERM` → 143) |
| the program was not found (`execvp` failed with `ENOENT`) | **127** |
| found but could not be executed (e.g. no execute permission) | **126** |
| a redirection could not be set up | **1** |

Use the macros `WIFEXITED`, `WEXITSTATUS`, `WIFSIGNALED`, `WTERMSIG` on the status from `waitpid`.

**Signals: `shell_install_signal_handlers` / `shell_reset_child_signals`.** When you press Ctrl-C the
terminal sends `SIGINT` to *every* process in the foreground, including the shell. The shell must survive
(ignore it); the program it started must die (default action). The trap: "ignore" is inherited across `exec`,
so each child has to put the default action back **before** `exec`, or `sleep 100` could never be interrupted.
Use `sigaction` (not `signal`) with `SIG_IGN` / `SIG_DFL` for `SIGINT`, `SIGQUIT` and `SIGTSTP`.

**`redirect_io(cmd)`** applies the `<`, `>`, `>>` redirections in the *current* process (it is called in a
child just before `exec`). Requirements:

1. open the input file read-only; **if that fails, return -1 without touching the output file** (it must not
   be created or truncated);
2. open the output file write-only with `O_CREAT`, and `O_APPEND` or `O_TRUNC`, **mode `0644`**;
   if that fails, close the input descriptor you opened and return -1 (keeping `errno` from the failed `open`);
3. `dup2` the descriptors onto 0 and 1, then close the originals so **no extra descriptors remain**;
4. **special case:** if `open` happens to return 0 or 1 itself (because the program had closed stdin/stdout),
   `dup2(fd, fd)` does nothing, and calling `close(fd)` afterwards would close your new file. The tests do this
   on purpose.

On failure the function is **all-or-nothing**: descriptors 0 and 1 are unchanged and nothing is leaked.

**`execute_command(cmd, st)`**

```
parent                                   child
  fork() ─────────────────────────────►  shell_reset_child_signals()
  waitpid(child)  (retry on EINTR)       redirect_io(cmd)        failed → message, _exit(1)
  decode status → *st, return it         execvp(argv[0], argv)   failed → message, _exit(127 or 126)
```

* Failure messages go to **stderr**: `mysh: redirection failed: <reason>` and `mysh: <program>: <reason>`
  (`strerror(errno)` gives the reason; save `errno` before calling anything else).
* **Children must call `_exit`, never `exit`.** `exit` flushes stdio buffers; the child owns a *copy* of the
  parent's unflushed buffers, so the data would be written twice. The tests detect this.
* The parent must not change its own stdin/stdout, must not leak descriptors, and must not leave zombies.
* An empty command (`argc == 0`) or `NULL` returns `-1` **without forking**.

**`execute_pipe2(left, right, st)`** runs `left | right`:

```
parent (mysh)                  left child                         right child
 pipe(fds)
 fork ────────────────────────► dup2(fds[1], 1)
                                close(fds[0]); close(fds[1])
                                redirect_io(left); execvp
 fork ──────────────────────────────────────────────────────────► dup2(fds[0], 0)
                                                                   close(fds[0]); close(fds[1])
                                                                   redirect_io(right); execvp
 close(fds[0]); close(fds[1])
 waitpid(left); waitpid(right)  ──►  return the status of the RIGHT command
```

Both commands must run **at the same time**. If you wait for the left one to finish before starting the
right one, the left one blocks as soon as the pipe is full (64 KiB) and you deadlock; one of the tests pushes
300 000 bytes through to catch that.

**Symptoms and likely causes**

| Symptom | Probable cause |
|---|---|
| `TIMEOUT` in a pipe test, a program like `cat` or `wc` never finishes | a copy of the pipe's write end is still open: in the parent, or in the right child (close **both** ends everywhere) |
| output appears twice | a child called `exit` instead of `_exit`, or forked with unflushed stdio output |
| `test_execute_leaves_no_zombies` fails | you did not `waitpid` for every child |
| Ctrl-C does not stop a program (`...child_gets_default_sigint...` fails) | the child did not restore `SIG_DFL` before `exec` |
| the test that closes stdin or stdout fails | `dup2(fd, fd)` followed by `close(fd)` |
| "lowest free descriptor" check fails | a descriptor leaked on some path, probably an *error* path |
| exit status is always 0 or 255 | you returned the raw `waitpid` status instead of decoding it |

### 4.4 Debugging tools for processes

* `strace -f -e trace=process,desc ./build/test_m3 2>&1 | less` shows every `fork`, `dup2`, `close`, `execve`
  and `wait4` of **all** processes (`-f` follows children). Reading a pipe test's trace is the fastest way to
  find an unclosed descriptor.
* `ls -l /proc/<pid>/fd` lists a process's open descriptors (run a program like `sleep 100` and inspect it
  from another terminal).
* In the VS Code debugger the entries from Section 1.4 already follow the child process. In a plain terminal
  `gdb`, type `set follow-fork-mode child` before `run` to debug the child instead of the parent.
* Print debugging output to **stderr** (`fprintf(stderr, ...)`): it is unbuffered, so nothing is lost when a
  process is killed.

### 4.5 Checking Milestone 3

```
make test3      # expect 52/52
make asan3
make valgrind3  # two tests are skipped under valgrind (it explains why when it runs)
```
