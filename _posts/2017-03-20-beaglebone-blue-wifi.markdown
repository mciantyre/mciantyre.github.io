---
layout: post
title:  "BeagleBone Blue WiFi"
date:   2017-03-20 20:38:22 -0400
categories: jekyll update
---
A [BeagleBone _Blue_][bbblue] arrived in the mail today. First task: get it on the network.

Unlike other models in the BeagleBoard family, including the popular BeagleBone Black, the _Blue_ doesn't have an ethernet port. We instead have WiFi and Bluetooth LE for wireless connectivity. The Intel Edison is the only SBC I have used with built-in wireless. The process for getting a Blue on the network was not as obvious as it was with the Edison. Working with the Blue was straight-forward once you know the right tools. I hope this post may save you some Googling.

### Pre-first steps: talking to the Blue

I'm assuming some familiarity with the BB Black in this section. If this is your first time with a BeagleBoard SBC, check out their getting started [here][getting-started]. The Blue and the Black have a similar connection interface: connect to your computer with a compatible USB cable and wait for the LED heartbeat. Network-over-USB, as well as serial-over-USB, both work out of the box (from the Blue's perspective; you'll need to install some drivers depending on your host machine). There's no graphical environment to configure a network, so we are working from the command line. Drop yourself into an SSH or serial shell, and log in with the default credentials.

### connmanctl

`connmanctl` is a pre-installed command line tool for managing network connections. It seamlessly connected the Blue to my WiFi network, and it looks like it can also be used for managing the Bluetooth radio.

From the shell, enter

{% highlight bash %}
debian@beaglebone:~$ connmanctl
{% endhighlight %}

to start up `connmanctl`. Note: you do _not_ need `sudo`. Notice a change in the shell prompt. Enter `help` to see a list of commands. `quit` and `exit` bring you out of the utility.

Scan for wifi networks, then list services:

{% highlight bash %}
connmanctl> scan wifi
# Wait for this to complete...
Scan completed for wifi
connmanctl> services

# Lots of SSIDs...
YOUR_SSID_NAME  wifi_abc123_lotsofchars_somemanagementqualification
# ...
{% endhighlight %}

Enable the wireless agent, used for entering passphrases:

{% highlight bash %}
connmanctl> agent on
Agent registered
{% endhighlight %}

Finally, connect to your network using the `wifi_abc123_reallylongstring`:

{% highlight bash %}
connmanctl> connect wifi_abc123_lotsofchars_somemanagementqualification
# Enter password, if prompted
{% endhighlight %}

By this point, a very green light shines from the Blue. Exit `connmanctl` with `quit` or `exit`. Check out the `wlan0` field of `ifconfig` to see your IP. Finally, try a ping or a `sudo apt-get update`!

The Blue auto-connects to known networks after powering up, so there will be no need to re-connect every startup. The help documentation has an "autoconnect" configuration, but I didn't personally play with it yet.

### Future TODOs:

- The Blue also has `wpa_supplicant` installed, but I didn't have much luck getting it to work. Would be interested in exploring further.
- Try with Bluetooth.


[bbblue]: https://beagleboard.org/blog/2017-03-13-meet-beaglebone-blue/
[getting-started]: https://beagleboard.org/getting-started