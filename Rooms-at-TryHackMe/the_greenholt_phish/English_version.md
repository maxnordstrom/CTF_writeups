# The Greenholt Phish

https://tryhackme.com/room/phishingemails5fgjlzxc

![Pasted image 20251124195438.png](img/20251124195438.png)

(English writeup found [here](English_version).)

## Introduction

This is the first time I'm taking on a blue challenge on TryHackMe. The task involves analyzing a suspected phishing email that has been sent to a Sales Executive at Greenholt PLC. Since I have some experience investigating PCAP files from FRA's challenges, I'm thinking it should go pretty smoothly. Searching through email headers should be about the same.

The task consists of 12 questions, all connected to the current email. (To be safe, I have defanged all links and email addresses from the task)

## Question 1

What is the Transfer Reference Number listed in the email's subject? `09674321`

## Question 2

Who is the email from? `Mr. James Jackson`

## Question 3

What is his email address? `info[@]mutawamarine[.]com`

## Question 4

What email address will receive a reply to this email? `info[.]mutawamarine[@]mail[.]com`

## Question 5

What is the Originating IP? Time to check headers.

![Pasted image 20251124233426.png](img/20251124233426.png)

The answer is `192[.]119[.]71[.]157`

## Question 6

Who is the owner of the Originating IP?

I looked it up on IPinfo

![Pasted image 20251124233756.png](img/20251124233756.png)

The answer is `Hostwinds LLC`

## Question 7
What is the SPF record for the Return-Path domain?

I looked this up on [dmarcian.com](img/https://dmarcian.com)

![Pasted image 20251124234938.png](img/20251124234938.png)

So the answer is `v=spf1 include:spf.protection.outlook.com -all`

## Question 8

What is the DMARC record for the Return-Path domain?

I checked this on the same site

![Pasted image 20251124235204.png](img/20251124235204.png)

The answer is `v=DMARC1; p=quarantine; fo=1`

## Question 9

What is the name of the attachment?

![Pasted image 20251124235321.png](img/20251124235321.png)

The answer is `SWT_#09674321_PDF.CAB`

## Question 10

What is the SHA256 hash of the file attachment?

To answer that, I needed to download the attached file, but obviously without opening it. I used `sha256sum` in the terminal and got the hash `2e91c533615a9bb8929ac4bb76707b2444597ce063d84a4b33525e25074fff3f`

## Question 11

What is the attachments file size?

I ran a simple `ls -l` in the terminal to get the file size, but the task wanted me to compare the hash against a database. I turned to **Talos File Reputation**. That worked fine, but I got the same file size.

![Pasted image 20251125000451.png](img/20251125000451.png)

It turned out that the task wasn't satisfied with what **Talos** came up with, so I looked up the hash on **VirusTotal** which gave me the answer `400.26 KB`

![Pasted image 20251125003259.png](img/20251125003259.png)

## Question 12

What is the actual file extension of the attachment?

When I had downloaded it, it looked like a zip file. **Talos File Reputation** revealed that it was a rar file. So the answer is `rar`

## Conclusion

A nice challenge that made me more comfortable searching for information among email headers and also using some online tools to check up on a suspected file. Not nearly as challenging as the red rooms I've previously completed, but I look forward to encountering more blue challenges in the future! I suspect it can get much trickier.

> 25 November 2025. Original text and markdown formatting by me. Translation by AI.