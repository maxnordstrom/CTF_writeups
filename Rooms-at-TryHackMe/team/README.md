# Challenge: Team

https://tryhackme.com/room/teamcw

![Screenshot](img/Pasted%20image%2020251110121657.png)

## Introduction

A "simple" boot2root challenge. I tackled the task with [onind00](https://tryhackme.com/p/onind00) and [HENZU](https://tryhackme.com/p/HENZU) - thanks for the team spirit and great insights!

## Initial Recon

We started by running nmap, which showed that port 21 (ftp), 22 (ssh) and 80 (http) are open.

![Screenshot](img/Pasted%20image%2020251110121817.png)

I then went to the website to see what I could find, `http://10.10.5.55`. However, we only reached the template page for Apache2 Ubuntu:

![Screenshot](img/Pasted%20image%2020251110122514.png)

A couple of runs with **Gobuster** and **ffuf** gave nothing more than a 403 on `/server-status` which is standard.

## Digging Deeper

Upon closer inspection, the website's title element said "If it works, add **team.thm** to your hosts file". I first thought it wouldn't give us anything more, but would only be a way for us to go to *team.thm* instead of providing the entire IP address. But when I added the IP and name to `/etc/hosts` and went to `http://team.thm`, I immediately noticed that I reached a nice website.

Time to start **Gobuster** again and also look more closely at the website's code.

My friend did a run with **feroxbuster** and got a lot of interesting hits, including `/scripts/script.txt` and `/scripts/script.old`

![Screenshot](img/Pasted%20image%2020251110131816.png)

In `script.old` we found credentials for FTP, more specifically a large base64-encoded blob, but when it was decoded we could read `ftpuser:T3@m$h@r3`

## FTP

When we logged in via FTP we could find a text file in `/workshare/New_site.txt` that revealed that someone is developing a page for the team:

```
Dale
I have started coding a new website in PHP for the team to use, this is currently under development. It can be found at ".dev" within our domain.

Also as per the team policy please make a copy of your "id_rsa" and place this in the relevent config file.

Gyles
```
## Let's add a Subdomain

We added `dev.team.thm` to the hosts file and took a closer look. It really is a work-in-progress, but it appears to be the beginning of some kind of **team share** or **shared folder**.

![Screenshot](img/Pasted%20image%2020251110131432.png)

![Screenshot](img/Pasted%20image%2020251110134713.png)

Not much to the world right now, but could it be that we can upload things to this share to then exploit some kind of vulnerability? Maybe upload a webshell?

## Reading files in the browser

No webshell, but we got a "parameter-thing-in-the-browser". Could we exploit some kind of Remote/Local File Inclusion - reading files from the server directly in the browser? My friend tested to see if it worked, and indeed. We could read `/ect/passwd`, but also `/etc/ssh/sshd_config`

![Screenshot](img/Pasted%20image%2020251110152938.png)

It wasn't entirely easy to interpret, but at the bottom we could see the private key for the user `Dale` which begins at the end of the screenshot above. This is not a normal location for private keys, but for some reason this particular company had it as policy... Bad for them, good for us! However, the key needed to be edited a bit before I could use it.

When I copied it from the browser it was written on a single line and interspersed with several `#` and spaces, presumably because each line of the key was commented out. I took a look at what a regular OpenSSH key looked like (memory is good but short) and edited the one I found on the server with a simple find & replace and a simple regex.

![Screenshot](img/Pasted%20image%2020251110164543.png)

The private key for `Dale` is thus:

```
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```

I saved it to the file `key.pem`, ran `chmod 600 key.pem`, after which I could ssh to the server through `ssh -i key.pem dale@10.10.x.x` and didn't need to provide any password.

The first flag should be called `user.txt` and presumably it would be in the same folder where we landed when we logged in with ssh as dale, which was also the case.

<details>
  <summary><b>Click to see the first flag</b></summary>

  `THM{6Y0TXHz7c2d}`
</details>

## Privilege Escalation

>After having tested like a thousand different ways to get root privileges we could finally boil down the process to the following. Follow along!

The next flag is called `root.txt` and is presumably in `/root`. The task is thus to escalate privileges to be able to access it, because the user `dale` is not allowed to do that.

After running `sudo -l` as **dale** we see that one can run a script as dale where **gyles** is owner, and the script looks like this:

```bash 
#!/bin/bash 
printf "Reading stats.\n" 
sleep 1
printf "Reading stats..\n" sleep 1 read -p "Enter name of person backing up the data: " name
echo $name >> /var/stats/stats.txt
read -p "Enter 'date' to timestamp the file: " error 
printf "The Date is " 
$error 2>/dev/null 
date_save=$(date "+%F-%H-%M") 
cp /var/stats/stats.txt /var/stats/stats-$date_save.bak 
printf "Stats have been backed up\n"
```

The script has a vulnerability on the lines:

```bash
read -p "Enter 'date' to timestamp the file: " error
... 
$error 2>/dev/null
```

The script will be able to run whatever one enters there. The approach thus becomes:
- at the first prompt you write anything
- at the second you write `/bin/bash` and then get a shell as **gyles**

Now we have a shell as **gyles** and time to search through the system. `ls -la /home/gyles` gives many interesting hits, including `.bash_history`.

`/home/gyles/.bash_history` contains a lot of commands, but the interesting thing is that the user writes to `/opt/admin_stuff/script.sh`

Upon closer inspection the script contains:

```shell
#!/bin/bash 
#I have set a cronjob to run this script every minute

dev_site="/usr/local/sbin/dev_backup.sh" main_site="/usr/local/bin/main_backup.sh" 
#Back ups the sites locally 
$main_site 
$dev_site
```

As the comment says, a cronjob runs that backs up the website. If **gyles** has write permission on any of the scripts then I can edit it and either read the flag in `/root` or spawn a root shell.

When I run `id` and `groups` as **gyles** I see that the user belongs to the groups **lxd**, **editors** and **admin**.

I ran `ls -la /usr/local/sbin/dev_backup.sh` but the owner and group are **root**. After running the same command on `/usr/local/bin/main_backup.sh`, the owner is **root**, but the file belongs to the group **admin** - gyles can edit the script!

I commented out the lines in the script and instead added:

```shell
echo "cp /bin/bash /tmp/rootbash" >> /usr/local/bin/main_backup.sh 
echo "chmod +s /tmp/rootbash" >> /usr/local/bin/main_backup.sh
```

When I saw that the file **rootbash** was created I ran `/tmp/rootbash -p`, then `whoami` and got the following answer:

![Screenshot](img/Pasted%20image%2020251110232927.png)

After that I made sure that the flag was in `/root` through `ls -la /root`, then `cat /root/root.txt`. 

<details>
  <summary><b>Click to see the second flag</b></summary>
  
  `THM{fhqbznavfonq}`
</details>

## Final Words

Once again a "beginner friendly" challenge took an incredibly long time, but I learned many things along the way, including:
- You can find so much more if you add a domain name in `/etc/hosts` than just looking at the IP address.
- Always test if a website is vulnerable to File Inclusion/Path Traversal if you get the chance.
- How one can use an OpenSSH Private Key to SSH in without providing a password.
- That a simple script can contain vulnerabilities that can spawn a shell for another user.
- How I can edit a script that runs as a cronjob and spawn a root shell.

Thanks for reading and happy hacking! ✨

> 11 November 2025. Original text and markdown formatting by me. Translation by AI.