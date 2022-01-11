---
layout: post
title: "Automatically turn on camera lighting using HomeKit on MacOS"
modified:
categories:
excerpt: "My desk lamp now turns on whenever I join a meeting for work. Magic! This guide will walk you through setting up something similar for your Mac."
tags: [automation]
comments: true
image:
  feature:
date: 2022-01-05T11:37:25-05:00
---

Looking great on camera while working from home requires good lighting.

Working at Doximity, a company that has been remote-first since long before the
pandemic, necessitates a fair number of video calls as we collaborate and work
on challenging problems together. Though meetings constitute only a fraction of
my time, I appreciate having some good lighting during those calls. I also appreciate
having a little less light at other times to avoid irritating my eyes. I have a
simple desk lamp that does the job, but it's slightly tedious to turn it on and
off around my meetings. Time for some **auomation**!

![AUTOMATE ALL THE THINGS! meme](https://i.ibb.co/NnvdVg8/automate.jpg)

I picked up a pack of [Meross Smart Plugs][] that are HomeKit-compatible, so I
could control them with my iPhone or Mac, added one using the Home app on my
phone, and plugged in the lamp. After I could toggle the lamp on and off using
the Home app, I set out to build some automation to turn on and off the lamp
whenever my Macbook camera turns on or off.

This guide walks through the process of setting up HomeKit automation on MacOS
whenever any of your Mac's cameras turn on or off.

[Meross Smart Plugs]: https://www.amazon.com/gp/search/ref=as_li_qf_sp_sr_tl?ie=UTF8&tag=nilbus-20&keywords=meross%20smart%20plug%20mini%20homekit&index=aps&camp=1789&creative=9325&linkCode=ur2&linkId=27ebef0cb81e4e253735c537634f16f8

Overall steps
-------

1. Identify camera off events for your triggers.
2. Use the Shortcuts app to create shortcuts that trigger your desired HomeKit actions.
3. Create a script that triggers shortcuts when the camera turns on/off.
4. Run the script manually or automatically.

Identify camera events for your triggers
---

The `log stream` command provides a stream of logged system events, which we can
filter and monitor for camera on/off events.

In Terminal, run this to filter the logs to the expected messages on MacOS Monterey 12.1:

    log stream | /usr/bin/grep -E 'UVCAssistant:.*(stop|start) stream'

While that's running, open the Photo Booth app or any video conferencing software. You'll hopefully see lines like these output from the filtered log stream when the camera starts and stops (which you can trigger by quitting and restarting Photo Booth or toggling the camera).

```
2022-01-04 14:56:58.628006-0500 0xb35b3    Default     0x18025b             266    0    UVCAssistant: (UVCFamily) [com.apple.UVCFamily:device] UVCUSBDeviceStreamingInterface: 0x1000005a6 [0x7fcd7bd08260] [start stream] format : UVCDeviceStreamFormat:[1280 * 720 (YUV420_420v)] [0x7fcd7bd08e90] [subtype 4] frameInterval : 333333
2022-01-04 14:57:31.179027-0500 0xb2227    Default     0x1803a4             266    0    UVCAssistant: (UVCFamily) [com.apple.UVCFamily:device] UVCUSBDeviceStreamingInterface: 0x1000005a6 [0x7fcd7bd08260] [stop stream] format : UVCDeviceStreamFormat:[1280 * 720 (YUV420_420v)] [0x7fcd7bd08e90] [subtype 4] frameInterval : 333333
```

**If you get output, you're done with this step** and can use the the regular expression matchers I provide in the next steps.

**If you get no output**, you'll need to broaden the filter to search for event logs that appear when the camera starts and stops. This might be a good starting filter for your search:

    log stream | /usr/bin/grep -iE 'UVCAssistant|camera|stream|tccd'

For example, according to [this AskDifferent question][question], an earlier version of MacOS instead emitted event log messages containing `Post event kCameraStreamStart` and `Post event kCameraStreamStop`.

Ensure the events you choose are emitted for any camera usage (e.g. in Zoom and Meet) and not just Photo Booth.

You'll need to come up with a regular expression for `grep` that matches only when your camera is toggled on or off. Modify your `grep` regular expression until you get (ideally) a single line of output when the camera turns on or off. Beware of anything too generic, like `stream`, which may trigger your automation when other stream events happen like activating a Bluetooth microphone stream when the microphone turns on.

If you have multiple cameras, you may also want to test switching between cameras.

[question]: https://apple.stackexchange.com/a/424794/30953

Create Shortcuts to trigger HomeKit actions
---

First, confirm that you're able to use the Home app on your Mac to trigger changes to whatever accessories or scenes you're interested in.

In the Shortcuts app, create a new shortcut (`+` button). For toggling a light, you'll need a separate shortcut for each state (e.g. on/off). Give the shortcut a name such as _Turn on lamp_.

For each shortcut, select the _Home_ app and add its _Control \<home name\>_ action to your shortcut (drag or double-click). Select the specific scene or accessory you want to control, and select a state for you to set it to when the shortcut runs.

[![shortcut configuration][1]][1]

When complete, you should see your shortcuts in your shortcuts list and be able to run them and trigger your expected behavior.

[![both shortcuts][2]][2]

Last, confirm that you can trigger these successfully on the command line via Terminal:

```
shortcuts run 'Turn on lamp'
shortcuts run 'Turn off lamp'
```

[1]: https://i.stack.imgur.com/9uSOD.png
[2]: https://i.stack.imgur.com/3tm8i.png

Trigger shortcuts on camera state events
---

Create a script file (e.g. `camera-lamp.sh`) that, while running, will toggle your light based on whether the camera is running. A starter example:

```
#!/bin/bash
exec log stream |
  /usr/bin/grep -E --line-buffered 'UVCAssistant:.*(stop|start) stream' | # filter log events
  tee /dev/stderr |                           # output matching events for debugging
  /usr/bin/sed -Eu 's/.*(start|stop).*/\1/' | # reduce the log message down to a single word identifying the event/state
  while read -r event; do                     # store that word in the $event variable
    echo "Camera $event"
    if [ "$event" = "start" ]; then
      echo "Lamp on"
      shortcuts run 'Turn on lamp' &
    else
      echo "Lamp off"
      shortcuts run 'Turn off lamp' &
    fi
  done
```

Replace my shortcut names `Turn on lamp` and `Turn off lamp` with your own, if different. Feel free to alter messages like `"Lamp on"` to fit your scenario.

If you needed to choose a different filter in step 1:
1. replace the `grep` regular expression with one that matches both your camera on and camera off event log messages,
2. replace the `sed` substitution regular expression with one that matches the equivalent word that indicates the camera state, and
3. if that word is not `start` when the camera turns on, update the `if` conditional that matches the `start` word.

Run the script
---

Run the script, and test it out. If you named it `camera-lamp.sh`, run

    bash camera-lamp.sh

While the script is running, your HomeKit shortcuts should trigger whenever your camera turns on or off.

Optional steps:

- Once the script is stable, run it in the background, and close the terminal.

      bash camera-lamp.sh &

- Drop the extension, make the file executable, and relocate it to `/usr/local/bin` or elsewhere in your `$PATH`, so you can just run `camera-lamp` as a command.
- Find a way to run the script in the background as a daemon on login. How to accomplish this is beyond the scope of this answer. Importantly though, this cannot run as a launchctl daemon, as these daemons cannot run shortcuts due to `Error: Couldn’t communicate with a helper application`. If you find a good approach, feel free to comment.


With that running… Magic! My desk lamp automatically turns on whenever I join a
meeting or a [Roundsy][] social mixer (a free app we built for virtual event
attendees to chat with other random attendees in small groups).

Next up: Light up a sign _outside_ my office to signal my kids not to walk into
my meetings! 😃

[Roundsy]: https://roundsy.com
