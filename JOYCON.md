# Shadow MD with Joy-Con support

*The bench test sketch, the printable button map and the pairing details live with the rest of the Joy-Con work, in the [ShadowMD Lite repo](https://github.com/Fernando-Beltran/ShadowMDLite/tree/main/JoyCon). This file covers what changed inside Eebel's sketch.*

This is Eebel's Shadow MD, unchanged in behaviour, with one structural addition: the sketch no longer talks to the PS3 controller directly. It talks to a small controller interface, and behind that interface there is either a PS3 Move Navigation, exactly as before, or a single Nintendo Switch Joy-Con.

**Why.** Sony stopped making the Nav years ago. The buttons work, the sticks drift, and replacements get more expensive and harder to find every year. A single Joy-Con is the closest thing still in production: one hand, pocket size, analog stick, USB-C charging, and it speaks the same classic Bluetooth HID that the Navs use, so no new hardware is needed. Same Mega ADK, same Bluetooth dongle.

## How to switch controller

Edit `ControllerConfig.h`. That is the only file you need to touch.

```c
#define USE_PS3_NAV        // original behaviour, two PS3 Navs
//#define USE_JOYCON       // two Joy-Cons, left for the feet, right for the dome
```

## Pairing

No re-uploading and no editing.

- **The first time**, with no Joy-Con recorded, the droid opens a pairing session by itself at power up. Hold the small sync button on the rail of the left Joy-Con until it connects and vibrates, then on the right one. The second one is optional.
- **Every day**, press any button on each Joy-Con and it calls the droid back. After a disconnect, wait about ten seconds before pressing.
- **To pair again**, hold **Minus + Capture** (left) or **Plus + Home** (right) for 5 seconds, or reset the Mega three times in a row. While the session is open the droid does not move; it closes when both halves are connected, or after 60 seconds.
- **A Joy-Con that has been used on a Switch** is caught when it pairs: the droid asks you to hold sync again, and the second time it is accepted.

Each step plays an R2 sound, set in `ControllerConfig.h`. On connection the player LEDs alternate outer and inner for two seconds, then settle on LED 1 (left) or LED 2 (right). The full user guide, with the sounds and what each one means, is in [the ShadowMD Lite repo](https://github.com/Fernando-Beltran/ShadowMDLite/blob/main/JoyCon/README.md#pairing).

## What actually changed in the sketch

98 lines added, 5 removed, across three files, counting the pairing. That is the whole port.

| File | Change |
|---|---|
| `a_INIT.ino` | The two `PS3BT` pointers become two `DroidPad` pointers, built from whichever controller `ControllerConfig.h` selects. Plus the includes and, with Joy-Cons, the pairing session object. |
| `b_SETUP.ino` | Two `begin()` calls and, with Joy-Cons, starting the pairing session with its sounds. |
| `d_FUNCTIONS.ino` | Three function signatures change from `PS3BT*` to `DroidPad*`; `readUSB()` refreshes both controllers right after `Usb.Task()` and, while pairing, stops the motors; with Joy-Cons the over speed toggle is stick click + ZL and the PS3 MAC whitelist is skipped (it would drop every Joy-Con). |

Everything else is untouched. All 161 calls to `getButtonPress()`, `getAnalogHat()`, `PS3NavigationConnected` and the rest keep working exactly as they are, because `DroidPad` answers the same API.

That was the design goal. A port that rewrites every call site produces a diff nobody can review and a droid nobody trusts at an event.

## New files

| File | What it is |
|---|---|
| `ControllerConfig.h` | The only file you edit. Controller choice and Joy-Con options. |
| `DroidPad.h` | The compatibility shim: speaks the old PS3BT API, answers from the abstraction. |
| `DroidController.h` | The controller interface: button roles, normalised stick, click detection, watchdog. |
| `PS3NavController.h` | The PS3 Nav implementation. The reference: it must behave as before. |
| `JoyConController.h` | The Joy-Con implementation, built on `SwitchProBT`. |
| `JoyConPairing.h` | The pairing session: first boot, button combo, triple reset, the Switch check. |

## Requirements

**USB Host Shield Library 2.0: the version depends on the controller.** For Joy-Cons you need **1.7.0 or newer**, because older releases do not ship `SwitchProBT`. For PS3 Navs, **keep the older library**, the one in `USB_Host_ShieldEebel.zip` in this repo: the Navs do not work with 1.7.0. Keep both copies around and install the one that matches the controller you build for. A PS3 Nav build does not include any Joy-Con code, so it compiles fine against the older library.

## Button map

A Joy-Con has nine usable buttons, the same number as a Nav, but not the same nine, so one role has to give.

| Foot, left Joy-Con | Dome, right Joy-Con | Was on the Nav |
|---|---|---|
| d-pad | X / B / Y / A | d-pad |
| Minus | Plus | CROSS |
| Capture | Home | CIRCLE |
| L | R | L1 |
| ZL | ZR | L2 |
| stick click | stick click | PS |
| nothing | nothing | L3 |

L3 is the one that goes, because in the stock map almost nothing uses it: every cross controller L3 combination is FREE. The single function that moves is the over speed toggle, which was `L3 + L1` and is now `stick click + ZL`. Sacrificing CIRCLE instead would have cost eight working functions.

`SL` and `SR` on the rail are not exposed by the Switch Pro parser, and they are awkward to reach while gripping the controller anyway.

## Build sizes, Arduino Mega ADK

| Build | Flash | RAM |
|---|---|---|
| Original, untouched, older library | 56,548 | 4,738 (57%) |
| This fork, PS3 Nav, older library | 55,984 | 5,320 (64%) |
| This fork, Joy-Con, library 1.7.0 | 61,030 | 5,438 (66%) |

Each build is measured with the library it actually needs (see Requirements). The PS3 Nav build comes out slightly smaller in flash than the original. RAM goes up by about 600 bytes, which on a Mega leaves roughly 2.8 KB free. That is enough, but it is worth knowing if you have already added things of your own to this sketch.

## What is tested and what is not

- Both configurations compile clean. **Neither has been run on a droid yet.**
- `JoyConController.h` has passed the bench test with two real Joy-Cons connected at once: full scale on both sticks, all nine buttons on each half, link lag under 25 ms. The same file is used here.
- The Joy-Con implementation handles the switch to full report mode, stick centring on connection, drift recentring while the stick rests, detecting which half connected, the different button map of each half, and grip orientation.
- The order does not matter: whichever Joy-Con wakes up first, the left half drives the foot and the right half the dome (`DroidPad::updatePair()`). If only the right one is connected, it goes to the dome and the foot stays still. Tested on the bench with the right half connecting first.
- Pairing without re-uploading is bench-tested: first boot, the button combo, the 60 second timeout, triple reset, a Joy-Con docked on a Switch (rejected, then accepted after syncing again), and reconnecting by button after a reset or after turning a Joy-Con off. The bench test runs the same `JoyConPairing.h` used here.
- There is a bench test sketch, [`BenchTest`](https://github.com/Fernando-Beltran/ShadowMDLite/tree/main/JoyCon/BenchTest), that reads controllers and prints what it sees without driving anything. Run that first, with the droid powered down.

Do not take this to an event until you have driven it in your workshop.

## Credit

The Joy-Con protocol details come from dekuNukem's reverse engineering notes, https://github.com/dekuNukem/Nintendo_Switch_Reverse_Engineering. Thank you.

Original Shadow MD by Eebel, from https://github.com/Eebel/SHADOW_MD_EEBEL, built on KnightShade's SHADOW. The controller abstraction and the Joy-Con support come from the work on ShadowMD Lite, https://github.com/Fernando-Beltran/ShadowMDLite
