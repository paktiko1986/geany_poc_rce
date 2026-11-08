# Geany Remote Code Execution (RCE) Proof of Concept

This repository contains a Proof of Concept (PoC) for a **Remote Code Execution (RCE)** vulnerability in the Geany text editor (tested on version 2.1).

## Vulnerability Details

*   **Vulnerability Type:** OS Command Injection (CWE-78) leading to Remote Code Execution (RCE).
*   **Affected Software:** Geany (Tested on version 2.1, likely affects all prior versions).
*   **Impact:** An attacker can craft a filename containing shell metacharacters (e.g., `;`). When a user opens the file and triggers a build or execute command (F5/F9), arbitrary commands are executed with the user's privileges.

## Root Cause Analysis

The vulnerability stems from a data flow flaw where a user-controlled filename is passed to a shell without proper sanitization:

1.  **Unsanitized Substitution (`src/utils.c`)**
    In the function `generate_document_replacements()` (around line 2562), Geany extracts the raw filename using functions like `g_path_get_basename()` and substitutes it into the build command string for placeholders like `%f` and `%e`. It does **not** apply `g_shell_quote()` to escape shell metacharacters.

2.  **Direct Shell Execution (`src/build.c`)**
    The substituted, unescaped command string is then passed to the system shell:
    *   **For Build (F9):** In `build_spawn_cmd()` (around line 716), Geany explicitly sets up an argument vector `{ "/bin/sh", "-c", cmd, NULL }` and passes the raw command string.
    *   **For Execute (F5):** In `build_create_shellscript()` (around line 1105), Geany generates a temporary shell script. While the working directory is properly escaped using `g_shell_quote(working_dir)`, the command string (`cmd`) is embedded raw into the script.

Because `/bin/sh` interprets characters like `;`, an attacker can break out of the intended command and execute arbitrary code.

## Reproduction Steps

**Prerequisites:** A Linux system with Geany installed. Ensure the default "Execute" command is used (e.g., `./%e` without enclosing quotes).

1.  **Create a Malicious File:**
    Open a terminal and create a file with a name designed to execute the GNOME Calculator:
    `touch "test; gnome-calculator; echo.c"`

2.  **Open in Geany:**
    Launch Geany and open the file you just created.
    
4. **Trigger Execution:**
Ensure the default "**Execute**" command is set (e.g., `./%e` without any enclosing quotes). Press F5.

5. **Observe the Result:**
The shell interprets the semicolon (`;`) as a command separator. Geany executes `./test` (which fails), and then immediately executes the injected command gnome-calculator. The calculator opens, proving arbitrary code execution using the default configuration.

## Expected Fix
To mitigate this vulnerability, Geany should apply proper shell escaping (e.g., using `g_shell_quote()`) to the substituted filename values (like `%f` and `%e`) inside `generate_document_replacements()` before passing them to the shell. Alternatively, avoid invoking a shell entirely by passing the command directly to `execve()` with a proper argument vector.

## References
*   Geany Issue #4611: Filename-Based Shell Injection via Run Script (Discusses the same root cause).
