---
description: Warmups
---

# Finder's Fee

<figure><img src="../../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

The name and description of the challenge is a pretty big giveaway as to what you have to do to get the flag. Once we log onto the server using ssh, we can use the `ls` command to list the files/directories present in `/home`. `user` does not have permissions to read the contents of `finder`'s home directory.

<figure><img src="../../../.gitbook/assets/Screenshot from 2024-10-11 19-42-34.png" alt=""><figcaption></figcaption></figure>

Checking `/bin`, we see that it links to `/usr/bin`. Listing out the files, the `find` binary stands out as it has it's [SGID](https://linuxconfig.org/how-to-use-special-permissions-the-setuid-setgid-and-sticky-bits) bit set. This allows us to run the `find` command as though we were in the `finder` group.

<figure><img src="../../../.gitbook/assets/Screenshot from 2024-10-11 19-43-12.png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/Screenshot from 2024-10-11 19-43-30.png" alt=""><figcaption></figcaption></figure>

We can use the `find` command to list out the files present in `finder`'s home directory, something that we did not have the permission to do previously.

<figure><img src="../../../.gitbook/assets/Screenshot from 2024-10-11 19-45-07.png" alt=""><figcaption></figcaption></figure>

We can specify the `-exec` flag to execute a command based on the result of the search. The command we will be using in this case is `cat` to print out the flag.

<figure><img src="../../../.gitbook/assets/Screenshot from 2024-10-11 19-47-37.png" alt=""><figcaption></figcaption></figure>
