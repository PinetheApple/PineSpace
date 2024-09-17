---
layout:
  title:
    visible: true
  description:
    visible: false
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
---

# Fundamentals

This section only covers the last room of the module. The other rooms are freely accessible on TryHackMe.

{% embed url="https://tryhackme.com/module/red-team-fundamentals" %}
Link to the module
{% endembed %}

## Intro to C2

C2 stands for "Command and Control." It is essentially a server through which you can send commands to / communicate with devices that are compromised and in your control.

### Structure of C2

A C2 framework consists of the following -

* C2 Server
  * This serves as the central location where agents call back to and through which operators send commands to compromised devices
* Agents / Payloads
  * An agent is a program running on the compromised device that calls back to a listener on the C2 server. It has greater functionality than a standard reverse shell in most cases.
* Listeners
  * A listener is a program running on the C2 server waiting for call backs from an agent over a certain port or protocol.
* Beacons
  * The process by which an agent calls back to a listener on a C2 server is known as a beacon.

### Obfuscating Agent Callbacks

Agents send out beacons to C2 server periodically. If these beacons are sent out at regular intervals or often they can be easily picked up on by security solutions or analysts. To make it harder to detect, it is important to obfuscate the callbacks. This can be done by using one of the following -

* Sleep timers
  * Sleep timers can be used to ensure that agents wait for a specified period of time before sending out a beacon.
* Jitter
  * Jitter refers to the introduction of random variation in the timing of callbacks from agents the the C2 server. It is essentially randomizing the duration of the sleep timer after each callback.

### C2 Modules

Modules add the ability to make C2 agents and the server more flexible. Examples of modules are -

* Post-exploitation module
* Pivoting module

### Facing the World



### Common Frameworks



<details>

<summary>Open-source frameworks</summary>

* Metasploit
* Armitage
* Powershell Empire/Starkiller
* Covenant
* Sliver

</details>

<details>

<summary>Paid / Premium frameworks</summary>

* Cobalt Strike
* Brute Ratel

</details>

{% embed url="https://howto.thec2matrix.com/" %}
website that covers various free C2 frameworks
{% endembed %}

### Setting up a C2 Server



<details>

<summary>Preparing the environment</summary>

It is important to ensure that metasploit is properly configured before using Armitage. Run the following commands to initialize the database after installing metasploit -

```bash
systemctl start postgresql && systemctl status postgresql;
msfdb --use-defaults delete;
msfdb --use-defaults init;
```

</details>

<details>

<summary>Starting and connecting to Armitage</summary>

Run the following command to start the Armitage server -

```bash
cd /opt/armitage/release/unix && ./teamserver <ip_address> <password>;
```

* ip\_address - the IP address of the server
* password - the shared password to access the server

Run the following command to start the Armitage client to connect to the server -

```bash
cd /opt/armitage/release/unix && ./armitage
```

Enter a user and the password used when starting the server. The user is a nickname and not a username for authentication.

</details>



### Accessing and Managing C2 Infrastructure



### Listener Types



### Advanced C2 Setup



## Additional Resources

{% embed url="https://redfoxsec.com/blog/introduction-to-c2-frameworks/" %}

{% embed url="https://hackmd.io/@VJ99/B1nBSfmZi" %}
