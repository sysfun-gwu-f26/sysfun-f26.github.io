---
layout: default
title: Homework 2
subtitle: Multi-Week Unix Shell Project Specification (Weeks 1–3)
permalink: /hw2
---

## 2. Milestone 1 (Week 1): the data model, state, built-ins and input

**Goal.** Get comfortable with the toolchain and with C's types, control flow, structs and arrays — and,
at the same time, define the vocabulary of the whole shell. No dynamic memory yet: everything here uses
fixed-size arrays, which is why the limits (`MAX_ARGS`, `MAX_ARG_LEN`, ...) are constants in `command.h`.

**Skills practised:** integer types and bit masks, `char` arrays and the terminating `'\0'`, off-by-one
reasoning, `struct`s (including arrays inside structs), `enum`s, tables of structs, functions that report
errors through return values, `snprintf`-style truncation.

> **Read the header before each function.** The header comment is the contract: every limit, every
> "returns -1 if...", every "leaves the struct unchanged". The tests check the contract, not your intentions.

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
