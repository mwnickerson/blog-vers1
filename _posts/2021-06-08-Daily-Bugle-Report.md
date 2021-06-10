---
layout: post
title: Daily Bugle writeup
date: June 8, 2021
categories: Vuln Report
---
## A vuln report for the TryHackMe Daily Bugle Room

# High Level Summary

<p1>Today I completed the Daily Bugle room as part of the Offensive Penetesting Learning Path on TryHackMe.com
I was given an ip address for the box after activating the box.
My goal was to find and collect both user and root flags for this box as well as answer a few questions along the way.
Using SQLi to exploit an out of date version of the CMS, I am able to gain access to the CMS admin panel
and gain an initial foothold in the machine with remote code. Once inside I was able to gain an initial access.
I was able to escalate my privileges from web server to user through clear text password in system files,
and finally from user to root through misconfigured user permissions.</p1>

# Methodologies

<p2>I will go into greater deeper and provide instructions for reproducing my attack</p2>

## Reconaissance

<p3>I started my attack with a "lite nmap" and enumeration nmap scan.
**nmap lite**
![Nmap Lite Scan](/assets/images/dailybugle/mnmaplite.png)
**nmap enum**
[Namp Enum Scan](/assets/images/dailybugle/nmapenum.png)
There are three ports open: 22, 80, 3306
</p3>
**browser view**
![broswer view of website](/assets/images/browserview.png)
*this reveals the answer to the first question!*
<p4>I see the webserver is open on 80 and there is an sql server running on 3306.
That most likely means I am going to be going up against a web server.
In order to get a better idea of what the website is,
 I visit it in my browser as well as begin a nikto scan in the background.</p4>

**nikto**
![nikto scan](assets/images/nikto.png)
<p5>The nikto scan comes completes.
It  enumerates some directories that are located within the web server as well.
The one that catches my eye is the /administrator/.
I navigate to it and find a Joomla admin panel.
I have no credentials, but may be able to find more upon further enumeration.
The nikto scan also returns that the webserver is running a CMS called Joomla.
However it does not return the version running.</p5>

<p6>I am not the most familiar with Joomla, but I googled it and learned a little bit more about it.
I learned that it is a Free and Open Source Content Management software for blogs.
it has had many vulns in the past, I just need to find out which version is running in order to choose the correct exploit.</p6>

<p7>As I researched further I came across the below site where it went further into depth on how to properly enumerate the Joomla Version.
https://hackertarget.com/attacking-enumerating-joomla/#joomla-core-version

`http://<targeturl>/administrator/manifests/files/joomla.xml`

I use the url location of an xml file that my research said would contain the version and sure enough it did.
I find that I am going up against Joomla 3.7.0
*this is another answer to a question* </p7>


## CVE-2017-8917 and Sqlmap
<p8>Upon researching the version number I come across [CVE-2017-8917](https://www.exploit-db.com/exploits/42033).
This vuln allows me to use sql injection within the url.
 I can then take advantage of this SQLi to use sqlmap to start dumping the databases</p8>

`sqlmap -u "http://10.10.134.118/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p list[fullordering]`
![SqlMap Database Dump](/assets/images/dailybugle/sqlmapdbdump.png)

I dump the names of the databases within the sql server.
There are five databases, but the one that interests me is the "information_schema" table.
This is the database that contains the #__users table, and that will be where I find the credentials needed to gain access. </p8>

<p9>Upon researching in the [Joomla documentation](https://docs.joomla.org/Tables/users),
I found the names of all the columns in the #__users table and am going to run an sqlmap dump for these
in order to obtain any user information inside

`sqlmap -u "http://10.10.134.118/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent -D joomla -T '#_users' -C id,name,username,email,password,usertype,block,sendEmail,registerDate,lastvisitDate,activation,params --dump `
![sqlmap #__users dump](/assets/images/dailybugle/sqlmap__users.png)
I was able to dump the user info including the hash for the super user.</p9>

<p10>I am able to take the hash for the user and run it through hashcat and generate the user password.
I can then take these credentials and login to the admin panel I found earlier.</p10>
**hashcat**
[hashcat](/assets/images/dailybugle/hashcat.png)

## Admin Panel and Initial Shell
[Admin Panel Access](assets/images/dailybugle/admin-panel.png)
<p11>I am inside the admin panel and have found access to the editing the templates for the index page.
I erase the current template and replace it with a reverse shell script for php from [Pentest Monkey Github](https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php).
I open a listener on my machine and visit the index and my shell connects.</p11>
![initial shell](/assets/images/dailybugle/initiallllshell.png)

## Privilege Escalation
<p12>I am in as a service account for the apache web server.
Unfortunately this account is limited in what I can do.
I run the Linpeas script to enumerate and find a way to escalate my privileges up to root.
I navigate to the /tmp/ drive and open a simple python server on my attacking machine and download linpeas with the shell on my target.
Once its on the machine, I run it.</p12>
![linpeas](/assets/images/dailybugle/linpeas.png)
After running the linpeas enumeration script I found that there is a password stored in plaintext in a configuration file.
This password is associated with the user jjameson.
I use these to login to his account and work on further escalating my privileges on the machine.</p12>

## Root
![sudo L](/assets/images/dailybugle/sudol.png)
**sudo -l**
![gtfobinsyum](/assets/images/dailybugle/gtfobinsyum.png)
**script for installing plugin in yum for shell**
<p13>I find that I am able to use yum as a super user.
I enter the approproiate script to install a plugin on yum that allows me to run a bash shell as root.
and I have it rooted.</p13>
![root](/assets/images/dailybugle/root.png)
**rooted**
