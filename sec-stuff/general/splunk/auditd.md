---
description: >-
  Auditd-related search macros, datamodel queries, transformations and
  configurations
---

# Auditd

Auditd is a tool for maintaining logs for events that take place on Linux systems to help system administrators and security analysts monitor security breaches and incidents.

## Rules

{% embed url="https://www.redhat.com/sysadmin/configure-linux-auditing-auditd" %}

You can use [auditd.rules](https://github.com/Neo23x0/auditd/blob/master/audit.rules) to get a better idea of how rules are written for auditd.

## Configuration

Auditd, by default, stores logs in `/var/log/audit` directory. This path can be changed in the `auditd.conf`  file.&#x20;

Read more about it at - [https://man7.org/linux/man-pages/man5/auditd.conf.5.html](https://man7.org/linux/man-pages/man5/auditd.conf.5.html)

The logs are stored in ASCII format as key-value pairs. Some of the values, such as `proctitle`, are encoded in hexadecimal format. You can use `ausearch -i` to decode it while viewing logs on the command line. The logs will have to be decoded using evals when importing them into Splunk -

<details>

<summary>Extracting time in a human-readable format along with the message ID</summary>

<pre class="language-splunk-spl"><code class="lang-splunk-spl">| rex field = msg "(?&#x3C;unixtime>\\d{10}\.\\d{3})"
<strong>| rex field = msg ":(?&#x3C;msg_id>\\d{6})"
</strong>| eval readable_time = strftime(unixtime, "%Y-%m-%d %H:%M:%S.%Q")
</code></pre>

</details>

<details>

<summary>Decoding "proctitle" field</summary>

```splunk-spl
| eval process_title = urldecode(replace(proctitle,"([0-9A-F]{2})","%\1"))
```

</details>

Since each event is stored over multiple lines, when imported into Splunk, it will be split up into multiple "events" that will have to be grouped using the `msg` field. This can be done using the `transaction` command or by using a subsearch.&#x20;

<details>

<summary>Using a subsearch to find complete events based on a key</summary>

```splunk-spl
index = auditd
| search [
    index = auditd
    | search key = <key_name>
    | fields msg
    | format ]
```

</details>
