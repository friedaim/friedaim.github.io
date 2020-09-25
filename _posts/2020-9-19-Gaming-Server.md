---
layout: post
title: "Try Hack Me: Gaming Server Walk Through"
---
![]({{site.baseurl }}/images/gamingServer/Title.png)

### Pre-Requisites
+ Beginner-Intermediate understanding of Linux Operating Systems and file structure.
+ Intermediate understanding of how webserver, the internet, and networking works.
+ Understanding of common penetration testing tools like Nmap, dirbuster, gobuster, dirb, etc.

#### Note
This machine is a very beginner-level machine, therefore, I'll be going into detail with almost everything I do in this walk through.


# Begin Walk Through
After navigating to the [Gaming Server Room](https://tryhackme.com/room/gamingserver) on the Try Hack Me website, we deploy and retrieve the IP address of the machine. The first steps in CTF or pentesting is to scan it with a port scanner. For this you can use [Nmap](https://nmap.org/) which comes built into Kali Linux, or the more popular option recently [Rustscan](https://github.com/RustScan/RustScan).  

## Scanning
So, after (hopefully) reading up on the Nmap and/or Rustscan material we will have generally learned how to scan a machine for open ports. I recently prefer Rustscan purely because it shows me what ports are open as it finds them instead of waiting until the entire scan finishes like Nmap.  

After scanning the machine with the Rustscan command below,

![]({{site.baseurl}}/images/gamingServer/rustscan.png)

we see that `port 22` and `port 80` are open. Meaning we have `SSH` and a `Webserver` running. Before looking further into the webserver, we should multitask by starting another scan first. It is also good practice to perform a `-A` Nmap scan on the ports we found open so we can find version info, OS info, and do general vulnerability scans off the bat. Luckily, Rustscan also has built-in functionality for running Nmap commands after its scan.

![]({{site.baseurl}}/images/gamingServer/rustscan-a-command.png)

## Enumeration
Now, with that running we can take a look at the website that is running on port 80. First impressions, it looks like a template website, or at least a website in the making. It's always good practice to look through the `source code` of websites to see if you can find any clues about where to look for insecurities. Taking a look at the source code of the home page we see that (at the very top) this website was indeed made with a template (**potential vulnerability to research later**. Additionally, at the bottom of the source code we see the comment `<!-- john, please add some actual content to the site! lorem ipsum is horrible to look at. -->` indicating that there might be a potential user `john` available on the remote machine through `SSH` (**another potential vulnerability to research later**).


If we use the `-A` flag after specifying we want to input an Nmap command into Rustscan, we get this:

![]({{site.bareurl}}/images/gamingServer/rustscan-a.png)

which has much more information to it, but the bulk of what we want to know for not is here. We get the version info of both the SSH and the webserver, take note of these by saving all the Rustscan output in a file.

Next, let's take a look at the other pages on the website. It seems like there's only 3 different pages on the website. Viewing the sources and general look of all the pages yields nothing out of the ordinary or suspicious. However, after more exploration on the home page and clicking different links, we see that the `Read More` button takes us to a 404 page.

![]({{site.baseurl}}/images/gamingServer/404-page.png)

Take note of this, because it's an information disclosure, showing the webserver service running and the version number. With this we could **potentially find vulnerabilities with the version number**. On the subject of links from the home page, **try to find another link on the dragon lore page leading to some information disclosures**.

Hopefully, you've explored the home page some more to discover a small link that links to some pretty sensitive information. The upload button leads to a pretty sensitive page with some interesting content.

![]({{site.baseurl}}/image/gamingServer/uploads-home.png)

Whenever you find information like this, download all the files in the directory immediately. Upon further inspection, it looks like we have a potential wordlist for passwords, a hacker manifesto, and a meme.

Maybe after a few more minutes of looking around, you'll start to feel stuck. This is where we introduce something like [gobuster](https://github.com/OJ/gobuster) and/or [sublist3r](https://github.com/aboul3la/Sublist3r). When you have a webserver of any kind, it's always good to **list directories**, because we don't know what the server could be hiding. Additionally, (*this is more for geared towards live targets like bug bounties*) we can list subdomains of websites with a tool like sublist3r. Go read up on both these if you are not familiar with them. **Let's run a Gobuster scan against the target, using the small wordlist that comes with dirbuster**.

While the scan is running, lets take a look at the wordlist we found from the /uploads directory. It's exactly as I expect, a password list. So, this means we should probably use it on something (**possibly to brute force our way into the SSH user john?**). However, after about 2 minutes of running Gobuster, it finds a secret directory called `/secret`. Going to that directory on the browser reveals something interesting...

![]({{site.baseurl}}/images/gamingServer/secret.png)

After downloading the file, we see that it looks like an SSH key. However, when we try to `SSH` into `john` at the target's IP using the key we found in the directory as an identity file, it prompts us for a passphrase. So I looked into this more, **literally googling "how to crack ssh key passphrase"**, I came across [this article](https://bytesoverbombs.io/cracking-everything-with-john-the-ripper-d434f0f6dc1c). This article really helped me out in finding out the next steps, so read through it and come back to this. In the article, it basically introduces the [SSH2John](https://github.com/openwall/john/blob/bleeding-jumbo/run/ssh2john.py) utility, which we need to use to convert the ssh key to a crackable john file. Then with this we can crack the password to the file by using the password list we found on `/uploads`.


## Exploitation

Okay, so let's convert the ssh key we found into a [JohnTheRipper](https://github.com/openwall/john) readable file and be sure to put the output into a new file `ssh2john.py ssh.key > sshjohn.txt`. Now, we need to use JohnTheRipper to crack the key using the word list `dict.lst` from the `/uploads` directory. I wasn't sure how to do this, so I just tried it and john spit back an error saying `Use the "--format=SSH" option to force loading hashes of that type instead`, which I added and it then cracked a password.

![]({{site.baseurl}}/images/gamingServer/sshkeycracked.png)

Alright, now we have the password to login to the `SSH` user `john` at the targets IP address. **Let's Login** with the ssh key we found on `/secret` and the password we just cracked from the key. **Aaaaand we're in!**
Now let's peak around a bit, we find the `user.txt` in the home directory. Take a look around and try to find anything interesting.  

## Privilege Escalation

The next steps here are almost always to escalate our privileges to try to gain root or full access to the system. Some helpful things to lookup are `Privilege Escalation Linux`, or `Tips for Priv Esc on Linux`, stuff like that.
Typically, the first commands or things to test would be to see if the current user is on the sudoers list with `sudo -l`, but that is also a problem in this instance because we don't know the user `john`'s password, we only know the passphrase for the ssh key we submitted. Okay, so next steps would be to take a look at the `/etc/passwd` to get an idea of what users are on the system and/or some of the services installed. When looking at the users file, note that the UID for user accounts start at `1000` (excluding the root account), so everything from `1-999` is a service account meant to be run for a particular service, that was either installed or used by the system. Also when looking at this, if you're unfamiliar with what typical `/etc/passwd` files look like, look one up! Having looked at that information, there's nothing that really stands out, some other commands you may have run into when looking at Priv Esc articles is **taking a look at what groups the user is a member of**.

When looking at the groups the user is a member of and comparing it to the typical Linux groups, we see that there's one service running called `lxd`. After looking up LXD and reading up on it through [articles](https://linuxcontainers.org/lxd/introduction/), we see that it's essentially a service that runs a VM of a terminal (very broad definition). Okay, so now let's just look up an exploit with it. We come across [this exploit tutorial](https://www.hackingarticles.in/lxd-privilege-escalation/) showing us how to escalate our privileges through LXD.

Normally in the pentesting field, you will have to research multiple different techniques and articles before anything works. After reading through the article and potentially following along, you will likely find that for some reason the target machine cannot launch a LXD container for Ubuntu 18.04, this is where I got stuck.  
*Note: It's 100% okay to get stuck. You will get stuck often, sometimes we just need to take a break and come back in 10 mins or maybe an hour or two.*  
I later went back to my original google search of `LXD exploit` and found [another article](https://book.hacktricks.xyz/linux-unix/privilege-escalation/interesting-groups-linux-pe/lxd-privilege-escalation) that has a writeup on exploiting lxd **without internet**. The article introduces a prebuilt and offline image for LXD available on github called [lxd-alpine-builder](https://github.com/saghul/lxd-alpine-builder). *Note: I read over method 1, realized it was a lot of things to download and skipped it, plus method 2 is so much simpler.* Download the lxd-alpine-builder to your attacker machine either through git clone or wgetting the raw `build-alpine` file in the repository. Then build the file by enabling execute permissions and running the `build-alpine` file. *Note: You may need to run the file multiple times in order for the correct file structures to be created.* Once the build is complete, you should have an `alpine_blah_blah.tar.gz` file. This file is the entire image we need to be able to escalade our privileges to root on the target machine.

Alright, you should have something like this:

![]({{site.baseurl}}/images/gamingServer/alpine-ls.png)

I recommend renaming `alpine_blah_blah.tar.gz` to something easy like `alpine.tar.gz`. Now we need to actually transfer the alpine image to the target machine, but since it doesn't have internet we need to host a server ourselves and wget or curl the alpine image onto the target machine from the target machine. Here's the steps:  

1. Start a SimpleHTTPServer (Python 2) or http.server (Python 3) module from python in your project directory for GamingServer. If you don't know how to do this [read about it here](https://medium.com/@ryanblunden/create-a-http-server-with-one-command-thanks-to-python-29fcfdcd240e).
2. Next we need to fetch the alpine image from the attacker machine and download it to the target machine. To do this we can use curl or wget, it doesn't matter. For this just make sure to put the address in a url format, and include your TryHackMe IP address (*your tun0 adapter*), as well as the port number the server is being hosted on.

**Switch back to the target machine** where we're logged in as `john@exploitable` and navigate to a place in the file system where you can create directories, I chose `/var/tmp/` for this. Now lets curl a copy of the alpine image into the directory (*I also created a directory for lxd for easy removal later*). Now that it's on the target machine, lets keep following the tutorial. The next step would be to import the image with `lxc image import ./alpine.tar.gz --alias *image name here*`, then create a container with `lxc init myimage mycontainer -c security.privileged=true`. Just keep following the tutorial on the article from here. So from here we mount the **/mnt/root** folder onto the container and start the container. Now we need to execute `/bin/sh` from the container using lxc, then naturally I would navigate to the `/root` folder to try to find the `root.txt` However, we **mounted** the root folder to a different location, if you can recall from earlier. After navigating to that location we see a `/root` folder where, after navigating into that directory, we see the **root.txt** file.

![]({{site.baseurl}}/images/gamingServer/root.png)

**Now we can celebrate!**

## Covering Tracks
Finally, before we call any pentest done, we need to erase any evidence that we were on the system. There's many levels of depth this can go but for this instance we'll just leave it at deleting any files we created on the target machine. So go back to the `/var/tmp/` directory and delete the entire lxd directory. Some other measures we can take to cover our tracks include: deleting our lxd containers and images, deleting our presense from log files, and others. If you want to learn more about covering your tracks after pentests, read [this article](https://github.com/lamontns/pentest/blob/master/post-exploitation/covering-your-tracks.md). 
