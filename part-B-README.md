# Ansible Ad-hoc Commands Lab

## Project Overview

This project demonstrates the use of Ansible ad-hoc commands to manage remote Linux servers.

Instead of writing Ansible playbooks, this exercise focuses on running individual Ansible modules directly from the command line to perform common system administration tasks.

The tasks covered include:

- Testing server connectivity
- Running commands remotely
- Gathering system information
- Installing packages
- Managing services
- Creating files and directories
- Managing users
- Editing configuration files
- Understanding host patterns and parallel execution

---

# Inventory Structure

Ansible uses an inventory file to know which machines it should manage.

My inventory contains three sections:

```ini
[web]
webserver ansible_host=<web-server-ip> ansible_user=ubuntu

[db]
dbserver ansible_host=<database-server-ip> ansible_user=ubuntu

[local]
localnode ansible_connection=local

[datacenter:children]
web
db
```

The datacenter group allows me to target all production servers without including my local machine.

The final structure is:
```text
datacenter
├── web
│   └── webserver
│
└── db
    └── dbserver


local
└── localnode
```

----

# Understanding Ansible Ad-hoc Command Syntax

The general structure of an Ansible ad-hoc command is:
```bash
ansible -i <inventory> <host-pattern> -m <module> -a "<arguments>"
```

Example:
```bash
ansible -i inventory all -m ping
```
Explanation:

- ansible
	* Runs an Ansible ad-hoc command.
- -i inventory
	* Specifies the inventory file Ansible should use.
- all
	* Specifies which hosts should receive the command.
- -m ping
	* Specifies the Ansible module to execute.
- -a
	* Provides arguments passed to the module.

---

## Privilege Escalation

Some tasks require administrator permissions because they modify system-level resources.

For these tasks, I used:
```bash
--become
``` 
OR
```bash
-b
```
This tells Ansible to execute the command with elevated privileges using sudo.

Examples of operations requiring root privileges:

- Installing packages
- Managing services
- Creating users
- Editing files inside /etc

Equivalent manual command:
```bash
sudo <command>
```

----

# 1. Test Connectivity to All Hosts

## Objective

The first step when managing servers with Ansible is verifying that the control machine can communicate with all managed hosts.

Instead of using the traditional `ping` utility, Ansible provides a built-in `ping` module that checks:

- SSH connectivity
- Authentication
- Python availability on the remote machine
- Whether Ansible can successfully execute modules

---

## Command

```bash
ansible -i inventory all -m ping
```

---

## Explanation

The command uses the `ping` module:

```bash
-m ping
```

The Ansible ping module does not send ICMP packets like the normal Linux `ping` command.

Instead, it connects to the remote machine through SSH and runs a small test module.

A successful response looks like:

```text
webserver | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

dbserver | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

The response:

```json
"ping": "pong"
```

confirms that Ansible can communicate with the host successfully.

---

## Result

All managed servers responded successfully.

This confirmed that:

- The inventory file was correctly configured.
- SSH connectivity was working.
- The `ubuntu` user could access the EC2 instances.
- Ansible was ready to execute further commands.

---

## Issue Encountered

During execution, Ansible displayed this warning:

```text
[WARNING]: Host 'localnode' is using the discovered Python interpreter at '/opt/anaconda3/bin/python3.12'
```

---

## Explanation of the Warning

Ansible requires Python on managed machines because most Ansible modules are Python-based.

The warning means:

> "Ansible found Python on this machine, but if another Python installation appears later, it may choose a different interpreter."

This happened because `localnode` points to my local Mac:

```ini
[local]
localnode ansible_connection=local
```

Ansible detected my Anaconda Python installation:

```text
/opt/anaconda3/bin/python3.12
```

This was only a warning and did not affect execution.

---

## Lesson Learned

Before running commands against `all`, always understand what hosts are included.

In this inventory:

```ini
all
```

includes:

```
webserver
dbserver
localnode
```

If a task is meant only for cloud servers, it is safer to target:

```bash
datacenter
```

instead of:

```bash
all
```

to avoid accidentally running administrative tasks on the local machine.

---

# 2. Display Current Date and Time on Web Servers

## Objective

The goal of this task was to execute a Linux command remotely on all servers inside the `web` group.

This introduces the Ansible `command` module, which allows us to run simple commands on managed hosts.

---

## Command

```bash
ansible -i inventory web -m command -a "date"
```

---

## Explanation

This command targets only hosts inside the `web` group:

```bash
web
```

From the inventory:

```ini
[web]
webserver ansible_host=<web-server-ip> ansible_user=ubuntu
```

Only the web server receives the command.

The module used is:

```bash
-m command
```

The `command` module executes a command directly on the remote machine.

In this case:

```bash
date
```

is a Linux command that displays the current system date and time.

Example output:

```text
webserver | CHANGED | rc=0 >>

Mon Jul 28 14:45:32 UTC 2026
```

---

## Why Use the Command Module?

The `command` module is useful when:

- Running simple Linux commands
- The command does not require shell features
- You do not need pipes, redirects, variables, or shell operators

Examples:

```bash
uptime
```

```bash
hostname
```

```bash
whoami
```

---

## Command Module Limitation

The `command` module does not run commands through a shell.

This means shell features such as:

```bash
|
```

(pipes)

```bash
>
```

(output redirection)

```bash
&&
```

(command chaining)

will not work.

For example, this would fail with the `command` module:

```bash
df -h / | tail -1
```

because the pipe operator requires a shell.

When shell functionality is required, the `shell` module should be used instead.

---

## Result

The command successfully returned the current date and time from the web server.

This confirmed that:

- Ansible could execute commands remotely.
- The `web` inventory group was correctly configured.
- The remote Ubuntu server accepted commands from the Ansible controller.

---

## Lesson Learned

The choice between `command` and `shell` depends on the complexity of the command:

| Module | Use Case |
|---|---|
| `command` | Simple commands without shell features |
| `shell` | Commands requiring pipes, redirects, or shell operators |

A good Ansible practice is to use `command` whenever possible because it is safer and more predictable.


----

# 3. Check Available Disk Space on Datacenter Servers

## Objective

The goal of this task was to check how much free disk space is available on the root filesystem (`/`) of every server in the `datacenter` group.

This task introduces the Ansible `shell` module and explains when it should be used instead of the `command` module.

---

## Command

```bash
ansible -i inventory datacenter -m shell -a "df -h / | tail -1"
```

---

## Explanation

This command targets the `datacenter` group:

```bash
datacenter
```

The group was created in the inventory as:

```ini
[datacenter:children]
web
db
```

This allows Ansible to target both production servers:

```
datacenter
├── webserver
└── dbserver
```

without including the local machine.

---

## The Shell Module

The module used is:

```bash
-m shell
```

The `shell` module executes commands through the remote machine's shell environment.

Unlike the `command` module, it supports shell features such as:

- Pipes (`|`)
- Redirects (`>`)
- Command chaining (`&&`)
- Environment variables

---

## Understanding the Linux Command

The command executed remotely is:

```bash
df -h / | tail -1
```

It contains two parts:

### 1. Check filesystem usage

```bash
df -h /
```

`df` means "disk filesystem".

The options:

```bash
-h
```

display sizes in a human-readable format:

Example:

```
G = Gigabytes
M = Megabytes
```

The `/` specifies that we want information about the root filesystem.

Example output:

```
Filesystem      Size  Used Avail Use%
/dev/root        20G  8.5G   11G  45%
```

---

### 2. Filter the output

```bash
| tail -1
```

The pipe sends the output of `df` into another command.

`tail -1` displays only the last line.

Without the pipe:

```bash
df -h /
```

we get:

```
Filesystem      Size  Used Avail Use%
/dev/root        20G  8.5G   11G  45%
```

With:

```bash
df -h / | tail -1
```

we only get:

```
/dev/root        20G  8.5G   11G  45%
```

This removes the header and gives the actual disk usage information.

---

## Why Not Use the Command Module?

Initially, it may seem like this should work:

```bash
ansible -i inventory datacenter -m command -a "df -h / | tail -1"
```

However, the command module does not understand shell operators.

It would interpret:

```bash
|
```

as part of the command itself instead of passing it to the shell.

The `shell` module is required because we need the pipe functionality.

---

## Result

The command successfully returned disk usage information from:

```
webserver
dbserver
```

Example:

```
webserver | CHANGED | rc=0 >>
/dev/root        20G  8.5G   11G  45%

dbserver | CHANGED | rc=0 >>
/dev/root        20G  7.9G   12G  40%
```

---

## Lesson Learned

Use:

```
command
```

for simple commands.

Use:

```
shell
```

when the command requires shell features.

However, because the shell module executes through a shell, it should only be used when necessary. For simple commands, `command` is safer because it avoids shell interpretation issues.

---

# 4. Retrieve Total Memory Using the Setup Module

## Objective

Retrieve only the total memory available on each managed host using Ansible facts.

---

## Command

```bash
ansible -i inventory all -m setup -a "filter=ansible_memtotal_mb"
```

---

## Explanation

The `setup` module gathers system information (facts) from managed hosts.

By default, it returns a large amount of information, including:

- Operating system details
- Network information
- CPU information
- Memory information
- Disk information

The `filter` argument limits the output to only the required fact:

```bash
filter=ansible_memtotal_mb
```

This returns the total RAM available on the host in megabytes.

Example output:

```json
"ansible_memtotal_mb": 1934
```

---

## Why Use a Filter?

Without a filter:

```bash
ansible -i inventory all -m setup
```

would return hundreds of lines of system information.

Using:

```bash
filter=ansible_memtotal_mb
```

makes the output smaller and easier to read.

---

## Result

Ansible successfully returned the total memory value for each host.

Example:

```
webserver | SUCCESS =>
    "ansible_memtotal_mb": 1934

dbserver | SUCCESS =>
    "ansible_memtotal_mb": 1934
```

---

## Lesson Learned

The `setup` module is used for gathering Ansible facts.

The `filter` option helps retrieve only the specific information required instead of collecting all available facts.


----

# 5. Install htop Using the Package Module

## Objective

Install the `htop` system monitoring tool on all servers in the `datacenter` group.

---

## Command

```bash
ansible -i inventory datacenter -m package -a "name=htop state=present" --become
```

---

## Explanation

The `package` module manages software packages on Linux systems.

It provides a distribution-independent way to install packages without needing to specify the package manager.

For example:

- Ubuntu → `apt`
- CentOS/RHEL → `yum` or `dnf`

The argument:

```bash
name=htop
```

specifies the package to install.

The argument:

```bash
state=present
```

ensures that the package exists on the server.

If the package is already installed, Ansible will leave it unchanged.

---

## Why Use --become?

Installing software modifies system directories and requires administrator privileges.

The `--become` option allows Ansible to execute the installation with root privileges.

Equivalent manual command:

```bash
sudo apt install htop
```

---

## Testing Idempotency

The same command was executed a second time:

```bash
ansible -i inventory datacenter -m package -a "name=htop state=present" --become
```

The second run returned:

```text
changed=0
```

because `htop` was already installed.

Ansible checked the current state of the system and did not make unnecessary changes.

---

## Result

First execution:

```text
changed=1
```

The package was installed.

Second execution:

```text
changed=0
```

The package was already present.

---

## Lesson Learned

Ansible modules are designed to be idempotent.

A task can be executed multiple times safely because Ansible only changes the system when the desired state has not already been achieved.


----

# 6. Ensure SSH Service Is Running and Enabled

## Objective

Ensure that the SSH service is running immediately and automatically starts after system reboot on all datacenter servers.

---

## Command

```bash
ansible -i inventory datacenter -m service -a "name=ssh state=started enabled=yes" --become
```

---

## Explanation

The `service` module is used to manage services running on Linux systems.

The arguments used:

```bash
name=ssh
```

Specifies the service to manage.

```bash
state=started
```

Ensures the service is currently running.

```bash
enabled=yes
```

Ensures the service starts automatically during boot.

---

## Why Use --become?

Managing system services requires administrator privileges.

The command performs operations similar to:

```bash
sudo systemctl start ssh
sudo systemctl enable ssh
```

so Ansible needs elevated permissions.

---

## Issue Encountered

The first attempt used:

```bash
ansible -i inventory all -m service -a "name=ssh state=started enabled=yes" --become
```

This failed on:

```text
localnode | FAILED!
sudo: a password is required
```

---

## Cause

The `all` pattern included every host in the inventory:

```
webserver
dbserver
localnode
```

`localnode` represents my local Mac machine:

```ini
[local]
localnode ansible_connection=local
```

Unlike the EC2 instances, my local machine required a sudo password when Ansible attempted privilege escalation.

---

## Solution

The command was changed to target only production servers:

```bash
ansible -i inventory datacenter -m service -a "name=ssh state=started enabled=yes" --become
```

The `datacenter` group contains only:

```
webserver
dbserver
```

---

## Result

The SSH service was successfully verified on the EC2 instances.

Example:

```
webserver | SUCCESS
dbserver | SUCCESS
```

---

## Lesson Learned

Avoid using `all` when the inventory contains machines that should not receive administrative commands.

Use specific groups such as:

```bash
web
db
datacenter
```

to control exactly which hosts are affected.


----

# 7. Create a File Using the Copy Module

## Objective

Create a file named `/tmp/hello.txt` on all hosts in the `web` group containing the text:

```
Hello from Ansible
```

---

## Command

```bash
ansible -i inventory web -m copy -a "content='Hello from Ansible' dest=/tmp/hello.txt"
```

---

## Explanation

The `copy` module is used to transfer files or content from the Ansible controller to managed hosts.

In this task, instead of copying an existing local file, the `content` argument was used:

```bash
content='Hello from Ansible'
```

This creates the file directly on the remote server with the specified text.

The destination is defined using:

```bash
dest=/tmp/hello.txt
```

which tells Ansible where to create the file.

---

## `content` vs `src`

The `copy` module can use two common approaches:

### Using `src`

Copies an existing file from the Ansible controller:

```bash
src=hello.txt
```

Example:

```
Ansible Controller
        |
        |
        v
Remote Server

hello.txt → /tmp/hello.txt
```

---

### Using `content`

Creates the file directly from provided text:

```bash
content="Hello from Ansible"
```

No local file is required.

For this task, `content` was required by the assignment.

---

## Result

The file was successfully created on the web server.

Verification:

```bash
cat /tmp/hello.txt
```

Output:

```text
Hello from Ansible
```

---

## Lesson Learned

The `copy` module can either:

- Copy an existing file using `src`
- Create a new file directly using `content`

The `content` argument is useful for creating small configuration files or adding simple text without creating temporary files on the Ansible controller.


----

# 8. Create a Directory Using the File Module

## Objective

Create a directory named `/tmp/ansible_test` on every host with permissions set to `0755`.

---

## Command

```bash
ansible -i inventory all -m file -a "path=/tmp/ansible_test state=directory mode=0755"
```

---

## Explanation

The `file` module is used to manage files and directories on remote hosts.

In this task, it ensures that a directory exists.

The arguments used:

```bash
path=/tmp/ansible_test
```

Specifies the location of the directory.

```bash
state=directory
```

Tells Ansible that the target should be a directory.

If the directory does not exist, Ansible creates it.

If it already exists, Ansible leaves it unchanged.

```bash
mode=0755
```

Sets the Linux permissions for the directory.

The permission breakdown:

```
Owner  → rwx (7)
Group  → r-x (5)
Others → r-x (5)
```

Meaning:

- The owner can read, write, and access the directory.
- Other users can read and access the directory.

---

## Result

The directory was created successfully:

```
/tmp/
└── ansible_test/
```

The permissions were set to:

```
0755
```

---

## Verification

The directory permissions can be checked with:

```bash
ls -ld /tmp/ansible_test
```

Example output:

```text
drwxr-xr-x 2 ubuntu ubuntu 4096 Jul 28 15:00 /tmp/ansible_test
```

---

## Lesson Learned

The `file` module manages the desired state of files and directories.

Using:

```bash
state=directory
```

does not simply create a directory every time. Instead, Ansible checks the current state and only makes changes when necessary.


----

# 9. Create a Deploy User on the Database Server

## Objective

Create a user called `deployer` on the database server only.

The user should:

- Have a home directory
- Not have a login shell

---

## Command

```bash
ansible -i inventory all -m user -a "name=deployer create_home=yes shell=/usr/sbin/nologin" --limit db --become
```

---

## Explanation

The `user` module manages Linux user accounts.

The arguments used:

```bash
name=deployer
```

Creates a user with the username:

```
deployer
```

---

```bash
create_home=yes
```

Creates the user's home directory:

```
/home/deployer
```

---

```bash
shell=/usr/sbin/nologin
```

Sets the user's login shell.

`/usr/sbin/nologin` prevents the account from being used for interactive login sessions.

This is commonly used for service accounts that need to exist but should not allow SSH access.

---

## Using --limit

The command uses:

```bash
--limit db
```

to restrict execution to only hosts in the database group.

Although the host pattern is:

```bash
all
```

Ansible only executes the task against:

```
dbserver
```

because of the limit.

---

## Why Use --become?

Creating users modifies protected system files such as:

```
/etc/passwd
/etc/shadow
/etc/group
```

These files require root privileges.

---

## Issue Encountered

The first attempt was:

```bash
ansible -i inventory all -m user -a "name=deployer create_home=yes shell=/usr/sbin/nologin" --limit db
```

It failed with:

```text
useradd: Permission denied.
useradd: cannot lock /etc/passwd
```

---

## Cause

The command was executed without privilege escalation.

Ansible connected successfully to the server, but the `ubuntu` user did not have permission to create system users.

---

## Solution

The command was rerun with:

```bash
--become
```

This allowed Ansible to perform the operation as root.

---

## Result

The `deployer` user was successfully created on:

```
dbserver
```

Verification:

```bash
id deployer
```

and:

```bash
grep deployer /etc/passwd
```

confirmed:

- The user exists.
- The home directory was created.
- The login shell is set to `/usr/sbin/nologin`.

---

## Lesson Learned

User management tasks require elevated privileges because they modify system account files.

The `user` module can also be used to create service accounts by disabling interactive login with:

```bash
shell=/usr/sbin/nologin
```


----

# 10. Manage `/etc/motd` Using the lineinfile Module

## Objective

Add the following message to `/etc/motd` on all datacenter servers:

```
# Managed by Ansible
```

The line should only be added if it does not already exist.

---

## Command

```bash
ansible -i inventory datacenter -m lineinfile -a "path=/etc/motd line='# Managed by Ansible' create=yes" --become
```

---

## Explanation

The `lineinfile` module manages individual lines inside text files.

It is commonly used for:

- Updating configuration files
- Adding settings
- Ensuring required entries exist

The arguments used:

```bash
path=/etc/motd
```

Specifies the file to modify.

```bash
line='# Managed by Ansible'
```

Defines the exact line that should exist in the file.

```bash
create=yes
```

Creates the file if it does not already exist.

Without this option, `lineinfile` expects the file to already be present.

---

## Why Use --become?

The file:

```
/etc/motd
```

is a system file owned by root.

Modifying it requires administrator privileges.

---

## Issue Encountered

The first attempt used:

```bash
ansible -i inventory all -m lineinfile -a "path=/etc/motd line='# Managed by Ansible'" --become
```

Two issues occurred.

### Issue 1: Local Machine Failure

The command failed on:

```text
localnode | FAILED!
sudo: a password is required
```

---

### Cause

The `all` pattern included:

```
webserver
dbserver
localnode
```

Since `localnode` represents my Mac, Ansible attempted to use sudo locally, which required my Mac administrator password.

---

### Solution

The target was changed from:

```bash
all
```

to:

```bash
datacenter
```

so only the EC2 servers were affected.

---

### Issue 2: Missing `/etc/motd`

The EC2 servers returned:

```text
Destination /etc/motd does not exist!
```

---

### Cause

The `lineinfile` module modifies existing files by default.

Since `/etc/motd` was not present on the servers, Ansible could not update it.

---

### Solution

Added:

```bash
create=yes
```

This allowed Ansible to create the file before adding the required line.

---

## Result

The file was created and updated successfully.

Verification:

```bash
cat /etc/motd
```

Output:

```text
# Managed by Ansible
```

Running the command again does not add duplicate entries because `lineinfile` checks whether the line already exists.

---

## Lesson Learned

The `lineinfile` module is useful for maintaining configuration files because it is idempotent.

It ensures the required line exists without repeatedly adding duplicates.


---

# 11. Run Commands Using Host Patterns

## Objective

Run an Ansible command against hosts selected by a naming pattern instead of using an inventory group.

---

## Command

```bash
ansible -i inventory "*server" -m ping
```

---

## Explanation

Ansible allows hosts to be selected using **patterns**.

Patterns match hostnames from the inventory.

The pattern used:

```bash
"*server"
```

matches any host whose name ends with:

```
server
```

From the inventory:

```ini
[web]
webserver

[db]
dbserver
```

The matching hosts are:

```
webserver
dbserver
```

---

## Group vs Pattern

Ansible groups are defined in the inventory:

Example:

```ini
[web]
webserver
```

Targeting a group:

```bash
ansible -i inventory web -m ping
```

uses the group name.

A pattern matches hostnames dynamically:

```bash
ansible -i inventory "*server" -m ping
```

does not depend on a group.

---

## Issue Encountered

The first attempt used:

```bash
ansible -i inventory "node*" -m ping
```

The result was:

```text
[WARNING]: Could not match supplied host pattern, ignoring: node*
No hosts matched, nothing to do
```

---

## Cause

The pattern:

```bash
node*
```

means:

> Match any host whose name starts with "node"

However, the inventory did not contain any hosts starting with `node`.

Current hostnames were:

```
webserver
dbserver
localnode
```

Since no hostname matched the pattern, Ansible had nothing to execute against.

---

## Solution

A pattern matching the existing inventory was used:

```bash
"*server"
```

This successfully matched:

```
webserver
dbserver
```

---

## Result

The ping command successfully executed against hosts matching the pattern.

Example:

```text
webserver | SUCCESS => {
    "ping": "pong"
}

dbserver | SUCCESS => {
    "ping": "pong"
}
```

---

## Lesson Learned

Ansible patterns are based on inventory hostnames, not IP addresses or group names.

Before using a wildcard pattern, check the actual hostnames defined in the inventory.

Examples:

```bash
web*
```

matches hosts starting with `web`.

```bash
*server
```

matches hosts ending with `server`.


---

# 12. Run Ping with 10 Parallel Forks

## Objective

Run the Ansible ping command using 10 parallel execution processes.

---

## Command

```bash
ansible -i inventory all -m ping -f 10
```

---

## Explanation

The `-f` option controls the number of **forks** Ansible uses.

A fork represents the number of hosts Ansible can communicate with simultaneously.

By default, Ansible uses a smaller number of parallel connections. Increasing the fork count allows commands to execute against more machines at the same time.

Example:

With:

```bash
-f 10
```

Ansible can handle up to:

```
10 hosts simultaneously
```

---

## Why Forks Matter

For a small inventory:

```
2 servers
```

the difference is not noticeable.

However, in larger environments:

```
500 servers
```

running commands sequentially would take much longer.

Increasing forks allows Ansible to complete tasks faster by communicating with multiple servers at once.

---

## Result

The ping command executed successfully using 10 parallel forks.

Example:

```text
webserver | SUCCESS => {
    "ping": "pong"
}

dbserver | SUCCESS => {
    "ping": "pong"
}
```

---

## Lesson Learned

Forks control Ansible's level of parallelism.

Higher fork values can improve execution speed in larger environments, but they should be configured carefully because each fork consumes system resources such as CPU, memory, and network connections.


----

# Skills Demonstrated

This project demonstrates:

- Writing and using Ansible inventories
- Running Ansible ad-hoc commands
- Using Ansible modules:
  - ping
  - command
  - shell
  - setup
  - package
  - service
  - copy
  - file
  - user
  - lineinfile
- Managing Linux servers remotely
- Using privilege escalation with become
- Understanding Ansible idempotency
- Using host groups and host patterns
- Running parallel execution with forks





































