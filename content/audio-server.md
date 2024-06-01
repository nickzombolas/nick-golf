---
title: "Audio Server"
date: 2024-02-19T19:15:55-05:00
---

# Home Audio Server

Client and Server configuration files for streaming music across the network, utilizing Ubuntu (Desktop and Server), PulseAudio, and Netplan. This is a project I did because I needed the ability to 'cast' my computer audio to a set of speakers on the other side of the room, so I used this as an opportunity to learn some basic things on server management. It's possible there's a better way to do this but this is the way I ended up figuring it out, and I'll mention some of the difficulties I encountered.

#### Architecture
For the project we have my desktop computer as a client, and my Raspberry Pi connected to my speakers as a server. Since the client and server are running the same software on the same network, we can have them communicate with each other. The client is configured to broadcast it's audio, and the server is configured to recieve audio, all over my wifi network. The audio recieved by the server is then played on my speakers. Let's talk a bit more in depth about each side.


### Server
To get started, I installed Ubuntu Server and enabled SSH using the `openssh-server` package. Now I can do the rest of the work more comfortably from my desktop via SSH with `ssh <user>@<ip_address>`. 

#### Configure a Static IP Address
I need to configure a static IP Address so I can easily connect if i turn the server off for any reason. For this i used NetPlan to set the IP of my choice. [Here](https://github.com/nickzombolas/audio-server/blob/main/server/netplan/01-network-manager-all.yaml) is the configuration file for this, with specific values templated out. To apply this configuration you can replace the contents of your `/etc/netplan/01-network-manager-all.yaml` and add your specific values. Use `sudo netplan try` to test your configuration. If there are no issues, you can use `sudo netplan apply` to persist these changes.

#### Server-Side PulseAudio
On both the client and the server we are running PulseAudio, the most popular audio server for Linux. On the server, we must configure PulseAudio to recieve audio and output it to the speakers. I've provided the entire configuration file [here](https://github.com/nickzombolas/audio-server/blob/main/server/pulseaudio/default.pa), but in reality we only need to change 1 line in the `/etc/pulse/default.pa` file. This line is commented by default, but we must simply uncomment it to allow the server to act as a reciever. Once this is completed, the server is good to go.

```bash
load-module module-rtp-recv
```

### Client
It doesn't matter if the client has a static IP or not, so in this case we only need to make some edits to PulseAudio.

#### Client-Side PulseAudio
Similar to the server, I've provided the entire configuration file [here](https://github.com/nickzombolas/audio-server/blob/main/client/pulseaudio/default.pa), but we'll only need to change a couple lines. The following lines are commented by default, so we must uncomment them and make some changes. These lines create a new sink (an output source) and then allow us to send audio to the sink. We set the ip of our server for increased reliability. Now our server is considered as an output device to my computer, the same way physically plugged earphones would be a new output device. Now we're ready to play some music.

```bash
load-module module-null-sink sink_name=rtp sink_properties="device.description='server'"
load-module module-rtp-send source=rtp.monitor destination_ip=<server_ip>
```


#### Usage
When I turn on my computer the default output is my physically connected speakers, but they aren't very good. So to use my better speakers across the room, I can change the output source of my computer audio to `server` and connect to the server via SSH. Typically the server will remain on since the Raspberry Pi won't draw much power, but if not i can simply boot the pi and turn on my speakers. Now with the client broadcasting and the server recieving I can listen to whatever audio is playing on my computer from my good speakers. To stop, I can just switch my computer audio output back to my local speakers and exit the SSH connection.



#### Difficulties and Improvements
It seems very simple since in each case we only had to modify a few lines, but it did take a lot of time of tinkering and learning on my own. The current solution does have some inconsistencies that I'm working to solve. First, it can be unreliable for reasons I don't know yet. Most of the time it works very well, but sometimes it can sound crackly because I'm experiencing packet loss. Other times, PulseAudio on the server-side can crash and restart, due to something about the sampling rate being incorrect. I'm not excactly sure what the root cause of each issue at the moment. To get more insight I monitor the server system logs with `journalctl -f` and verbose PulseAudio debug logs with `pulseaudio -vv`. At the moment, the client and server must be connected via SSH to recieve audio. Ideally this should not be the case since they are on the same network, so it would be nice if i simply had to change the output source on the client. 

Update:
After explicitly setting the ip of the server, packet loss is no longer occuring, and music is loud and clear without crackling. 
