************
Lab 5: Crash
************

Summary
-------

In this lab, you will build a small shell called ``crash`` — a much simpler version of what you interact with when you ssh into a Linux machine.

Shells allow users to start new processes by typing in commands with their arguments. They also receive certain signals — such as keyboard interrupts — and forward them to the relevant processes.

Although the code you need to write is fairly simple — we've covered spawning processes and catching signals in lecture — the devil lies in the fact that several things might be happening *concurrently*. For example, your code might be modifying some data structure when control is suddenly transferred to a signal handler, which might also want to modify that data structure.


Advice
------

Make sure you review the process lectures, and understand *everything*; if you don't, you will find this lab very difficult.

You might also want to re-do the assigned readings for the process lectures.

Much of the documentation you will need is typically accessed via the ``man`` command at the Linux prompt, which brings up the manual page for a specific command or function. Here are some manual pages you might find useful:

- ``man 7 signal``
- ``man signal-safety``
- ``man execvp``
- ``man waitpid``
- ``man 2 kill``

Sometimes there are multiple manual pages for the same keyword in different "chapters". For example, ``kill`` is both a command you can type in at the shell prompt (section 1) and a system call (section 2), while ``signal`` is both a system call (section 2) and a concept (section 7). In those cases, you need the number to specify which one you mean.

Before starting the lab, experiment with the reference implementation to get a good idea of the functionality you will need to implement.

Read the Hints section for each task. It's there for a reason.

After implementing each task, double-check with the specification and the reference implementation to make sure your code meets the specification.

When you're logged into ``castor``, check that you don't have unwanted processes running. You can do this using the ``ps`` command, and you can kill unwanted processes using the ``kill`` command. ECE computers have a per-user limit on the number of concurrent processes; if you exceed this (and the processes are still running), you may be unable to log in again. In this lab, it's easy to create lots of processes that continue to run, and get in trouble.


Logistics
---------

As in the prior lab, we will use ``castor.ece.ubc.ca`` to evaluate your submissions. We have tried to make the code work across different machines, but if your submission works on some machine you have but not on ``castor.ece.ubc.ca``, you will be out of luck.

As in the other labs, all submissions are made by committing your code and pushing the commits to the assignment repo on GitHub.

As with all work in CPEN212, all of your code must be original and written by you from scratch after the lab has been released. You may not refer to any code other than what is found in the textbook and lectures, and you must not share code with anyone. See the syllabus for the academic integrity policy for details.


=================================
The ``crash`` shell specification
=================================

Reference implementation
------------------------

We have provided a reference implementation of ``crash`` which you can run with ``~cpen212/Public/lab5/crash-ref``. You might find it useful in case you're not 100% sure how things are supposed to work.


Prompt and parsing
------------------

The ``crash`` shell accepts inputs one line at a time from standard input. Each time a new line is being accepted, ``crash`` displays ``crash>`` followed by a single space (ASCII 32).

Each line consists of tokens deliminated by whitespace, ``&``, or ``;``. Spaces, tabs, ``&``, and ``;`` are not tokens (and are not commands). Multiple spaces/tabs are equivalent to one. Your implementation may limit whitespace to ASCII space and horizontal tabs (characters 32 and 9).

Both ``;`` and ``&`` terminate a command; the remainder of the line constitutes separate commands. Programs launched by commands terminated with ``&`` will run in the background.

For example, ``foo     bar;glurph&&quit``  has four commands: one that runs ``foo bar`` in the foreground, another that runs ``glurph`` in the background, an empty command (technically in the background), and the ``quit`` shell command.

You do not need to implement any quoting or character escape mechanisms.


Commands
--------

The following commands are typed at the ``crash>`` prompt; pressing the Enter key executes the command.

All commands below must be supported. In the following examples, job IDs, process IDs, and programs to run and their arguments, will of course all vary depending on the circumstances.

- ``quit`` takes no arguments and exits ``crash``.

- ``jobs`` lists the jobs currently managed by ``crash`` that have not terminated.

- ``nuke`` kills all jobs in this shell with the KILL signal.

- ``nuke 12345`` kills process 12345 with the KILL signal if it is a job in this shell.

- ``nuke %7`` kills job %7 with the KILL signal.

- ``fg %7`` puts job 7 in the foreground.

- ``fg 12345`` puts process 12345 in the foreground.

- ``bg %7`` puts job 7 in the background.

- ``bg 12345`` puts process 12345 in the background.

- ``foo bar glurph`` runs the program ``foo`` with arguments ``bar`` and ``glurph`` in the foreground, inheriting the current environment.

- ``foo bar glurph &`` runs the program ``foo`` with arguments ``bar`` and ``glurph`` in the background, inheriting the current environment.

Separate commands may be separated with newlines, ``;``, or ``&``, so ``jobs ; quit`` or ``foo bar & quit`` each have two separate commands. Empty commands (i.e., commands that consist of no tokens) have no effect.

Commands that identify a job or a process (``fg``, ``bg``, and ``nuke``) **only work if the job or process was launched from the current shell** (i.e., they do not work on external processes). Sending *any* signals to a process not spawned by the current instance of your shell is considered **incorrect behaviour.**

Commands that launch programs search the current PATH for the program binary (e.g., ``ls`` should run ``/bin/ls`` if ``/bin`` is first in your PATH).


Job numbers and PIDs
--------------------

Jobs are launched with sequential job numbers starting at 1 (including jobs that failed to run), and should go up to at least 2,147,483,647 before reusing previous numbers. Zero is not a valid job number. No two concurrently running jobs may have the same job number.

Process IDs you display must match the PID assigned by the OS.


Messages
--------

All non-error messages go to *standard output* (*not* to standard error).

In all the examples below, the job IDs, process IDs, and programs being run (``sleep``) are for illustration purposes and will vary to match the circumstances.

- The ``jobs`` command shows the jobs currently in existence (i.e., running or suspended), one job per line. Each line shows the job number (1 and 2 in the example below), process IDs (12345 and 12346 in the example below), the status (``running`` or ``suspended``), and the command being run without its arguments (``sleep`` below). The jobs are sorted by job number, in ascending order::

        [1] (12345)  running  sleep
        [2] (12346)  suspended  sleep

- When a job is placed in the background, either by ending the command with ``&`` or via the ``bg`` command, ``crash`` prints::

        [1] (12345)  running  sleep

- When a *background* job terminates normally (not because of a signal), ``crash`` prints::

        [2] (12345)  finished  sleep

- When a job is suspended by sending STOP or TSTP signals (e.g., by pressing :kbd:`Ctrl+Z` for a foreground job), ``crash`` prints::

        [2] (12345)  suspended  sleep

- When a suspended job resumes execution  normally (not because of a signal), ``crash`` prints::

        [2] (12345)  finished  sleep

- When a job is terminated by any signal (e.g., by pressing :kbd:`Ctrl+C` or :kbd:`Ctrl+\\` for a foreground job, a segfault, etc.), ``crash`` prints one of these two messages, depending on whether the process also dumped core::

        [1] (12345)  killed  sleep
        [1] (12345)  killed (core dumped)  sleep

  Typically signals like SIGQUIT (:kbd:`Ctrl+\\`) or SIGSEGV cause the process to dump core, while signals like SIGTSTP (:kbd:`Ctrl+C`) don't.

Note the double spaces before the status and the command names in all cases; you must preserve these exactly.

All commands are displayed without arguments, but with any path that was provided when the command was started. For example, if you ran the command ``sleep 10 &`` you might see::

        [1] (12345)  running  sleep

but if you ran ``/usr/bin/sleep 10&`` you might see::

        [1] (12345)  running  /usr/bin/sleep


Errors
------

All errors go to *standard error* (*not* to standard output).

The ``quit`` and ``jobs`` commands can print the following error:

- ``ERROR: quit takes no arguments`` if the command receives arguments (mutatis mutandis).

The ``fg`` and ``bg`` commands can print this error:

- ``ERROR: fg takes exactly one argument`` if there are no arguments / too many arguments (mutatis mutandis).

Commands that take process ID or job number arguments (``nuke``, ``fg``, and ``bg``) can also print several kinds of errors:

- ``ERROR: bad argument for fg: %133t`` if the job ID cannot be parsed as an integer (mutatis mutandis).

- ``ERROR: bad argument for fg: 133t`` if the process ID cannot be parsed as an integer (mutatis mutandis).

- ``ERROR: no job %1337`` if the shell has no running or stopped job with the given job ID.

- ``ERROR: no PID 1337`` if the shell has no running or stopped job with the given process ID.

Commands that launch programs can print the following error:

- ``ERROR: cannot run foo`` (mutatis mutandis) if the program ``foo`` cannot be executed for any reason (e.g., not found on path, no permissions, etc). The error message does *not* include the arguments passed to the program.

- ``ERROR: too many jobs`` if there are already 32 jobs running on suspended when a command to start another job is issued (in which case the new job does not start).

On error, the relevant command has no effect other than printing the error message.


Keyboard inputs
---------------

- :kbd:`Ctrl+C` kills the foreground process (if any) via the SIGINT signal. If there is no foreground process, this signal is ignored.

- :kbd:`Ctrl+Z` suspends the foreground process (if any) via the SIGTSTP signal. If there is no foreground process, this signal is ignored.

- :kbd:`Ctrl+\\` sends SIGQUIT to the foreground process (if any). If there is no foreground process, exits ``crash`` with exit status 0.

- :kbd:`Ctrl+D` at the ``crash>`` prompt exits the shell, or goes to the foreground process if there is one.

- all other keyboard inputs go to the foreground process if there is one, and to the shell if there isn't one.


Permissible differences with ``crash-ref``
------------------------------------------

Your implementation should match the behaviour of ``crash-ref``, with the following minor exceptions permitted to simplify your life:

- You may assume the total number of characters in a single input line does not exceed 1024, including the terminating ``\0`` character. (This is unlimited in ``crash-ref``.)

- You may assume that we will not execute more than 2,147,483,647 commands in a single session. Note that this refers to the number of jobs ever launched in one instance of the shell, not the number of jobs started concurrently. (The number of commands is unlimited in ``crash-ref``.)

- You may assume inputs are all ASCII characters, and that "whitespace" means ASCII space and ASCII tab (characters 32 and 9). (``crash-ref`` allows Unicode inputs and considers whitespace to be any code point with the ``White_Space`` property set.)


======
Coding
======

Template
--------

We've provided a template of ``crash.c`` in each task directory. We have already implemented the annoying but boring command parsing bit for you, as well as the ``quit`` command. We've also created a skeleton of some reasonable division of the work into functions; we will, however, test your code by interacting with the binary, so feel free to refactor the functions as you see fit.

For each task, you will need to replace ``crash.c`` file with the implementation that satisfies the relevant task requirements.

The tasks are cumulative. For example, Task 3 must implement all the functionality from Tasks 1 and 2, and so on.


Rules
-----

Some rules you must obey when writing code:

- When compiling your code, we will only compile ``crash.c`` in the relevant directory using a fresh copy of the ``Makefile``. This means that all your code must be in ``crash.c``.

- You may define whatever additional functions and variables you like, but you may not use any names that start with a double underscore (e.g., ``__foo``).

- Your code must be in C (specifically ISO C17).

- Your code must not require linking against any libraries other that the usual ``libc`` (which is linked against by default when compiling C).

- Needless to say, your code must compile and run without errors. If we can't compile or run your code, you will receive no credit for that task.

If you violate these rules, we will likely not be able to compile and/or properly test your code.

================================
Task 1: Starting background jobs
================================

When a shell runs a *background* job, control returns to the shell, and any keys you press go to the shell. The shell displays the prompt immediately, and you can issue more shell commands; keystrokes that would normally send signals to the process (e.g., :kbd:`Ctrl+C`) send them to the shell instead.


Requirements
------------

Required functionality:

- Typing a command name with arguments and ``&`` at the end should spawn a new process with the command / args, as specified.

- The ``quit`` command should work as specified.

- :kbd:`Ctrl+D` should work as specified.

You will likely want to implement the ``spawn()`` function, at least when ``bg`` is ``true``.


Deliverables
------------

- The modified ``crash.c`` in the ``task1`` directory, committed and pushed to GitHub.


Hints
-----

- How do you search the PATH for the executable you want? ``execvp`` is a wrapper for the ``execve`` system call that does just that. ``man execvp`` for more info.

- Remember to mask and unmask signals appropriately when you fork and modify any data structures to avoid race conditions.

- Check the messages and errors specification and the reference shell to make sure you produce the correct message when your job starts, and so on.

- The ``sleep`` program is really useful for testing throughout this lab, because it runs for a specified number of seconds and then finishes. It is installed by default on Linux machines.

- If you do use ``sleep``, don't make the time too long, or you might hit the per-user process limit.


==========================
Task 2: Listing job status
==========================

In this task, you will implement the ``jobs`` command that describes the status of jobs you've started inside ``crash``. This means you need to implement a data structure for tracking these jobs.


Requirements
------------

- The ``jobs`` command should display all jobs that have been started, as in the spec.

- Because you have not implemented the child signal handler, you will not know when jobs have terminated, so jobs that have died will be included in this list; this is fine for this Task *only*.

You will likely want to do:

- Create a data structure to keep track of jobs.

- Implement the ``spawn()`` function, at least when ``bg`` is ``true``.

- Implement the ``cmd_jobs()`` function.


Deliverables
------------

- The modified ``crash.c`` in the ``task2`` directory, committed and pushed to GitHub.


Hints
-----

- Remember to mask and unmask signals appropriately when you fork and modify any data structures to avoid race conditions.

- Check the specification and the reference shell for any messages and errors you need to implement.

- You will likely want to define a ``struct`` that represents a single job, so it is easy to extend later.

- You will need to access this structure later in signal handlers; you can't pass any arguments to those and C doesn't have closures, so you might want to use a global variable to hold the job-tracking structures.

- We've capped the number of *concurrent* jobs you must support at 32, so the data structure can be fixed-size.

- We care about correctness first, so linear-time searches to find specific jobs or process IDs are acceptable.

- You will need to store the program name in this data structure to display it in the ``jobs`` output. You can't directly use one of the pointers inside ``toks`` because the contents will change when the next command is input, so you will hae to copy the program name. You can either use ``malloc`` or take advantage of the fact that we've capped the maximum line length you need to support.


==========================================
Task 3: Nuking and reaping child processes
==========================================

A job spawned by the shell could *terminate* -- either because it simply finished its work or because it crashed. The only way for the shell to know this is by being notified via the SIGCHLD signal. In this task, you will partially implement the signal handler for SIGCHLD.


Requirements
------------

- The shell must correctly handle to the SIGCHLD signal *when the child has terminated* in any way.

- Once a job has terminated, it should never again appear in the output of ``jobs``. This means you will need to either remove jobs from your job list or mark jobs as terminated.

- The messages specified for jobs that have terminated (either finished or died because of a signal) must be implemented, including the core dump annotation.

- The ``nuke`` command must be implemented as specified.

You will likely want to:

- Implement ``handle_sigchld``.

- Modify ``install_signal_handlers`` to install ``handle_sigchld``.

- Implement ``cmd_nuke``.

- Possibly modify your fork/exec code to avoid race conditions.


Deliverables
------------

- The modified ``crash.c`` in the ``task3`` directory, committed and pushed to GitHub.


Hints
-----

- Check the specification to make sure the outputs for ``jobs`` and all the messages are exactly correct. We will test this automatically so if you use a different format our marking code will not accept it.

- Be sure you avoid data races when accesssing shared data structures. This probably means that you need to disable signals while you modify these structures *outside* of signals.

- Carefully read the manual page for ``waitpid`` (``man waitpid``) and go through the lecture examples.

- Recall from lecture that signals are *not queued*, so you *might not* receive a separate SIGCHLD for every process that has terminated. Than means you will need to reap *all* processes that have terminated.

- When using ``waitpid``, you don't want it to block if there are no children that can be be reaped. Make sure you provide suitable options when calling this function; ``man waitpid`` for details.

- Signals can be sent to other processes via the ``kill`` system call. Run ``man 2 kill`` to see its manual page.

- Note that ``nuke`` can take zero arguments or one, and if it does take an argument it can be either a job ID or a process ID. Be sure to implement *all variants*.

- For parsing numbers, you can use ``strtol`` (``man strtol`` for details). It detects errors better than ``atoi``. Of course neither will parse the initial ``%`` in a job number, so you need to work around that.

- Many useful functions are *unsafe* in signal handlers; ``man signal-safety`` for details.

  - In particular, ``malloc``, ``free``, and all variants of ``printf`` are unsafe. You can use ``write`` and ``strlen`` (both of which are safe) instead of ``printf``. To print numbers like job IDs or process IDs, you can either write your own function that uses ``write``, or see the next hint.

  - You can call these functions *outside* the signal handlers, though. For example, you can convert integers to strings when you first spawn the job, and store them in the job-tracking data structure.


=======================
Task 4: Foreground jobs
=======================

In contrast to the *background* job mechanism you've already implemented, a *foreground* job accepts inputs from the console.

The shell waits for the foreground job to finish before displaying the prompt and accepting more commands. Keystrokes that send signals send them to the foreground job.

At any time, there may be either exactly one foreground job, or no foreground jobs.


Requirements
------------

- Jobs started without the trailing ``&`` must pause the shell until they terminate (or stop).

- The SIGINT and SIGQUIT signals (whether sent via :kbd:`Ctrl+C` and :kbd:`Ctrl+\\` or received externally) must operate as specified *both* when there *is* a foreground job and when there is *no* foreground job.

- When no foreground job is running, issuing the ``fg`` command with a valid job ID or process ID must make the relevant background job a foreground job.

You will likely want to:

- Add a data structure (or modify your existing job list one) to identify which job is in the foreground (if there is one).

- Implement ``spawn`` when ``bg`` is ``true``.

- Implement ``handle_sigint``.

- Implement ``handle_sigquit``.

- Make sure these handlers are installed as appropriate.

- Implement ``cmd_fg``.


Deliverables
------------

- The modified ``crash.c`` in the ``task4`` directory, committed and pushed to GitHub.


Hints
-----

- How do you pause the shell? What you want to do is wait in one place until a signal terminates or stops the foreground job. A spin-loop is one way to do this, but it's crazily inefficient; see below for better ideas.

- There is a ``pause`` function call that waits until some signal is received. But you can't use it because you could run into a race condition: if the child quits, you might receive a SIGCHLD for it *before* ``pause`` starts, and then the ``pause`` would never finish.

- The easiest thing is to use ``sleep`` or ``nanosleep`` instead, as they also return when a signal is received. As usual, use ``man`` to read the manual pages. If you do this, be sure to sleep for *no more than 1ms at a time*.

- ``sleep`` will return when *any* signal is received, but this might not be a signal for the foreground job. This means you need to run ``sleep`` in a loop until the foreground job has terminated.

- You will of course need to track which job is in the foreground. It's easiest to do if you remember that there is either only one foreground job or none, and that job IDs as well as process IDs are never less than 1.

- Make sure to check the specification and the reference shell that you've implemented any messages and errors correctly.

- The ``kill`` function can send any signal to a process, not just SIGKILL. In particular you will need to forward some signals to a foreground child process if there is one.

- For this task you don't need to handle the case when the foreground job is *stopped*, just terminated. Stopped jobs are in the next task.


====================
Task 5: Stopped jobs
====================

Not surprisingly, a *stopped* job is one that is not currently running, unlike background and foreground jobs. Stopped jobs may be restarted either in the foreground or the background, or they can be terminated.

Jobs can be paused by receiving the SIGSTOP or SIGTSTP signals (the latter of which can be sent via :kbd:`Ctrl+Z` to a foreground process), and resumed by receiving the SIGCONT signal.


Requirements
------------

- The SIGTSTP signal must work as specified, whether sent by :kbd:`Ctrl+Z` to a foreground job or externally to a foreground or background job.

- Running ``fg`` or ``bg`` commands that specify a stopped job will resume the job and place it in the foreground or background depending on the command.

- Processes resumed by receiving SIGCONT from an external source continue as if resumed by the ``bg`` command.

- The ``jobs`` command must reflect whether each job is running or stopped, as specified.

You will likely want to:

- Modify your data structure to record whether a job is stopped or running.

- Implement ``handle_sigtstp``.

- Update ``handle_sigchld`` to deal with stopped and continued jobs.

- Update ``cmd_jobs`` to correctly list stopped jobs.

- Implement ``cmd*bg`` and update ``cmd*fg`` to handle stopped jobs.


Deliverables
------------

- The modified ``crash.c`` in the ``task5`` directory, committed and pushed to GitHub.


Hints
-----

- Read ``man waitpid`` again, especially the section about ``wstatus``. This allows you to determine whether the relevant child was terminated or stopped.

- To resume a stopped job, send a SIGCONT signal to it via the ``kill`` function.

- Be sure that jobs that are resumed as foreground cause the shell to pause as if they were launched without ``&``.

- Check the specification and the reference shell to make sure you've correctly implemented any messages or errors.


=====
Marks
=====

- Task 1: 2
- Task 2: 2
- Task 3: 2
- Task 4: 2
- Task 5: 2
