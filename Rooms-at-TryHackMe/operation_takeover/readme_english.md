> *This translation has been made by Claude, check out the original writeup [here](README.md).*

![Screenshot](img/Pasted%20image%2020260410103247.png)

https://tryhackme.com/room/operationtakeover

![Screenshot](img/Pasted%20image%2020260410103308.png)

Exciting, a room without instructions! Let's see what we're up against.

## Initial recon

I turn straight to nmap to see what we have to work with. This time I'll scan *all* the ports :)

Found three open ports:

![Screenshot](img/Pasted%20image%2020260410105542.png)

Ran a more thorough scan on the three ports to find out what's running, via `nmap -sC -sV -p22,179,2623 $IP -v`, and got the following:

![Screenshot](img/Pasted%20image%2020260410111256.png)

The room is about hacking a router, so FRRouting on port 2623 is highly interesting. I'm also curious about what's hiding behind port 179, which only returns `tcpwrapped`.

## FRRouting v 10.0

Did a quick google to see if there are any CVEs for FRRouting version 10, and it looks like there are

![Screenshot](img/Pasted%20image%2020260410105848.png)

Found a couple of different ones, but they all seem to be about DoS, and we want access, not to take it down...

If I try connecting to the router with netcat I get a password prompt - so we can connect to it. Chances a default password is used? Unknown. Guess we'll have to try.

![Screenshot](img/Pasted%20image%2020260410111751.png)

A google search says FRRouting doesn't have a default password. Bummer. Fire up Hydra to chug away in the background? Yeah, let's go. Rockyou it is.

## Hydra

Started with the command `hydra -l "" -P /usr/share/wordlists/rockyou.txt telnet://$IP:2623`. 

FRRouting apparently generally uses no username, hence the empty string.

But now it feels like I started at the wrong end. I create a custom wordlist instead to try my luck there.

```
zebra
frr
frrouting
admin
password
router
cisco
bgpd
ospfd
root
toor
raspberry
dietpi
test
uploader
password
admin
administrator
marketing
12345678
1234
12345
qwerty
webadmin
webmaster
maintenance
techsupport
letmein
logon
Passw@rd
alpine
```

No luck there either though. Ran Rockyou for a while but it feels like a bit of a stretch to just rely on that. There must be a smarter way...

## BGP on port 179

That nmap only shows port 179 running bgp, but that it comes back as tcpwrapped, is apparently expected. The router only connects to IPs it has in its list.

Seems tricky to figure out which IPs would be in that list though, so I keep looking...

## CVE again

A new CVE was published recently (March 2026) that's actually about access control.

![Screenshot](img/Pasted%20image%2020260410120128.png)

I don't love the phrasing "The attack is considered to have high complexity" and "The exploitability is reported as difficult" :D

## Nmap again

Now I remembered to scan all the ports. Only the TCP ports though. There could be a leaky service, SNMP, that uses UDP.

Started `nmap -sU -p- $IP -v`. It took a tremendous amount of time so I stopped that one.

But since we're curious about SNMP on port 161 specifically, I can use the program `onesixtyone`.

I run this program with a wordlist from seclists to fuzz different names and got the following hit:

![Screenshot](img/Pasted%20image%2020260410121355.png)

Then I use `snmpwalk` to find out more info about the router. Maybe it leaks some good stuff.

`snmpwalk -v2c -c pr1v4t3 $IP | tee snmpwalk.txt` gives a ton of information. The question is what we should use.

It turned out that with snmp we can run commands on the server. Onind00 came up with a proof of concept by writing a file to the server and then reading it:

![Screenshot](img/Pasted%20image%2020260410123618.png)

After that we could write commands to the server with `snmpset` which we then ran with `snmpwalk`

![Screenshot](img/Pasted%20image%2020260410125107.png)

![Screenshot](img/Pasted%20image%2020260410125131.png)

That last file there looks promising :)

![Screenshot](img/Pasted%20image%2020260410125158.png)

We got the flag!

![Screenshot](img/Pasted%20image%2020260410125214.png)

## Final words

This was completely new ground for me. Ran the good old reconnaissance from the start, which did lay the foundation for finding the open ports and services running, but I had no idea about the actual attack chain itself. So, thanks onind00, and thanks AI :D
