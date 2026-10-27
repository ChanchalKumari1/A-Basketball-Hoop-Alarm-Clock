## Case Study: Building a Basketball Hoop Alarm Clock

A client came to us with a simple brief: make an alarm clock you can't snooze. Their idea? A mini basketball hoop mounted above the bed. When the alarm fires, a ball drops, and the only way to silence it is to sink a shot. Sounds fun — until you try it at 6:45 AM.

We took it from sketch to working prototype in about seven weeks. Here's what went into it.

## The Problem With Normal Alarms

Phone alarms are too easy to kill. You tap snooze half asleep and never remember doing it. The client wanted physical effort in the loop — something that forces you upright and awake. A basketball shot does that. You have to stand, aim, and react. By the time the ball goes through the net, you're awake whether you like it or not.

The catch: the hardware has to be reliable. An alarm that fails to fire, or fires when it shouldn't, is worse than no product at all. That shaped most of our engineering decisions.

## Components We Used

Here's the full bill of materials for the prototype:

- **ESP32-WROOM-32** — main microcontroller, handles logic, BLE, and the ball-release mechanism
- **DS3231 RTC** — real-time clock module with a CR2032 backup cell, keeps time accurate to ±2 minutes per year
- **SG90 micro servo** — releases the ball from its cradle when the alarm triggers
- **IR break-beam sensor** — detects the ball passing through the hoop
- **Piezo buzzer** — the actual alarm sound, driven through a transistor
- **3.7V 2000mAh LiPo battery** with TP4056 charging circuit
- **Custom PCB** — designed in KiCad, roughly 60mm × 40mm
- **3D-printed enclosure** — hoop mount, ball cradle, and electronics housing, printed in PETG

## Why the ESP32

We considered the nRF52 for its lower power draw, but the ESP32 won on two counts. First, we needed enough RAM and flash to run the BLE stack alongside the alarm logic without fighting for resources. Second, the ESP32's dual-core setup let us pin the timekeeping and alarm checks to one core and keep BLE handling on the other. That separation matters — a BLE connection event shouldn't be able to delay an alarm firing.

We wrote the firmware on the ESP-IDF framework rather than Arduino. The Arduino layer is fine for quick tests, but we wanted direct control over the RTC wake sources and the BLE stack configuration. The alarm logic runs on a hardware timer synced to the DS3231, so even if the main loop stalls, the alarm still fires.

## The DS3231 and Why We Didn't Trust the ESP32's Clock

The ESP32's internal RTC drifts. Left alone, it can lose several minutes a week depending on temperature. For an alarm clock, that's unacceptable. The DS3231 has a temperature-compensated crystal oscillator, which keeps it within a couple of minutes per year. It also runs off its own coin cell, so time survives a full battery drain.

The ESP32 reads the time from the DS3231 over I2C at boot and after every wake. We also built in a daily resync in case of any I2C hiccups. The DS3231's alarm output pin is wired to an ESP32 GPIO with a wake interrupt, so the chip can sit in deep sleep and still wake exactly on time. Deep sleep pulls the whole board down to about 40µA, which gets us roughly two weeks per charge with normal use.

## Bluetooth Low Energy for Setup

Nobody wants to set an alarm with two buttons and a seven-segment display. We used BLE so the phone app handles all configuration — alarm time, days of the week, volume, and how many shots count as a snooze.

The ESP32 runs a custom GATT service. The phone writes alarm settings to a characteristic, and the ESP32 stores them in NVS flash so they survive power loss. We kept the BLE connection short-lived: the app connects, pushes settings, and disconnects. The radio stays off the rest of the time. That alone saved us close to 15mA on average.

Pairing uses a simple passkey so a neighbor can't quietly change your alarm time.

## The App: React Native

We built the companion app in React Native so one codebase covers iOS and Android. The app does four things:

- Connects to the hoop over BLE and reads current settings
- Lets the user set alarm times and repeat days
- Shows a log of recent alarms and whether the shot was made
- Pushes firmware updates over BLE when needed

React Native's BLE libraries handled the characteristic read/write cycle without much fuss. The trickiest part was connection reliability on Android — some phones drop BLE links aggressively when the app goes to background. We worked around it by keeping the setup flow short and reconnecting on resume rather than trying to hold a persistent link.

## Making the Shot Count

The IR break-beam sensor sits across the hoop opening. When the alarm fires and the servo drops the ball, the firmware waits for the beam to break. A clean pass through the hoop counts as a made shot and silences the buzzer. A miss triggers a five-second delay and drops another ball — the cradle holds up to six.

We debounced the sensor in firmware to ignore brief interruptions, and added a fallback: if no shot is registered within 90 seconds, the buzzer volume ramps up. Nobody sleeps through that.

## PCB and Enclosure

The board is a four-layer design in KiCad. We kept the servo power on a separate trace from the ESP32 supply and added a bulk capacitor near the servo connector — servos draw current spikes that can brown out a microcontroller if you're careless. The antenna keep-out zone on the ESP32 module is clear of copper and plastic, which kept BLE range solid at around 10 meters through a wall.

The enclosure went through three 3D-printed revisions. The first cradle dropped balls at odd angles. The second held them too tightly. The third uses a curved lip and a slight backward tilt, and the ball releases cleanly every time. We printed in PETG for impact resistance — PLA cracked at the mount points after a few weeks of use.

## Where We Landed

The prototype fires within one second of the set time, holds that accuracy across a two-week battery cycle, and registered a made shot on the first attempt in 47 of 50 test alarms. The three misses were all test throws, not hardware faults.

If you're building a connected hardware product and need firmware, PCB design, and an app that all talk to each other, this is the kind of work we do. Teams often reach out to [hire IoT developer](https://digitalmonk.biz/hire-iot-developer/) after their first prototype stalls at the "it works on a breadboard" stage — that's usually where the real engineering starts. We're happy to pick it up from there.
