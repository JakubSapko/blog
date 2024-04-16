+++
title = 'Fairy magic - about a Pixie Dust Attack'
date = 2024-04-14T10:49:28+01:00
draft = false 
tags = ["Cybersecurity", "Theory", "HTB"]
+++

# About

Lately I've been going through the **[WifineticTwo](https://app.hackthebox.com/machines/WifineticTwo)** machine on HackTheBox as a part of my Cybersecurity/pentesting adventures. Throughout years I always considered networks and everything connected with, well, connecting computers with each other as one of the most challenging and difficult things in IT that I've tried.

This is why I considered this machine to be quite challenging - especially the root flag part. On the other hand, the method that I found and used to obtain the root flag was so clever, interesting and jaw-dropping that I decided to dig a little bit more on it and consolidate my knowledge about this attack type.

I found it really hard to find any comprehensive source of knowledge about a Pixie Dust Attack so I also decided to share what I found and how I understand it right there, in this blog post. Hopefully you'll enjoy!

# Background

This is not a full WifineticTwo writeup, so I won't bother with leading you through the whole machine up to the point where this attack was needed to be used to advance further, however I think it is worth understanding at what point it was used, why and how did I manage to think about the potential attack vector.

It all started when, after gaining the initial foothold and receiving user's flag, I couldn't find anything relevant that would lead me to a root flag. I don't have a lot of experience in CTFs, I'm just 4 months into the whole CyberSec journey, however, what I've noticed is that usually you either get a somehow clear indication of where to look for vulnerabilities (which comes at the cost of the actual attack being harder to perform), you get something that seems suspicious (or rather, different from what you've seen so far in other CTFs), you get a strong clue from the machine's name or you get any combination of former.

In that case it was the combination of the second and third point.

# Understanding what's going on (sort of)

Let's start with the suspcious files that I've found. 
When browsing through the machine I stumbled upon the Bash scripts related to setting up [systemd-networkd](https://wiki.archlinux.org/title/systemd-networkd) daemon in different scenarios. 
This (along with the machine's name - WifineticTwo) made me think that there must be something to do with networks and this kind of stuff. 
At this time, I also run a ```systemd-detect-virt``` (https://www.linux.org/docs/man1/systemd-detect-virt.html) and realized that I'm in fact inside the Linux container ![linux container](/linux_container.png)



Running ```iwconfig``` confirmed that there's a ```wlan0``` interface available. ![wlan confirmation](/wlan.png)


All of this joined with the machine's name (from what I've found on different forums - the first Wifinetic machine was somehow famous for teaching the network attack vectors) made me think that there must've been another machine that I need to hop over from the container I gained access to.
This machine happened to be a local (for the machine) network's router.


# Educate yourself, dumbass!

This machine really challenged my status quo on the networking knowledge. You see, up to this point I've always thought that I won't ever need to get more into how the networks actually work, how the networks elements are configured, what they are supposed to achieve and so on.
It just wasn't my thing, ever. What I've known was really just enough to get my devices set up within my home network - that's all. Well, it's fairly easy when you've got all the credentials, isn't it?
The problem with pwning things is that you need to KNOW the stuff you're trying to own. Like, really know. What I've learned so far from the whole CTF/reverse engineering journey is that most often you don't really know how the things you're using work.
Like, sure, you might have some sort of UNDERSTANDING about the things you're touching but do you really KNOW how they work? The difference between understanding and knowing is tremendous. So I might have said that I understand the basic concepts of networking but I didn't really know them.
And when you know what you don't know, there's only one way from there - to educate yourself on the topic.

So I did. I started going through the masses of more or less useful articles, PDFs, videos, etc. on how the networks work. You see, it's fairly easy to say what must've been achieved - get the credentials to router, get to the router, most likely find the root flag as it is just a "medium" difficulty machine and you don't expect another machine jumps to be needed. The thing is - I couldn't find any trace of the router credentials on the machine. None. null. Nil. Nothing.
This is why I spent so much time going through all the resources that I could find. Because I was clueless. I had no other trail than to go through as much material as possible, hoping to find anything useful.

And I did.

# Getting something from nothing

You see, usually to connect to WiFi network, you need to provide two values: SSID (a "username" of sorts) and a Pre-Shared Key (which you can think of as a "password" to your router).
However! What if, just what if, someone forgot their router credentials? Well, worry not, because some smartass engineers thought about you! There's a high chance, that if you investigated your router a little bit more you'd find a value called WPS PIN.
WPS PIN is a mechanism that was supposed to let you easily recover the router's credentials or even log into the router itself. By providing this 8-digit number you're getting a full access to your router. This is where fun begins.
I found out that back in the time the common strategy to owning routers was to just bruteforce the WPS PIN. You'd think that this would take a lot of time (also depending on the distance between you and router) as it is \(10^8\) combinations but you'd be wrong - it's a lot less.

The last digit is considered a cheksum - so we can skip it. Now we got \(10^7\) combinations. But there's a small, but important, implementation detail. You see, you'll get \(10^7\) combinations if, and only if, you'll treat those 7 digits as a single number. 

But you don't.

WPA's specification actually says that the WPS PIN is validated as two, separate 4 digit numbers. This means that you end up with \(10^4+10^3\) (remember the checksum number) combinations instead of \(10^7\). Roughly 11k against 10 milions. This makes it extremely vulnerable to any bruteforce attack.
The feasibility of cracking the WPS PIN in the bruteforce attack made the manufacturers think about the security of the WPS a little bit more. Because the PIN validation is an architecture flaw the only way they could've (and had) gone is to hard-stop the potential attack at some point. Thus most of the network chips will block connections after several connection attempts, making it harder or near-impossible to break using any bruteforce technique.

And this is where Pixie Dust attack comes into play.

## A true wizard needs only one spell

So, if you have a limited amount of connection attempts in which you can try to break a PIN (sometimes limited up to one attempt) but you're a smelly evil hacker who wants to own the router, what would you do? Of course you'd find a way to find out the PIN in one shot. This is what Pixie Dust attack attempts to achieve.

To understand how we're able to find out the WPS PIN in one shot we first need to understand how the router-client connection works and what are the necessary steps that need to be taken in order to validate yourself to a router.
Picture below explains what steps (and in what order) are taken when trying to connect to enrolee.

| ![wps](/wps2.png) |
|:--:|
| *All thanks to @Dominique Bongard (a father of Pixie Dust) for this great presentation: http://archive.hack.lu/2014/Hacklu2014_offline_bruteforce_attack_on_wps.pdf* |

If you want to fully understand the whole process I suggest you visit the link in the above picture's caption - it is a PowerPoint presentation made by Dominique Bongrad, who first noticed the possible vulnerability and who created the Pixie Dust attack, and it goes into great details in order to explain the attack to it's core.

For us, the most important step is the M3 message. This is because at this step the enrolee (router) serves hashed values of E-S1 and E-S2 to a client. E-S1 and E-S2 are basically hashes of the both PIN halves that we're looking for - this is quite a simplification, but it's enough for us to understand the attack, if you wish to know more details about what, and in what form, is sent in those messages please refer to the Dominique's presentation.
One question that arises is - "so why would the enrolee send the PIN to a client, if it is a client who wants to connect to the router?". This is done because both the client and the enrolee need to prove that they know the PIN. This is done so that a client can have a
confidence that it is trying to connect to the correct AP and not the one that could be considered rogue.

There is one more thing to this step - cracking those hashes would seem fairly easy because, as we remember from before, there's a small number of combinations available. But it can be even easier. You see, for some strange reason, the manufacturers thought that it will be enough if the random number generator [seed](https://en.wikipedia.org/wiki/Random_seed) would be set to either 0 or a timestamp of a connection. Knowing both the RNG seed and the hashing algorithm that was used, an attacker is able to crack the PIN in a single shot.

| ![attack](/attack2.png) |
|:--:|
| *A result of the attack - cracked PIN, password and AP id* |

# Aftermath

The question arises - alright, if this is so easy to own someone's router, how can I protect myself from being attacked? First of all - this does not apply to _all_ routers. Only a number of manufacturers confirmed that their hardware and firmware are vulnerable. Of course, this is in no way a confirmation that it does not apply to others.
If you want to be 100% sure that you won't get owned this way you should disable the WPS PIN connection option in your router's settings. However, if it happens that you have an older/less reliable router, it is possible that even if you switched the option off the router will still be vulnerable - this is because sometimes you are told the option was disabled but in reality it is still on.
If this is your case - you should probably think about changing your stuff to a newer one.

# Summary

Hopefully I managed to introduce you a little bit to the topic of Pixie Dust attack and WPS configuration. If you have any comments or thoughts that you mind sharing, please reach out to me either on [LinkedIn](https://www.linkedin.com/in/jakub-sapko/) or via email - jakubsapko@protonmail.com
