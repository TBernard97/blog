---
layout: post
title:  "Pwning the BatCave Part 2"
date:   2020-11-14 20:33:10 -0400
categories: Labs
---

# SMB Signing Attacks

Now that we have confirmed the domain is vulnerable to LLMNR attacks it is time to leverage this vulnerability to gain even more access. When enabled, SMB signing mitigates replay attacks from occuring with related MITM attacks. However, this protocol is disabled by default on AD environments and also has the potential to cause a significant reduction of file transfer speeds on the environment when enabled. Because of this, SMB signing attacks can be a viable way to gain access to an environment. Let's get started.

## What is SMB signing?
SMB signing is a packet level protocol that provides a "signature" for the SMB protocol when being used. It allows devices to verify that they are authorized to use the protocol on the environment. Now what can happen when it is disabled? All kinds of bad stuff. You see this is supposed to be a mitigation to stop replay attacks from occuring. If a malicous user somehow gains access to the network, they could potentially leverage this vulnerability to replay SMB authentication.

## Checking for SMB signing
Nmap can be used to check to see whether or not SMB signing is enabled on a range of targets. 

{% highlight bash %}
nmap --script=smb2-security-mode.nse -p445 10.0.2.0/24
{% endhighlight %}

This command will display whether or not SMB signing is enabled and required. It should be noted that servers require message signing to be both enabled and required.


## Responder Configuration
Modification of the Responder.conf is necessary to to be able to perform the following attack. HTTP and SMB servers must be disabled in order to be able to relay packets.

![Responder.conf](/blog/assets/BatCave/Responder_config.png)

## SMB Relay Attacks
If the environment is vulnerable, SMB relay attacks are quite easy to pull off. The impacket toolkit has a script known as ntlmrelayx.py which allows an attacker to perform relay attacks for a wide range of protocols including SMB, LDAPS, HTTP and more. It is automatically built into the latest versions of Kali Linux.

If an admin is spoofed while attempting to authenticate via one of these protocols access can be obtained by relaying these credentials to a target device.

First run responder after the necessary configurations are made.

{% highlight bash %}
responder -I eth0 -rdwv
{% endhighlight %}

Next, run ntlmrelayx against a file containing target IP's with smb2support.

{% highlight bash %}
ntlmrelayx.py -tf targets.txt -smb2support
{% endhighlight %}

This will attempt to replay logins to the target machine. A shell will be gained if the account used is a local administrator.

# Pwning Alfred
![alred](/blog/assets/BatCave/alfred-1.jpg)

It appears that there is a second user, apennyworth, on the domain as well. 

However there is one key difference. The account seems to be a local administrator.

## Replay Attack

After a sucessful login our poisoner and relay script managed to dump the SAM of the target machine.

![SAM Dump](/blog/assets/BatCave/SAM-Dump-Alfred.png)

This clearly is not good. However it gets worse. It seems that ntlmrelayx opened up an interactive SMB shell on the target machine.


![shell-smb](/blog/assets/BatCave/shell-smb-alfred.png)

Using netcat we can access this on local port 11000

{% highlight bash %}
nc 127.0.0.1 11000
{% endhighlight %}

![shell-smb](/blog/assets/BatCave/shell-smb-alfred-nc.png)

After reading the help command output we see that shares can be listed via the shares command.


![shares](/blog/assets/BatCave/shares-list-alfred.png)

Interesting, what if we can write to this share?

## Getting writable shares with crackmapexec

[Crackmapexec](https://github.com/byt3bl33d3r/CrackMapExec/wiki) is a powerful tool that can be used for post compromise exploration with captured credentials. It essentially allows for things ranging from pass-the-password attacks, credential spraying, credential stuffing and much more. It comes pre-installed in the lastest versions of Kali Linux

After cracking apennyworth's hash using methods described in the last part it is discovered the password is the same as tdrake's.

Knowing this we can try to find machines that the account has write access to on the network. 

{% highlight bash %}
cracmapexec smb 10.0.2.0/24 -u 'apennyworth' -p 'Passw0rd!' --shares
{% endhighlight %}

![Writable Shares](/blog/assets/BatCave/crackmap-writable-shares.png)

It seems there are multiple writable shares this account has access to. Let's try to get a psexec session with it.

![Psexec](/blog/assets/BatCave/psexec-1.png)

Looks like Alfred is screwed.


![Alfred pwned](/blog/assets/BatCave/alfred-pwned.jpg)

## Next to Come
In the next part we will see how we can combine the SMB shell with psexec to perform further enumeration of the domain.

## Resources Used
[Hackers Playbook 3](https://www.amazon.com/Hacker-Playbook-Practical-Penetration-Testing/dp/1980901759/ref=sr_1_1?crid=39G7FKY1KN460&dchild=1&keywords=hackers+playbook+3&qid=1605190196&sprefix=Hackers+play%2Caps%2C191&sr=8-1)

[Heath Adams AD Zero to Hero Week 8](https://www.youtube.com/watch?v=_OseTyfXr3Q)