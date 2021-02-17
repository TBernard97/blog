---
layout: post
title:  "CMESS Writeup"
date:   2021-02-16 21:55:10 -0400
categories: tryhackme
---

## What does this box teach?

This machine illustrates the importance of locking down subdomains, updating admin CMS applications, and ensuring that security protocols are properly utilized.

## Initial Enumeration

As one does in CTF land we start off with an nmap scan:

{% highlight bash %}
lazy-scan -h 10.10.175.13
{% endhighlight %}

The tool utilized is a wrapper script I built called lazy-scan. For more info on it click [here](https://github.com/TBernard97/lazy-scan-script). It appears ports 22 and 80 are open. 14000 is filtered and is not of immediate interest.

![nmap_scan](/blog/assets/CMESS/nmap_scan.png)

After some initial enumeration I decide to take a look at the hint and perform some subdomain enumeration with wfuzz.

{% highlight bash %}
wfuzz -c -f [OUTPUT FILE NAME] -w [path to wordlist] 
-u 'http://cmess.thm' -H "Host: FUZZ.cmess.thm"
{% endhighlight %}

![wfuzz_scan](/blog/assets/CMESS/wfuzz_scan.png)

So what is this magic we just typed? Wfuzz is a python module that allows a user to perform fuzzing operations by simply designating said parameter with the "FUZZ" keyword.

- The -c flag instructs wfuzz to use colors. 
- -f is the filename to output. 
- -w tells wfuzz which file to use for a wordlist. 
- -u designates the target url. 
- -H designates target headers.

Upon initial use it seems that 200 responses are being generated for directories that do not exist. A filter should be used for the word count.

{% highlight bash %}
wfuzz -c -f [OUTPUT FILE NAME] -w [path to wordlist] 
-u 'http://cmess.thm' -H "Host: FUZZ.cmess.thm" --hw 290
{% endhighlight %}

![wfuzz_scan_filter](/blog/assets/CMESS/wfuzz_scan_filter.png)
 
With that command it is dicovered that there is a
"dev" subdomain present. After adding that subdomain to /etc/hosts, visting that subdomain gets us some credentials.

andre@cmess.thm:KPFTN_f2yxe%

![dev_subdomain](/blog/assets/CMESS/dev_subdomain.png)

Either through gobuster or just plain guessing an admin panel can be found off the /admin route.

![admin_panel](/blog/assets/CMESS/admin_panel.png)

## Exploiting the application

After initially logging in it is discovered that the CMS version is Gila CMS version 1.10.9.

![gila_version](/blog/assets/CMESS/gila_version.png)

Searching ExploitDB it is discovered that there is an LFI vulnerability on the site. [Exploit DB Link.](https://www.exploit-db.com/exploits/47407)

![Exploit_DB](/blog/assets/CMESS/exploit_db.png)
![LFI_Discovery](/blog/assets/CMESS/lfi_discovery.png)

This LFI is great! It also allows the attacker to edit files if they wanted to.

![robots_allow](/blog/assets/CMESS/robots_allow.png)
![robots_modified](/blog/assets/CMESS/robots_modified.png)

Taking a peak at the config.php we find more credentials: root:r0otus3rpassw0rd

![config_creds](/blog/assets/CMESS/config_creds.png)

Unfortunately these credentials are not for the actual root user on the machine but rather the MySQL database. This is good for when we get access to the machine however.

## .htaccess flaws

Taking a look at the conversation on dev.cmess.thm it seems that there is a conversation talking about a misconfigured .htaccess file. If the reader of this post does not know, .htaccess is a method of locking down directories to specific authenticated users, or disallowing all users completely. On this machine an example of a properly configured .htaccess file can be seen in /tmp.

![tmp_htaccess](/blog/assets/CMESS/tmp_htaccess.png)

After some further snooping it seems that /assets is poorly configured allowing access to any file under it from the public.

![assets_htaccess](/blog/assets/CMESS/assets_htaccess.png)

With this knowledge we are ready to test for file upload vulnerabilities.

## Exploiting file upload vulnerabilities

Before uploading a shell it would be wise to test a simple case. First lets see what /assets looks like when requested from the browser

![assets_before_test](/blog/assets/CMESS/assets_before_test.png)

It seems that assets does not list files under its directory. This means that the file has to be explicitly stated in the URL. 

First lets create a file with some simple text under the directory and then request it.

![assets_test_upload](/blog/assets/CMESS/assets_test_upload.png)

![assets_after_test](/blog/assets/CMESS/assets_after_test.png)

Excellent the method works. Now time to upload the shell.

![php_upload](/blog/assets/CMESS/php_upload.png)

![initial_shell](/blog/assets/CMESS/initial_shell.png)

It is time to escalate to user.

## Escalating to user

### MySQL potential rabbit hole

This machine is obviously using MySQL on its backend. So lets try the credentials we got from the config.php on the LFI.

![mysql_login](/blog/assets/CMESS/mysql_login.png)

We now have root access to the database. Lets see which databases are available.

![mysql_show_db](/blog/assets/CMESS/mysql_show_db.png)

Gila looks promising. Lets use it and show the tables.

![mysql_use_gila](/blog/assets/CMESS/mysql_use_gila.png)

Let's select all from user.

![mysql_select_user](/blog/assets/CMESS/mysql_select_user.png)

Awesome we got a bcrypt hash! Let's try to crack it in the background while we enumerate further.

![hashcat](/blog/assets/CMESS/hashcat.png)

So what is wrong with this? Well after thinking about it further two things. First we did not take a breadth first approach so we may have ended up looking into things that had nothing. The proper approach would have been to take note of the MySQL database, enumerate the machine, and comeback to it later after complete enumeration.

Furthermore, what if this hash is for the login to the CMS? Then we just wasted time cracking a hash we already have. This is why it is so important to use good methodologies when testing.

## Getting Andre's Creds

After looking through /tmp and /opt I found a .password.bak file under /opt with terrible permissions.

![discovering_password_bak](/blog/assets/CMESS/discovering_password_bak.png)

![reading_password_bak](/blog/assets/CMESS/reading_password_bak.png)

Now it is time to escalate to root.

## Escalate to root

After going through the checks for privesc something interesting is found in the /etc/crontab.

![crontab](/blog/assets/CMESS/crontab.png)

This is a classic wildcard vulnerability. The tar command is compressing everything under /home/andre/backup to /tmp/andre_backup.tar.gz as root with a wildcard at the end. Why is this bad? Well this means that special flags called "checkpoints" could be used to root the machine. In order to exploit this someone simply needs to create files with the names of these flags under the directory being compressed with an action that leads to a shell. 

{% highlight bash %}
cd /home/andre/backup
echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' > /home/andre/backup/runme.sh
touch /home/andre/backup/--checkpoint=1
touch /home/andre/backup/--checkpoint-action=exec=sh\ runme.sh
{% endhighlight %}

![checkpoint_privesc](/blog/assets/CMESS/checkpoint_privesc.png)

Wait about a minute or so and execute /tmp/bash and enjoy root!

![root](/blog/assets/CMESS/root.png)

# References
[To learn more about htaccess](https://www.whoishostingthis.com/resources/htaccess/)

[To learn more about tar checkpoint vulnerabilities](https://gtfobins.github.io/gtfobins/tar/)

[To learn more about tar checkpoints in general](https://www.gnu.org/software/tar/manual/html_section/tar_26.html)

[To learn more about Wfuzz](https://github.com/xmendez/wfuzz)

[To learn more about my "lazyscan" script](https://github.com/TBernard97/lazy-scan-script)











