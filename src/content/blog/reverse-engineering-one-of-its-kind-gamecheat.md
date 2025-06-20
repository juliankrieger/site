---
title: Reverse engineering a one of a kind fifa gamecheat
date: '2021-09-30'
draft: true
---

Recently, I was able to get my hands on a one of a kind cheat for FIFA 22, which was released almost a week ago.
It's similarity to the exact same software for the game's previous version lets me suspect that the developers at
EA are using much of FIFA 21's game code for FIFA 22. 

There are a couple of interesting things about the cheat executable, which is why we'll be trying to reverse engineer it!

First, when starting up the executable, we're greeted by an activation text box. 

// image here

Entering any kind of message and clicking on "Activate" will tell us that the activation code is incorrect, and the 
program will close.

// image here

So far so good. 

With activators, my first question is always about how they tell if an activation code is legit or not. I'm having a bit
of a suspicion that *this* activator may talk to a server which tells it if the entered code is correct. Since we don't seem to 
have to enter an account name, another suspicion is that it records our hardware id and sends that to the server as well. 

Now, to confirm what we suspect, let's open wireshark and have a look!


