# Checkpoint 1

**The verification of Alice and Bob being in their respective groups**
![Pasted image 20260914035554](assets/Pasted%20image%2020260914035554.png)


Alice and Root Showing Permissions while Bob gets denied

![Pasted image 20260914041416](assets/Pasted%20image%2020260914041416.png)

- `sudo -u bob cd ...` **wasn’t a valid access test** because `cd` is built into the shell. Bob `ls` and `touch` tests were valid.
- Setgid controls **group inheritance**, not the new file’s permission bits. Your file is `-rw-rw-r--`; Bob still cannot access it through this directory because the directory blocks him.

# Checkpoint 2

Creation of a private section where only Alice the **Owner** and Root can access, not even Alice's group
![Pasted image 20260914043617](assets/Pasted%20image%2020260914043617.png)

# Checkpoint 3

Creation of directories and how umask changes permissions
![Pasted image 20260914122207](assets/Pasted%20image%2020260914122207.png)

Stricter umask conditions
![Pasted image 20260914122754](assets/Pasted%20image%2020260914122754.png)

**changing umask affects future file and directory creation, not existing permissions**. **Even when umask is changed all folders have specific permissions**
![Pasted image 20260914124158](assets/Pasted%20image%2020260914124158.png)

# Checkpoint 4

**Editing the sudoers.d file via visudo and adding permission for alice to use the command systemctl status ssh with sudo**
![Pasted image 20261120190749](assets/Pasted%20image%2020261120190749.png)

**Normal users like alice are able to run systemctl status on services regularly, however due to the rule alice is able to run systemctl status ssh as root. The command allowed to be run with sudo must be the exact one in the rule which was ``sudo systemctl status ssh --no-pager`` .**
**This shows her attempting to run it normally, running a different command and getting access denied, running the intended command without correct arguments, and then finally running the exact command with the correct arguments and receiving the desired information. Alice was not allowed to disable SSH or use sudo without required arguments**

![Pasted image 20261120192148](assets/Pasted%20image%2020261120192148.png)

Listing all the commands alice can run as root and making sure its only the given command
![Pasted image 20261120193513](assets/Pasted%20image%2020261120193513.png)


As further evidence, bob using the same command is unable to receive information as they are not permitted in the sudoers file.
![Pasted image 20261120192648](assets/Pasted%20image%2020261120192648.png)


The user Kali, as someone in the sudoers file is able to use the same command.
![Pasted image 20261120192840](assets/Pasted%20image%2020261120192840.png)

I thought I was done, however using visudo -c it showed that the permissions for the  new sudoer file was incorrectly assigned. I changed the permissions to 440 and visudo -c properly returned that it was parsed correctly
![Pasted image 20261120194045](assets/Pasted%20image%2020261120194045.png)

# Checkpoint 5

Setting up the port forwarding on Kali VM so a connection is actually possible
![Pasted image 20261120213422](assets/Pasted%20image%2020261120213422.png)

Establishing Windows SSH key
![Pasted image 20261120215714](assets/Pasted%20image%2020261120215714.png)

Since Alice as a newly established user didn't have a section to input authorized keys for ssh. I created one as the Admin and set Alice as the owner. 
![Pasted image 20261120221147](assets/Pasted%20image%2020261120221147.png)

I realized authorized keys should not have been a folder so I removed it and created it as a file
![Pasted image 20261120221607](assets/Pasted%20image%2020261120221607.png)
![Pasted image 20261120221631](assets/Pasted%20image%2020261120221631.png)

Windows machine ssh public key was placed in authorized_keys
![Pasted image 20261120221759](assets/Pasted%20image%2020261120221759.png)


Testing SSH via authorized key. Works well, passphrase is for safeguarding windows machine's private key
![Pasted image 20261120222225](assets/Pasted%20image%2020261120222225.png)

Harden SSH by going into `sshd_config` and putting PubKey Authentication to yes, PasswordAuthentication no, PermitRootLogin no, KbdInteractiveAuthentication no
![Pasted image 20261120223143](assets/Pasted%20image%2020261120223143.png)

After editing, made sure the desired outputs were there and then reloaded ssh
![Pasted image 20261120223428](assets/Pasted%20image%2020261120223428.png)

When attempting to SSH into Alice via my windows machine, only the connection via the public key worked. Password authentication into Alice and ssh authentication into Root was denied
![Pasted image 20261120223904](assets/Pasted%20image%2020261120223904.png)

However, to further harden SSH, we want to make it so that noone except Machines authorized to ssh through alice are allowed to SSH in general. For that we make `AllowUsers alice` in the `sshd_config` file.
![Pasted image 20261120224735](assets/Pasted%20image%2020261120224735.png)

![Pasted image 20261120224920](assets/Pasted%20image%2020261120224920.png)

Running tests to ensure nobody can ssh except via Alice.
![Pasted image 20261120225107](assets/Pasted%20image%2020261120225107.png)


Read Journalctl logs based around ssh to view activity related to attempting ssh connections through bob, root and alice. Bob and Root connections disallowed due to not being in the allowedusers list, and connections accepted through alice due to allowusers and public key. 
![Pasted image 20261120225412](assets/Pasted%20image%2020261120225412.png)

# Checkpoint 6

Attempted Log ins using Bob, a password authentication with alice, and then a successful authentication in alice using the public key
![Pasted image 20261121000512](assets/Pasted%20image%2020261121000512.png)

Logs showing dates of attempted connections with their ports

![Pasted image 20261121000354](assets/Pasted%20image%2020261121000354.png)
