---
layout: post
title:  "Arduino Powered Standing Desk"
date:   2025-10-12 12:12:12 -0000
categories: p
published: true
---

I made my *standing desk* controllable with an *Arduino*.

---

This is the process I went through to get my standing desk controllable with an Arduino! I have an Uplift desk from around 2022. If you are inspired to do something similar then *have no fear*! - you can control your desk using any computer, not just an Arduino.

## Why?
Simply this: I have a goal to stand more throughout the day. And also I know that I'm lazy; it's better to have the desk raise automatically than rely on the myself pre-caffiene to have the will-power to raise the desk in the morning.

My goal is to have the standing desk raise automatically sometime in the night so that when I start my day it is standing. And also to have *fun* along the way!

## Background Info and Planning
After paroozing the internet, I found that many people have already accomplished programatically controlling their standing desks. The solutions came in two flavors: bluetooth and hard-wired. I chose to use a bluetooth solution. The hard-wired option seems a lot more likely that I would get the magic smoke from the desk which would be a costly scenario. And I don't have a sautering iron yet!

Therefore I ordered the [UPLIFT Desk Bluetooth Adapter](https://www.upliftdesk.com/bluetooth-adapter-for-uplift-desk/?) (I refer to this as the *adapter*) which connects to an RJ12 port in the desk, right next to the RJ45 port the existing desk control buttons uses. Officially the only way to use the *adapter* is with the iPhone or Android app. The app has extremely simple functionality.

Luckily there are a handful of repositories online that have already reverse engineered the *bluetooth comunication* to talk to the *adapter*. With this information at hand, I chose a *microcontroller* to send commands over bluetooth to the adapter.

I chose the [Arduino Nano RP2040 Connect](https://docs.arduino.cc/hardware/nano-rp2040-connect/) because it has bluetooth and wifi. I didn't put much more time into this decision because I know Arduino, and if this board didn't work out then I'm sure I could use it for something else in the future. This is also my first time using a microcontroller (but am comfortable in the C/C++/embedded world).

## Step 1 - Explore the Repos
While waiting for the packages to arrive I took a look at the following repos: [uplift-ble](https://github.com/librick/uplift-ble) (Python) and [uplift-ble-helper](https://github.com/mdisibio/uplift-ble-helper) (Go). I explored around the code and converted both to executables, using `pyinstaller` and `upx` for Python, and the regular `go build` and `upx` for Go.

Side node: [UPX](https://upx.github.io/) is a super cool tool!

I got the Python executable down to 13 MiB and the Go executable down to 3 MiB, then down to 1.1 MiB after removing out the Prometheus parts.

I was under the impression that I could upload binaries to the Arduino. I didn't end up trying or doing this, I used the Arduino IDE and sketch to create a binary. It is a pretty simple use-case so no need to get fancy.

## Step 2 - Verify on the Desktop
The packages arrived! First I plugged in the adapter to the desk, and verified the phone app can connect to and control the desk.

Next I tried to use the repositores *uplift-ble* and *uplift-ble-helper* (plus a couple of others) and *none of them worked*! They could not connect to the adapter.

## Step 4 - Learn More About the Adapter
Eventually (with lots of help from [this issue](https://github.com/Bennett-Wendorf/hass-uplift-desk/issues/4)) I looked at `bluetoothctl devices` and figured out the **bluetooth service UUID** for my adapter is different than what the repositories expect. This UUID identifies the type of device. Mine is `000000ff-0000-1000-8000-00805f9b34fb`, but the repos were expecting `0000ff12-0000-1000-8000-00805f9b34fb` or `0000fe60-0000-1000-8000-00805f9b34fb`.

So this means there are *at least three revisions of the adapter*, hopefully nothing major changed and I can utilize the existing repositories!

After I changed the service UUID in the repos I did finally get a successful bluetooth connection, but got further errors. One repo gave the error:
```
BleakCharacteristicNotFoundError: Characteristic 0000fe62-0000-1000-8000-00805f9b34fb was not found!
```

Not knowing anything about bluetooth, I learned the bare minimum about services, characteristics, and descriptors. Then I listed out all of these for my adapter:

- service000a `00001801-0000-1000-8000-00805f9b34fb` Generic Attribute Profile
- service000b `0000ff00-0000-1000-8000-00805f9b34fb` Unknown
    - char000c `0000ff01-0000-1000-8000-00805f9b34fb` Unkown
    - char000e `0000ff02-0000-1000-8000-00805f9b34fb` Unknown
        - desc0010 `00002902-0000-1000-8000-00805f9b34fb` Client Characteristic Configuration
    - char0011 `0000fe63-0000-1000-8000-00805f9b34fb` Connected Yard, Inc.
        - desc0013 `00002902-0000-1000-8000-00805f9b34fb` Client Characteristic Configuration
    - char0014 `0000fe64-0000-1000-8000-00805f9b34fb` Siemens AG
        - desc0016 `00002902-0000-1000-8000-00805f9b34fb` Client Characteristic Configuration
- service0017 `0000180a-0000-1000-8000-00805f9b34fb` Device Information
    - char0018 `00002a29-0000-1000-8000-00805f9b34fb` Manufacturer Name String
    - char001a `00002a24-0000-1000-8000-00805f9b34fb` Model Number String
    - char001c `00002a25-0000-1000-8000-00805f9b34fb` Serial Number String
    - char001e `00002a27-0000-1000-8000-00805f9b34fb` Hardware Revision String
    - char0020 `00002a26-0000-1000-8000-00805f9b34fb` Firmware Revision String
    - char0022 `00002a28-0000-1000-8000-00805f9b34fb` Software Revision String
    - char0024 `00002a23-0000-1000-8000-00805f9b34fb` System ID
    - char0026 `00002a2a-0000-1000-8000-00805f9b34fb` IEEE 11073-20601 Regulatory Cert. Data List
    - char0028 `00002a50-0000-1000-8000-00805f9b34fb` PnP ID

Looks good, but what do these all mean? The repo was looking for characteristic `0000fe62-0000-1000-8000-00805f9b34fb` which my adapter doesn't have. So which one should I choose? I probably could have tried each characteristic to see what worked, but I did this instead:

## Step 5 - The Android App
Luckily [librick](https://github.com/librick) gave great instructions on [this PR](https://github.com/librick/uplift-ble/pull/2#issuecomment-3157223949) to go through the Android app to get information about what each of the bluetooth services and characteristics mean and do.

I followed their instructions, but to my suprise the decompiled APK didn't have anything about my specific version of the adapter.

After sleeping on it I figured to check the version of the APK. First I used an old Android phone (thankfully I had one!) and got version `1.1.1` from the Play store. The app could control the desk. The APK I was looking at previously was version `1.0.1`.

So I downloaded a newer APK, version `1.1.0`. Thankfully this new decompiled APK has references to my adapter's service UUID, and I got some hints about the characteristics that control the desk:

- device UUID `00ff` corresponds `service4` in the APK
- "InCharacteristic" is `0000ff02-0000-1000-8000-00805f9b34fb`. Looking through the existing repositories I think this is the "desk height" characteristic. I don't need this for my use case, but would need it if setting the desk to a particular height. I use the "go to preset 1/2" command.
- "OutCharacteristic" is `0000ff01-0000-1000-8000-00805f9b34fb`. This is the main characteristic to send control commands to the desk.
- "BTnameCharacteristic" is `0000fe63-0000-1000-8000-00805f9b34fb`. I don't think I need this one.

Side note: the decompilation uses [jadx](https://github.com/skylot/jadx), another impressive tool!

## Step 6 - It Works! (Desktop)
With this information I edited the existing repos and finally got *one* to work. Both *uplift-ble* and *uplift-desk-controller* did not work with the changed UUIDs, but [uplift-desk-controller](https://github.com/Bennett-Wendorf/uplift-desk-controller) worked! Yay!!!

Next I make a simplified version of uplift-desk-controller [which you can find here](https://github.com/calvincramer/uplift-desk-ble-ctrl/blob/main/desk.py). This is the script that can control the desk from any computer. I had lots of trouble trying to use *pipenv* (seriously, [why is python packaging so hard](https://calvinlc.com/p/2025/06/10/thank-you-and-goodbye-python.html)), so I converted to a `requirements.txt` and recommend people use *conda*.

This python script just runs commands. It does not wait until a certain time to raise the desk. Use *cron* if that's what you want! My computer is not on 24/7 so this wouldn't work for me.

## Step 8 - Run it on Arduino
Here's the final [Arduino sketch](https://github.com/calvincramer/uplift-desk-ble-ctrl/tree/main/desk-arduino-rp2040-connect). I ran into some roadblocks along the way. Despite the obstacles, it now works great! It's been three days so far that it works perfectly.

Here's the steps and issues I faced:
- first off I tried connecting to the device over bluetooth
- I ran into an issue where the bluetooth connection was super slow or would not work reliably. I fixed it by removing a delay in the loop while scanning for devices. It turns out that calling `BLE.available()` more frequently just works way better. I was under the impression that the bluetooth scan would buffer any devices that it found, and `BLE.available()` just queries if any results. Apparantly that's not the case. Very weird behavior. Just busy wait in a loop without a delay.
- I added WiFi to call an NTP server to get the current time
- after adding the Wifi, I got an issue where the bluetooth no longer worked. Turns out you can only use one at a time, not both.
- there was another issue where using the RGB LED during bluetooth mode causes the WiFi connection to fail (after sending control command to desk and switching back to WiFi mode). Turns out you can't use the RGB LED in bluetooth mode :(

More funny details
- the first night I tried it didn't work. I was powering the device using micro-usb and the power brick I was using was broken. Make sure the green light on the Arduino is on! That means it has power!
- I copied some crazy manual UDP networking code to get the time from an NTP server (which worked perfectly). It turns out that the WifiNina library has a [getTime() function](https://docs.arduino.cc/libraries/wifinina/#%60WiFi.getTime()%60) that does exactly what I want.

IMO the limitations around WiFi bluetooth and the LED should be made more clear in the documentation, under something like a "Limitations" section. These limitations are mentioned in the documentation but it's buried deep.

Finally the Arduino works perfectly!

## Step 9 - Add a Case
Last step is to add a case and give the Arduino a home. The case I got for a different (older?) Nano version. After some dremmeling and drilling I got it to fit perfectly, even the reset button too! I made sure that the case wasn't putting any pressure on the small components on the board.

## Conclusion
I've very happy to see the desk raised in the morning the last few days. It makes me feel competent seeing my code affect the real world, in a way that directly improves my health and takes ownership of my devices.

Thankfully the *adapter*'s bluetooth protocol is simple and not obfuscated or encrypted at all. And thankfully the Android APK allows to easily glean some meaning about the various bluetooth characteristics and even the message formats!

I'm greatful that others have done the actual hard work in reverse engineering the bluetooth protocol. Below are links to the repos I used.

In the future, I may like to try a hard-wired solution using the RJ12 port, raise the desk at other times (maybe randomly lol), and since the RP2040 has a microphone add voice commands like "raise!" and "lower!".

## Links
- My python and Arduino sketch - https://github.com/calvincramer/uplift-desk-ble-ctrl
- uplift-desk-controller - https://github.com/Bennett-Wendorf/uplift-desk-controller
- uplift-ble - https://github.com/librick/uplift-ble
- uplift-ble-helper - https://github.com/mdisibio/uplift-ble-helper
- hass-uplift-desk - https://github.com/Bennett-Wendorf/hass-uplift-desk
- uplift-reconnect - https://github.com/justintout/uplift-reconnect
