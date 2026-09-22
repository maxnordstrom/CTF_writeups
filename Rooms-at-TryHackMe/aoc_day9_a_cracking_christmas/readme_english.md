> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# Passwords - A Cracking Christmas
### Advent of Cyber 2025, Day 9

https://tryhackme.com/room/attacks-on-ecrypted-files-aoc2025-asdfghj123

![Screenshot](img/Pasted%20image%2020251210165433.png)

## Intro
The room is about cracking passwords, and you get some basic info on how it's usually done. You get an explanation that it's rarely about "cracking" the passwords, but rather guessing correctly with the help of different wordlists. Running a full brute force and testing *every* conceivable combination is possible with the right computing power and time, but it's rarely worth it for the average pentester.

The tasks in question are about cracking the passwords to two password-protected files - a pdf and a zip. I'll briefly describe how I went about it, but that wasn't actually the reason I came to this room today. At the end of the task you get a hint that on this machine you can find the key to unlock Sidequest 2!

I'll briefly describe how I solved the tasks and then go into more depth on how I (and my friends) went about finding the key to Sidequest 2.

## Tasks
The tasks were about cracking the passwords to a pdf and a zip file.

### PDF
The pdf file was called **flag.pdf** and I used the program **pdfcrack**. This by running the command `pdfcrack -f flag.pdf -w /usr/share/wordlists/rockyou.txt`
The program found the password immediately and printed it in the terminal.

<details>
  <summary><b>Click here to see the password</b></summary>

  `naughtylist`
</details>

Since I used ssh to access the server I couldn't open the pdf file the normal way, so I had to convert it to text with the program **pdftotext**. This by the command `pdftotext -opw secretPassword flag.pdf` which instead gave me the file **flag.txt**. `cat flag.txt` gave me the flag!

<details>
  <summary><b>Click here to see the flag</b></summary>

  `THM{Cr4ck1ng_PDFs_1s_34$y}`
</details>

### ZIP
The zip file was called **flag.zip** and I used John the Ripper to crack the password. For john to be able to work with the file you first have to run `zip2john` to get a suitable password hash. This via `zip2john flag.zip > zip.hash`. That gave me the file **zip.hash** which john understands. After that I ran `john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash`, whereupon john found the correct password and printed it in the terminal.

<details>
  <summary><b>Click here to see the password</b></summary>

  `winter4ever`
</details>

I tried unzipping with **unzip** and **gzip**, but I couldn't find any options for supplying a password. It did work with **7-Zip** though. This via the command `7z e flag.zip`. I then got a prompt about how I wanted to save the extracted file - I chose to not overwrite the file **flag.txt** that already existed in the directory, and instead give it a new name. After that I got a prompt to enter the password. Once that was done I had the file **flag_1.txt** which contained the flag.

<details>
  <summary><b>Click here to see the flag</b></summary>

  `THM{Cr4ck1n6_z1p$_1s_34$yyyy}`
</details>

## Hunting for the key to Sidequest 2
Now, finally time to hunt down the key to Sidequest 2!

- Searched around a bit at random since we didn't have much to go on
- When I checked `sudo -l` on our user **ubuntu** it said `ALL:ALL:ALL`
- Ran `sudo su` to switch to root
- I checked `/etc/shadow` and thought we could crack root's password to possibly get a hint
- onind00, however, found an interesting file in **ubuntu**'s home directory, namely `.Passwords.kdbx`

![Screenshot](img/Pasted%20image%2020251210170801.png)

- I didn't know what kind of file that was so I had to google it. Found out it's a database file for the password manager KeePass.
- Since the room was about cracking passwords I assumed we should crack that very file.
- After a search online I found out that **John the Ripper** can crack such files, as long as you first create a hash that john can understand. You do this with **keepass2john**.
- All of john's installation files were on **ubuntu**'s desktop, so I went to `/home/ubuntu/Desktop/john/run` where **keepass2john** was located.
- After that I ran `./keepass2john /home/ubuntu/.Passwords.kdbx > /home/ubuntu/keepass.hash` which, in short, runs the program and saves its output to the file **keepass.hash** in ubuntu's home directory
- After that it was time for john to do its magic. I navigated to the home directory and ran the command `john --wordlist=/usr/share/wordlists/rockyou.txt keepass.hash`
- It turned out, however, that I got an error message about some kind of race condition.

![Screenshot](img/Pasted%20image%2020251210172442.png)

- I figured it might be because I was **root**. I switched back to the user **ubuntu** and ran the same command, and then it worked.

<details>
  <summary><b>Click here to see the password</b></summary>

  ![Screenshot](img/Pasted%20image%2020251210172523.png)
</details>

- John showed off its best side again! I was a bit puzzled at first though, because what would I want with *a single* password? But it didn't take long before I understood I'd use it to open the database itself.
- A quick search told me you can open KeePass files with **keepassxc**, which as it happens was already on the machine.
- It didn't want to open in the terminal though - I need a display.

![Screenshot](img/Pasted%20image%2020251210172823.png)

- I downloaded `.Passwords.kdbx` to my Kali VM, installed **keepassxc** on it and ran the command to open the file. It opened the following window:

![Screenshot](img/Pasted%20image%2020251210173113.png)

- Once unlocked I could see a key but no interesting information. I clicked the **Advanced** tab and saw there was a png file.

![Screenshot](img/Pasted%20image%2020251210173305.png)

- I selected the file and clicked preview, and there I found the sought-after key to Sidequest 2

![Screenshot](img/Pasted%20image%2020251210173450.png)

- The key is in the image, but I'll keep it out of here for the thrill of it ;)

## Epilogue
I appreciated the room for letting me brush up on using John the Ripper. I like the tool, but I've always found it a bit tricky to use since you have to adapt the files so much for john to be able to read them.

But above all it was the subsequent hunt for the key to Sidequest 2 that was really exciting. There's such a great feeling when you search and search, and finally find a lead that takes you to the goal. I learned more about what KeePass databases look like, how you can crack them with john (provided a weak password is used...) and then open the file to read its contents. Great fun!
