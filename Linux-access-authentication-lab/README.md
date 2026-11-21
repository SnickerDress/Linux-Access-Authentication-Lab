## Project Overview:
This lab demonstrates the configuration and verification of Linux access controls and SSH authentication in a Kali Linux VM. I created standard user accounts and integrated shared and private directories using ownership, group membership, permissions, and setgid. I also investigated how umask affects newly created files and directories.

I applied least privilege through a sudo policy permitting one exact administrative command. I then established a passphrase-protected ED25519 key authentication from Windows, restricted SSH access to Alice, and disabled password-based authentication and direct root login.

Finally, I generated controlled login attempts and analyzed SSH logs to distinguish account-policy denials, invalid usernames, and successful public-key authentication. The six checkpoints document configuration decisions, troubleshooting, test results, and supporting evidence.


## Lab Details
- **Objective:** Configure and validate least-privilege access controls, harden SSH authentication, and investigate controlled login attempts in a multi-user Linux environment.
- **Environment:** Kali Linux running in Oracle VirtualBox, with a Windows OpenSSH client. VirtualBox NAT forwarded Windows loopback port `2222` to Kali’s SSH port `22`. Accounts included the administrator `kali` and standard users `alice` and `bob`.
- **Configuration:** Implemented group-based and owner-only directory access, tested umask settings, granted Alice one specific sudo command, and restricted SSH to Alice using public-key authentication. Disabled SSH password authentication, keyboard-interactive authentication, and direct root login.
- **Validation:** Tested permitted and denied actions under different accounts, validated sudoers and SSH configurations, and compared expected behavior with screenshots and authentication logs.
- **Troubleshooting:** Corrected ownership and permission errors, created missing home directories, resolved NAT connectivity through port forwarding, and corrected the structure of SSH authorization files.
- **Findings and limitations:** The tested controls enforced the intended access policy, and no successful unauthorized authentication appeared in the reviewed log window. All attempts were controlled simulations. VirtualBox NAT presented connections as originating from `10.0.2.2`, and SSH keys authorized key holders rather than specific physical machines.
## Checklist 
- [x] **Record your completed access-control work.**  
    Save evidence of user memberships, directory ownership and permissions, Alice’s successful access, and Bob’s denied access. Explain why each result occurred.
    
- [x] **Test private files and shared-file permissions.**  
    Create a private area for one user and verify the other user cannot access it. Learn how _umask_ sets default permissions and how that differs from setgid’s group inheritance.
    
- [x] **Configure limited administrative access.**  
    Give one lab user permission to perform one narrowly defined administrative task. Verify that task succeeds and unrelated administrative actions remain denied.
    
- [x] **Prepare and harden SSH.**  
    Confirm the VM’s network mode and decide where you’ll connect from. Establish key-based login, restrict permitted accounts, and disable root login and password authentication after verifying key access. Keep VM console access available during testing.
    
- [x] **Generate and investigate authentication events.**  
    Perform controlled successful and failed logins against your VM. Identify usernames, timestamps, source addresses, and outcomes in authentication logs. Investigate repeated failures and denied privilege attempts.
    
- [x] **Finish portfolio report.**  
    Include your configuration choices, a test-results table, selected evidence, and a short investigation explaining what happened and how you reached that conclusion.
# Checkpoint 1 — Group-Based Access Control

The directory `/srv/project` is owned by the user `root` and the group `projectteam`. Its permissions are `2770`:

- **`2` — setgid:** New files and subdirectories inherit the group `projectteam`. New subdirectories also inherit the setgid bit.
- **First `7` — owner:** `root` has read, write, and execute permissions.
- **Second `7` — group:** Members of `projectteam` have read, write, and execute permissions.
- **`0` — others:** Users outside the owner and group categories receive no permissions.

For a directory, **read** allows listing entry names, **write** allows modifying directory entries, and **execute** allows entering the directory and accessing items within it. Creating files requires both write and execute permissions.

Alice belongs to `projectteam`, so she successfully created a file and listed the directory’s contents. Her new file belonged to `alice:projectteam`, confirming that group inheritance worked.

Bob does not belong to `projectteam`. His attempts to list the directory and create a file returned **Permission denied**.

These tests demonstrate **least-privilege access control**: Alice received access through her assigned group, while Bob was denied access. Neither account needed administrative privileges.
  

# Checkpoint 2 - Owner Only Directory Access

Created `/srv/alice-private/` and assigned Alice as its owner. Set the directory’s permissions to `700`, giving Alice read, write, and execute permissions while granting no permissions to the directory’s group or other users.

Verified that Alice could create a file and list the directory’s contents. Bob could neither create a file nor list its contents, and the unelevated `kali` account also could not list the directory. These attempts returned **Permission denied**.

Root successfully listed the directory because root can bypass ordinary Linux file permissions. This demonstrates that owner-only permissions restrict standard users but do not prevent administrative access.

# Checkpoint 3 - Umask Default permissions
`umask` is a shell setting that removes permissions from those requested when programs create files and directories.

My shell’s initial umask was `002`. With normal creation defaults, this produces files with permissions `664`: the owner and group can read and write, while others can only read. Directories receive `775`: the owner and group have read, write, and execute permissions, while others have read and execute permissions.

I tested two stricter masks:

|Umask|New file permissions|New directory permissions|
|---|---|---|
|`027`|`640` — owner read/write; group read; others none|`750` — owner full access; group read/execute; others none|
|`077`|`600` — owner read/write only|`700` — owner full access only|

After restoring the shell’s umask to `002`, I verified that the existing files and directories retained their permissions. This demonstrates that changing a shell’s umask affects future creation by that shell and programs inheriting its mask, without changing existing permissions.

# Checkpoint  4 - limited admin access
The goal was to grant a standard user like Alice, one specific administrative capability while enforcing least privilege. In a real environment, an operator may need elevated access for a defined monitoring task without receiving unrestricted administrative privileges.

I configured Alice to view the SSH service’s status as root using `sudo systemctl status ssh --no-pager`. Although standard users can normally view basic service status, elevated access can reveal additional information, including protected log entries.

Using `visudo`, I created `/etc/sudoers.d/alice-ssh-status` with this rule:

```
alice ALL=(root) /usr/bin/systemctl status ssh --no-pager
```

Testing confirmed that Alice could execute the exact permitted command through sudo. Omitting `--no-pager` resulted in denial, demonstrating that the rule restricts command arguments. Alice’s sudo privilege listing showed only the intended command. Bob’s attempt to execute the same command through sudo was denied, while the administrative `kali` account succeeded.

During validation, `visudo -c` identified incorrect permissions on the policy file. I corrected its permissions to `0440` and verified its ownership was `root:root`. A subsequent validation reported that all sudoers files parsed successfully.

This checkpoint demonstrated narrowly scoped administrative access, argument-based command restrictions, and validation of sudo policy files.

# Checkpoint 5 - Hardening SSH
The goal was to restrict remote access to the project environment and reduce exposure to password guessing, password spraying, and stolen account passwords. The intended policy allowed only Alice to connect through SSH, required public-key authentication, and prohibited direct root SSH login.

I created `/home/alice/.ssh` with ownership `alice:alice` and permissions `700`. Within it, I configured the `authorized_keys` file with ownership `alice:alice` and permissions `600`.

On the Windows client, I generated an ED25519 key pair using `ssh-keygen -t ed25519` and protected the private key with a passphrase. I copied only the public key into Alice’s `authorized_keys` file. The private key remained on Windows.

I verified a successful SSH connection using the key through VirtualBox’s forwarding rule from Windows port `2222` to Kali port `22`. The private-key passphrase unlocked the key locally; it was not Alice’s account password.

I configured the SSH server with:

```
AllowUsers alice
PasswordAuthentication no
PermitRootLogin no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

I validated the configuration, checked its effective values, and reloaded SSH. Fresh connection tests confirmed that Alice could authenticate with her key, while password-only authentication was denied.

Server logs explicitly showed Bob and root being rejected because they were not listed in `AllowUsers`. They also recorded successful public-key authentication and session creation for Alice. The source address appeared as `10.0.2.2` because the connections passed through VirtualBox NAT.

These controlled tests demonstrated enforcement of the intended SSH policy. Key authentication restricts access to holders of an authorized private key; it does not bind access to a specific physical computer.

# Checkpoint 6 - Authentication Log Investigation
Using `sudo journalctl -u ssh --since "15 minutes ago" --no-pager -o short-iso` logs were shown of ssh requests.

**23:52:44–23:54:54 EST**

Investigation timeline:

|Time (EST)|Account|Source port|Outcome and evidence|
|---|---|--:|---|
|23:53:21|`bob`|49840|Denied: not listed in `AllowUsers`|
|23:53:25|`bob`|49843|Denied: not listed in `AllowUsers`|
|23:53:28|`bob`|49845|Denied: not listed in `AllowUsers`|
|23:53:56|`randomuser`|49850|Rejected as an invalid username|
|23:54:11|`alice`|49854|Connection reset before authentication completed|
|23:54:24|`alice`|49855|ED25519 public key accepted; session opened|

All connections appeared from **`10.0.2.2`**, the VirtualBox NAT address.

- Three Bob connections were rejected within **seven seconds** a repeated-access pattern that would normally be worth investigating within a real work environment.
- Alice’s connection reset corresponds to the planned password-only test. However that message alone doesn't confirm the specific authentication method.
- The only successful authentication shown belongs to Alice using an authorized key.
- **No successful unauthorized login is shown within this window.** 

Images for all the checkpoints: [Supporting evidence](Evidence.md)
 