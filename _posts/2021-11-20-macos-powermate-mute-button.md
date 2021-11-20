---
layout: post
title: Toggle Mic Mute on MacOS with a Griffin Powermate
categories: macos videocalls
---

Many of us are on more and more video calls these days, and on a variety of platforms. One thing that frustrates me with them is that the keyboard shortcut to toggle the mic is not the same across any of them. On top of that, if the call window doesn't have focus you can't mute or unmute without getting back to it first, which can be annoying when you want to reply to someone quickly. I just let myself grumble about it for a while, until I found an old Griffin Powermate in a box in the basement.

Griffin gave up on the Powermate years ago, but you can still find them on Ebay sometimes, and the [last release of the software](https://griffin.zendesk.com/hc/en-us/articles/360004190680-Powermate-USB-Windows-and-MAC-Drivers) continues to work (even on Monterey).

To make your own button, just download the software from Griffin and set it up a global press action that runs an AppleScript.

Then add the following as the script:

{% highlight applescript %}
on getMicrophoneVolume()
	input volume of (get volume settings)
end getMicrophoneVolume
on disableMicrophone()
	set volume input volume 0
end disableMicrophone
on enableMicrophone()
	set volume input volume 100
end enableMicrophone

on setUnmutedLights()
	tell application "PowerMate"
		set aDevice to first device
		tell aDevice
			make light state with properties {state type:counter, pulse count:50, pulse length:0.1, name:"Alert x3"}
		end tell
	end tell
end setUnmutedLights

on setMutedLights()
	tell application "PowerMate"
		set aDevice to first device
		tell aDevice
			make light state with properties {state type:steady, brightness:0.0, name:"Off"}
		end tell
	end tell
end setMutedLights

if getMicrophoneVolume() is greater than 0 then
	disableMicrophone()
	setMutedLights()
else
	enableMicrophone()
	setUnmutedLights()
end if
{% endhighlight %}

When this runs, pressing the button will check if the system input volume is 0 or 100. If it's 0, we set it to 100 (mic is now active) and the lights on the PowerMate start blinking. Pressing the button when the system input volume is 100 will drop it to 0 (mic is muted) and the blinking light will turn off.

You can change the pulse rate on the button if you find it annoying, just mess around with this line and tweak the pulse count and length to your needs:
`make light state with properties {state type:counter, pulse count:50, pulse length:0.1, name:"Alert x3"}`

I like having the light blinking quickly, so that I am more likely to notice if I forgot to mute the mic. I do not want to accidentally say something over a call when I thought I was muted.

Example video is coming soon.

References:

These posts were immensely helpful in figuring out getting the PowerMate working and the AppleScript doing what I wanted.
* https://www.somegeekintn.com/blog/2010/07/secrets-of-powermate-3-0/
* https://medium.com/macoclock/how-in-the-bleep-do-i-mute-my-mic-anywhere-on-macos-d2fa1185b13
