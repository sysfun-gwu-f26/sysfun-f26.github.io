---
layout: default
title: Homework 2
subtitle: Multi-Week Unix Shell Project Specification (Weeks 1–3)
permalink: /hw2
---

## Overview

In this multi-week project, you will build a fully functional, custom Unix shell from the ground up. Rather than completing isolated, disposable exercises, every single line of code you write starting in Week 1 is a permanent component of your shell's core toolkit.

By approaching development iteratively, you will design the data structures, dynamic memory management utilities, and low-level system call helpers in Weeks 1–3 so that assembling the full, interactive shell in Weeks 4–6 becomes an exercise in modular integration rather than a high-pressure build from scratch.

<div class="callout arch-note">
  <div class="callout-title">Pedagogical Philosophy: The "Toolkit" Model</div>
  <ul>
    <li><strong>Zero Disposable Code:</strong> Every assignment deliverable produces a concrete header, implementation file, or scaffolding structure that directly plugs into your final shell executable.</li>
    <li><strong>Incremental Integration:</strong> Complex systems like Unix shells are best built by designing data structures early, thoroughly unit-testing subroutines in isolation, and composing them into larger subsystems.</li>
    <li><strong>Production Habits First:</strong> Professional toolchains, strict compiler warnings, leak-free memory management, and robust error handling are enforced from day one.</li>
  </ul>
</div>

---

## High-Level Execution Pipeline

The subroutines you build during Weeks 1–3 will power the core life cycle of every command executed by your shell:

<div class="pipeline">
  <div class="pipeline-stage">
    <div class="stage-num">Stage 1</div>
    <div class="stage-name">REPL Prompt</div>
  </div>
  <span class="pipeline-arrow">&rarr;</span>
  <div class="pipeline-stage">
    <div class="stage-num">Stage 2</div>
    <div class="stage-name">Tokenizer</div>
  </div>
  <span class="pipeline-arrow">&rarr;</span>
  <div class="pipeline-stage">
    <div class="stage-num">Stage 3</div>
    <div class="stage-name">Redirection</div>
  </div>
  <span class="pipeline-arrow">&rarr;</span>
  <div class="pipeline-stage">
    <div class="stage-num">Stage 4</div>
    <div class="stage-name">Fork &amp; Exec</div>
  </div>
  <span class="pipeline-arrow">&rarr;</span>
  <div class="pipeline-stage">
    <div class="stage-num">Stage 5</div>
    <div class="stage-name">Waitpid / Status</div>
  </div>
</div>

---

## Weekly Breakdown & Key Deliverables

### Week 1 — Project Infrastructure & Core Data Model
**Focus:** Toolchain Setup, Data Modeling, Constructors/Destructors, and REPL Scaffolding

Week 1 establishes the lifetime skeleton and foundational data abstractions of your shell.

#### Key Deliverables

1. **Permanent Build Infrastructure:** Set up the standard repository directory layout (`src/`, `include/`, `build/`) and an extensible Makefile with strict flags: `-Wall -Wextra -Werror -g`. This build system will be used through Week 6.
2. **Shell Data Model (`struct command` & `struct shell_state`):**
   * Design `struct command` to represent a single parsed command, including `argv` array, `argc`, input/output redirection file paths, and background execution flags (`is_background`).
   * Design `struct shell_state` to maintain long-term execution context (current working directory, last exit status, and placeholder hooks for background jobs).
   * Write and unit-test initialization and teardown subroutines (`command_create`, `command_free`, `shell_state_init`, `shell_state_free`).
3. **Interactive REPL Skeleton:** Implement the main shell execution loop in `src/main.c`. This loop prints a prompt, reads raw user input, echoes the input back, and cleanly exits when requested. *(Command execution logic will be wired in subsequent weeks.)*

---

### Week 2 — Memory Management, Parsing, & Dynamic Utilities
**Focus:** String Manipulation, Tokenization, Dynamic Arrays, Linked Lists, and Memory Safety

Week 2 produces the primary utility and parsing library that drives input processing and process tracking.

<div class="callout danger">
  <div class="callout-title">Valgrind Enforcement</div>
  All Week 2 deliverables must execute completely clean under <code>valgrind</code> (0 leaks, 0 illegal memory accesses) before submission. Memory safety discipline must be established before multi-process <code>fork()</code>/<code>exec()</code> development in Week 3.
</div>

#### Memory Layout Target: Argv String Array

When tokenizing an input line like `"ls -l /usr/bin"`, your dynamic string array memory allocation must look like this on the heap:

<div class="memory-grid">
  <div class="mem-header">Address</div>
  <div class="mem-header">Offset +0</div>
  <div class="mem-header">Offset +8</div>
  <div class="mem-header">Offset +16</div>
  <div class="mem-header">Offset +24</div>
  <div class="mem-header">Offset +32</div>
  <div class="mem-header">Offset +40</div>
  <div class="mem-header">Offset +48</div>
  <div class="mem-header">Offset +56</div>

  <div class="mem-addr">0x55ff0000</div>
  <div class="mem-cell highlight-heap">argv[0]</div>
  <div class="mem-cell highlight-heap">argv[1]</div>
  <div class="mem-cell highlight-heap">argv[2]</div>
  <div class="mem-cell highlight-stack">NULL</div>
  <div class="mem-cell">0x00</div>
  <div class="mem-cell">0x00</div>
  <div class="mem-cell">0x00</div>
  <div class="mem-cell">0x00</div>
</div>

#### Key Deliverables

1. **Command-Line Tokenizer (`src/parser.c`):**
   * Implement basic pointer drills (`my_strlen`, `my_strcpy`).
   * Implement the core parser function that takes a raw input line and splits it into a dynamically allocated `argv`-style array of strings, properly handling leading, trailing, and redundant whitespace, as well as quoted strings.
2. **Dynamic String Array (Resizing Vector):**
   * Implement a generic dynamic array of strings with automatic growth (`realloc`) and comprehensive deep-deallocation logic. This structure backs both the tokenizer's argument list and future features such as command history.
3. **Job Control List Stub (`src/job_list.c`):**
   * Implement a doubly linked list structure designed to track background process nodes (storing `pid_t`, job IDs, and command strings).
   * Write subroutines for `job_insert`, `job_remove_by_pid`, and `job_print_all`.
4. **Valgrind Memory Verification:** Run all test suites through Valgrind to ensure dynamic heap allocations are properly deallocated by `command_free()`.

---

### Week 3 — System Calls, Process Execution, & I/O Redirection
**Focus:** File Descriptors, Redirection, Process Creation, Pipelines, and Signal Handling

Week 3 builds the core system execution subroutines that translate user commands into real operating system processes.

<div class="code-compare">
  <div class="code-col">
    <h4>C System Call Interface</h4>
```c
/* Duplicate file descriptor */
int fd = open("out.txt", O_WRONLY|O_CREAT|O_TRUNC, 0644);
dup2(fd, STDOUT_FILENO);
close(fd);
execvp(cmd->argv[0], cmd->argv);
