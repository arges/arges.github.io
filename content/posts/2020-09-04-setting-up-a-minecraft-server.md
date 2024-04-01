---
type: post
title: setting up a Minecraft server
date: '2020-09-04T10:22:46-0500'
author: arges
tags:
- linux
- parenting
modified_time: '2020-09-04T10:22:55-0500'
---

We've been spending a lot of time at home lately due to the current pandemic.
Because of this the kids have really enjoyed playing Minecraft together.
Initially I set up a LAN server for them to play together, but wanted to set up
a community server that their friends could also access. Here I'll describe
some approaches to this with the goal of keeping things automated, low cost,
secure and easy for friends to play.

I'll be only talking about setting up a Minecraft server that works with the
'Java Edition' of Minecraft since it is possible to deploy on your own. This
becomes a source of confusion for many friend's parents as they may have
installed the myriad of other editions that exist.

This is a fun write-up because it really is a mini exercise in deploying a
service and having 'customers'. I'll go through hosting options, deployment,
managing the application, monitoring and various other things I encountered
along the way.

# Specifications

Ideally we'll be hosting about 10 player max, so we don't need a ton of
CPU/Memory. In addition we'll want to be able to do backups periodically in
case there are crashes. In practice I've been getting away with a 2 vCPU
machine with 8GB of ram with proper JVM tuning flags. It seems that the
Minecraft server is very CPU intensive and needs faster clock speeds rather
than more parallel threads.

# Hosting

One could choose a first-tier public cloud such as AWS, GCP, or Azure. Another
option would be using a VPS and paying monthly. This is such an open ended
question and really dictates your cost. I chose GCP because they give free
credit, I had never really used the service, and they had a pretty good write
up on how to run Minecraft on GCP.

# Deployment

Here are a few write-ups on how to deploy Minecraft on GCP. This is where I
started to figure out what was needed.

- https://cloud.google.com/solutions/gaming/Minecraft-server
- https://cloud.google.com/blog/products/gcp/brick-by-brick-learn-gcp-by-setting-up-a-Minecraft-server
- https://medium.com/@jtlimson/deploy-Minecraft-server-in-gcp-d3536e2e1d82

Instead of duplicating the above articles, I was able to follow these mostly
to get things setup. Below I'll go over how I made operations a bit smoother.

# Operation

After using the above write-ups to get into working state, operating this
service requires a bit more tuning and attention. For example just running
Minecraft in a screen session doesn't really help if things crash. The first
thing I did was to make a systemd service to run my service. The first version
was a little rough since it was grafted on top of the original example:

```
[Unit]
Description=Minecraft Server
After=network.target

[Service]
Type=simple
RestartSec=5
WorkingDirectory=/home/user

User=user

Restart=always
ExecStart=/usr/bin/screen -L -D -m -S mcs java -Xms1G -XX:+UseConcMarkSweepGC -XX:ParallelGCThreads=2 -XX:+AggressiveOpts -Xmx7G -jar /home/user/server.jar nogui

[Install]
WantedBy=multi-user.target
```

With the above service running, if there was a crash the server would be
relaunched. In addition, I was able to attach to the screen session and issue
commands as needed.

# Backups

In addition I added a backup script that would be run by a cronjob:
```
#!/bin/bash -x
/snap/bin/gsutil cp -R /home/user/world gs://somename-Minecraft-backup/$(date "+%Y%m%d-%H%M%S")-world
```

# Tuning

With these settings things were mostly stable, if there were crashes the
service would restart. However I would very frequently see the message:
```
[16:23:42] [Server thread/WARN]: Can't keep up! Is the server overloaded? Running 12503ms or 250 ticks behind
[16:24:08] [Server thread/WARN]: Can't keep up! Is the server overloaded? Running 10362ms or 207 ticks behind
```

This is an indication your CPU is a bit underpowered. So scaling up your CPU
would help here. In addition the server may actually terminate if it gets too
overloaded. So I had to change the following setting in `server.properties` to
get this working:
```
max-tick-time=-1
```

# Security

I wanted my Minecraft server to be 'invite-only' so that only users I knew
could access the service. The only mechanism I found for this with Minecraft
server would be to 'white-list' access to specific user names.

Ensure you set the following in server.properties:

```
white-list=true
```

Now you must manually white-list new players that join the game. You can do
this by adding names in the `whitelist.json` file,
or by using `/whitelist add <username>` in the Minecraft console.

Finally ensuring that automatic updates are performed is critical:

```
sudo apt install unattended-upgrades
# edit /etc/apt/apt.conf.d/50unattended-upgrades with proper settings
```

# Conclusion

At this point the only other things is having fun and reducing costs. Overall
this has been a hit and something that's provided a lot of fun for my kids.
