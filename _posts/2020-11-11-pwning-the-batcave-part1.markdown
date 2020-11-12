---
layout: post
title:  "Pwning the BatCave Part 1"
date:   2020-11-11 18:20:10 -0400
categories: Labs
---
## What is this BatCave you speak of?
![interesting](/blog/assets/BatCave/interesting.gif)


Welcome to the BatCave Lab series! This is an active directory lab that I built for the purpose of learning AD exploitation techniques.
Topics will include SMBrelay attacks, LLMNR poisoning, Silver and Golden Attacks, SAM Dumps and more. Let's get started!

## LLMNR Poisoning
So what is LLMNR? Link-Local Multicast Name Resolution is a protocol that is very similar to DNS. It kicks in when DNS name resolution fails on active directory environments. Why is this important? Well when this is enabled funky stuff can happen if a malicous user is on the network. LLMNR by default spits out NTLMv2 hashes when used on the environment. Using certain poisoning tools an attacker can trivially intercept these and crack them if the password is weak.

### Intercepting Tim Drake's Hash
The tool used in this excercise is [Responder](https://github.com/SpiderLabs/Responder). Responder is a tool that poisons LLMNR, NBT-NS and MDNS on active directory environments. It basically "answers" requests when they fail on the network. It is automatically built into the latest versions of Kali Linux. 

To listen on an interface simply type the following in a terminal:

{% highlight bash %}
responder -I [INTERFACE] -rdwv
{% endhighlight %}

Now it is time to wait for hashes to appear in the terminal. In this scenario a failed DNS query is simulated by typing in an IP address that does not exist in a browser or via failed share requests using SMB. However, in real life it would probably be best to run responder early in the morning when everyone is getting into work, right after lunch hours, or towards the end of the day when people are finishing up last minute tasks and are scrambling.

It appears that Mr. Drake is attempting to login to a terminal on the network.

![Tim_Login](/blog/assets/BatCave/tim_drake_logging_in.png)

Oh no... it looks like he just tried to access a non-existent share. It appears that his hash has just been intercepted.

![Tim_Hash](/blog/assets/BatCave/tim_drakes_hash.png)

### Cracking the Hash
With the hash in our hands we can now save this and crack it using hashcat. 

To find the mode needed simply grep the help page for it.

{% highlight bash %} 
hashcat --help | grep NTLM
{% endhighlight %}


{% highlight bash %} 
   5500 | NetNTLMv1                                        | Network Protocols
   5500 | NetNTLMv1+ESS                                    | Network Protocols
   5600 | NetNTLMv2                                        | Network Protocols
   1000 | NTLM                                             | Operating Systems
{% endhighlight %}


We are trying to crack NTLMv2 hashes so we are going to use mode 5600.

![hashcat_start](/blog/assets/BatCave/hashcat_start.png)

But surely Bruce Wayne's third apprentice would have a strong enough password...

![cracking_tim_drakes_hash](/blog/assets/BatCave/cracking_tim_drakes_hash.png)

Nope.

![slapped](/blog/assets/BatCave/slapped.jpeg)


## Next Section
That concludes the cracking section. Next I will demonstrate how these hashes can be used to gain shell access when local admin access is utilized on the environment.

## Resources Used
[Hackers Playbook 3](https://www.amazon.com/Hacker-Playbook-Practical-Penetration-Testing/dp/1980901759/ref=sr_1_1?crid=39G7FKY1KN460&dchild=1&keywords=hackers+playbook+3&qid=1605190196&sprefix=Hackers+play%2Caps%2C191&sr=8-1)

[Heath Adams AD Zero to Hero Week 8](https://www.youtube.com/watch?v=_OseTyfXr3Q)

