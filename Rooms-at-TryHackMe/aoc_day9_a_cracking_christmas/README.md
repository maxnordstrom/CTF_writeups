# Passwords - A Cracking Christmas

https://tryhackme.com/room/attacks-on-ecrypted-files-aoc2025-asdfghj123

## Introduction

## Initial Recon
- Sökte lite på måfå eftersom vi inte hade så mycket att gå på
- När jag kollade `sudo -l` på vår användare **ubuntu** stod det `ALL:ALL:ALL`
- Körde `sudo su` för att byta till root
- Jag kollade i `/etc/shadow` och tänkte att vi kunde knäcka lösenordet till root för att eventuellt få en ledtråd
- onind00 hittade däremot en spännande fil i hemkatalogen för **ubuntu**