---
title: Developing naive rootskits for unix systems
date: '2021-11-03'
draft: true
--- 

A recent lecture I was attending called *Cyber Defense and Digital Forensics* made us develop naive linux rootkits. We were given the following tasks:

```
- Each group will have access to an *enemy* group's network first, then gain access to their own network later.
- In 3 weeks time, develop a collection of persistent linux rootkits to the best of your ability
- After those 3 weeks, we'd again have 3 weeks time to find the shenanigans a different group installed on *our* system
```

A group's network existed in 8 different servers that were able to communicate in a subnet:

- Mail
- Application
- Gitlab
- Admin
- Postgres
- ...

### Planning Phase

In a structured project like this, the first thing to do is planning. We started by collecting different ideas ranging from those that were easily found with the right tools to those of which we were unsure if they were even possible to begin with. 

Here's a loose list of ideas which were memorable enough that I still remember them:

- Install a reverse shell via netcat backdoor on port 7
- Install telnet
- Recompile the ssh binary to accept *any* rsa certificate
- Recompile different binaries to hide changed hashes in target executables (md5, apt, ...)
- At server startup, start a local qemu session and forward all ssh requests except those from specific IPs to that session. Overwrite the 'exit' command to directly disconnect the ssh session
- create an apt repository which contains malicious binaries if the enemy team decided on reinstalling anything
- write a small script that changes the date of *every* file on the system to hide what we changed
- install a malicious linux service which hides backdoors to the system
- symlink all shell histories to /dev/null
- automatically clear the apt cache every once in a while to hide previously installed headers
- hide malicious data in a custom virtual disk via `dd`
- recompile the linux kernel to hide malicious code and hook syscalls to hide file accesses
- add a linux kernel module which hides different processes
- recompile `gcc` to add malicious code after a compiled program's main function


### Choosing what to work on

The biggest difficulty in a project like this is always the hard time limit that comes with its rules. We only had 3 weeks and 5 team members, so we had to carefully choose those ideas that seemed like we could realistically expect to implement them in the given time. 

First of, we chose to *install a backdoor on port 7 via netcat*. We also installed `telnet` on each of the servers. These are unbelievably easy to find via any process status tool. However, if we were to change anything in relation to the `ssh` famliy later on, we still had access to the server in case thing went wrong (which they did, as we will find out later).

We also decided to recompile the linux kernel in order to hide different syscalls on their service. This one actually turned out to be pretty complicated, but previous kernel experience from one of our team's members made this possible. 

Symlinking histories, changing up timestamps and other obfuscating techniques like clearing application caches were absolute nobrainers as well. We could have the best rootkit possible, but if the enemy team found simple traces of our activity, it would all be for naught. For example, if we left `zlib` and `openssl-dev` headers on their previously pristine systems, they'd immediately know we recompiled an `ssh` binary.

This brings me to our final plan: We wanted to find the exact place in the `sshd` binary where key files were checked, and hardcode an `rsa` key which would be accepted in any case. Since I was the one suggested this simple trick, it was my task to realize it. 

### Finding keyfile logic in the sshd binary

