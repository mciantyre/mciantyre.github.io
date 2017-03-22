---
layout: post
title:  "BeagleBone Blue WiFi"
date:   2017-03-21 20:10:22 -0400
categories: embedded
---

A [BeagleBone _Blue_][bbblue] arrived in my mail today. My first task: get it on a wireless network.

Unlike other models in the BeagleBoard family, including the popular BeagleBone Black, the _Blue_ has no ethernet port. The Blue is all wireless with WiFi and Bluetooth 4.1 and LE. Also unlike the Black, the Blue does not have a graphical interface. Therefore, we rely on command line tools to configure wireless networking.

Most of this was new to me. Apart from some projects with the Intel Edison, I never had to set up WiFi on a headless Linux system. I hope this post saves you some searching!

### Talking to the Blue

I assume familiarity with the BB Black in this section. Check out the getting started [here][getting-started] to learn about first-time startup.

The Blue and the Black have a similar interface: connect it to your host machine with a USB cable, and wait for the LED heartbeat. Network-over-USB, as well as serial-over-USB, both work out of the box (from the Blue's perspective; you may need to install some drivers depending on your host machine). Drop yourself into SSH or a serial screen session, and log in with the default credentials.

### connmanctl

`connmanctl` is a pre-installed command line tool for managing network connections. The utility seamlessly connected the Blue to my WiFi network.

From the shell, enter

{% highlight bash %}
debian@beaglebone:~$ connmanctl
{% endhighlight %}

to start the utility. Notice a change in the shell prompt. Enter `help` to see a list of all commands. `quit` and `exit` stop the utility, saving configurations.

Scan for wifi networks, then list services:

{% highlight bash %}
connmanctl> scan wifi
# Wait for this to complete...
Scan completed for wifi
connmanctl> services

# Other SSIDs...
YOUR_SSID_NAME  wifi_abc123_lotsofchars_somemanagementqualification
# ...
{% endhighlight %}

Enable the wireless agent for entering and accepting passphrases:

{% highlight bash %}
connmanctl> agent on
Agent registered
{% endhighlight %}

Finally, connect to your network using the `wifi_abc123_blahblahblah` value:

{% highlight bash %}
connmanctl> connect wifi_abc123_lotsofchars_somemanagementqualification
# Enter password, if prompted
{% endhighlight %}

By this point, a very green light shines from the Blue. Exit `connmanctl` with `quit` or `exit`. Check out the `wlan0` field of `ifconfig` to see your IP. Finally, try a ping or a `sudo apt-get update`!

The Blue auto-connects to known networks after startup, so there will be no need to re-connect. Consider the "autoconnect" configuration if there is a need to change this behavior.

### Future TODOs:

- The Blue also has `wpa_supplicant` installed, but my endeavor using it flopped. I would be interested in exploring further.
- Follow the same process for Bluetooth. The `technologies` command lists the Bluetooth radio. The process may not be applicable.


[bbblue]: https://beagleboard.org/blog/2017-03-13-meet-beaglebone-blue/
[getting-started]: https://beagleboard.org/getting-started