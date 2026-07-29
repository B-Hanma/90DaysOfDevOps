Day 02 — Linux Architecture, Processes, and systemd
Core Components of Linux (kernel, user space, init/systemd)

1. Kernel: It's the core component of any operating system, which directly talks with the hardware and works as a gatekeeper — it accepts requests coming from the shell and works on their behalf.

2. User space: User space is anything which is not directly talking to the hardware — a user (human) can interact with it and put in commands or make changes.

3. Init and systemd: Once the kernel loads, systemd turns up with PID 1, and it is responsible for starting and managing everything in the system — this role is called the init system. Init system is like a post (job title), and systemd is the first employee to hold it.

How Processes Are Created and Managed

Once the system is turned up and managed by systemd, whenever you click on any service or tool, the kernel starts a process for it and tracks it. There are three states a process can be in:

1. Running — the process is actively using the CPU.
2. Sleeping — the process is waiting for something (like user input or a resource) before it can continue running.
3. Zombie — a process that has finished its work, but the parent process hasn't acknowledged it yet. It's like a ghost process — not using any CPU, but still occupying a unique process ID. If zombies pile up over time, it can lead to future errors (running out of available process IDs).

5 Commands I Use Daily
cd — change directory
ls — list all files
cat — to read file contents
mkdir — to make a new directory
man — to read documentation about commands
