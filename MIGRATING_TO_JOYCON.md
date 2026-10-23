# Moving your own Shadow MD to Joy-Cons

This guide is for you if your droid already runs a Shadow MD sketch with PS3 Navs (Eebel's, KnightShade's, or your own version of either) and you want to drive it with two Nintendo Switch Joy-Cons. **You keep your sketch.** Your button map, your MarcDuino commands and your tuning stay as they are: you add six files and change a handful of lines.

**No new hardware.** The same Mega ADK, the same Bluetooth dongle, the same Sabertooth and SyRen.

**If your sketch is Eebel's Shadow MD and you never changed it**, there is a shorter way: use the sketch in this repo, `Shadow_MD_Eebel_Tabbed`, copy your own settings into its `a_INIT.ino` (controller MACs, Sabertooth and SyRen settings, speeds), and go to step 7.

> **Before you start.** Bench-tested only so far: nothing has been driven on a droid yet. Make a copy of your working sketch folder, keep it, and do the first runs with the wheels off the ground.

---

## 1. Back up

Copy your whole sketch folder somewhere safe. If anything goes wrong, that copy is your droid as it was this morning.

## 2. Install the right library

Joy-Cons need **USB Host Shield Library 2.0, version 1.7.0 or newer** (older releases do not have `SwitchProBT`). PS3 Navs do **not** work with 1.7.0, so keep your old library too:

1. Close the Arduino IDE.
2. In `Documents/Arduino/libraries/`, move your current `USB_Host_Shield_2.0` folder **out** of `libraries`, for instance to `Documents/Arduino/USB_Host_Shield_2.0_for_Navs`. Do not just rename it inside `libraries`: the IDE would still find it.
3. Open the IDE, go to *Library Manager*, search for **USB Host Shield Library 2.0** and install the latest version.

To go back to the Navs, swap the two folders again.

## 3. Add the six files

Download these files from [`Shadow_MD_Eebel_Tabbed`](Shadow_MD_Eebel_Tabbed) in this repo and put them in your sketch folder, next to your `.ino`:

| File | What it does |
|---|---|
| `ControllerConfig.h` | Choose the controller. The only one you edit. |
| `DroidController.h` | The controller interface. |
| `PS3NavController.h` | The PS3 Nav, so you can switch back with one line. |
| `JoyConController.h` | The Joy-Con. |
| `DroidPad.h` | Answers the old `PS3BT` calls, so the rest of your sketch does not change. |
| `JoyConPairing.h` | Pairing without re-uploading. |

Then open `ControllerConfig.h` and select the Joy-Cons:

```c
//#define USE_PS3_NAV
#define USE_JOYCON
```

## 4. Replace the two controllers

Find these two lines (in Eebel's sketch they are in `a_INIT.ino`):

```c
PS3BT *PS3NavFoot=new PS3BT(&Btd);
PS3BT *PS3NavDome=new PS3BT(&Btd);
```

Replace them with this block. It must come **after** the line `BTD Btd(&Usb);`:

```c
#include "ControllerConfig.h"
#include "DroidController.h"
#include "PS3NavController.h"
#include "DroidPad.h"

#ifdef USE_PS3_NAV
  PS3NavController navFootImpl(&Btd, "FOOT", LED1);
  PS3NavController navDomeImpl(&Btd, "DOME", LED2);
  DroidPad  footPad(&navFootImpl, &navFootImpl);
  DroidPad  domePad(&navDomeImpl, &navDomeImpl);
#else
  JoyConController joyFootImpl(&Btd, JOYCON_LEFT,  "FOOT", JOYCON_PAIRING_MODE);
  JoyConController joyDomeImpl(&Btd, JOYCON_RIGHT, "DOME", JOYCON_PAIRING_MODE);
  DroidPad  footPad(&joyFootImpl);
  DroidPad  domePad(&joyDomeImpl);
#endif

DroidPad *PS3NavFoot = &footPad;
DroidPad *PS3NavDome = &domePad;

#ifdef USE_JOYCON
  #include "JoyConPairing.h"
  JoyConPairing joyPairing(&Btd, &joyFootImpl, &joyDomeImpl);
#endif
```

Then search the whole sketch for **`PS3BT*`** and change every one to **`DroidPad*`**. In Eebel's sketch there are three, all in `d_FUNCTIONS.ino`:

```c
boolean ps3FootMotorDrive(DroidPad* myPS3 = PS3NavFoot)
int ps3DomeDrive(DroidPad* myPS3 = PS3NavDome)
void ps3ToggleSettings(DroidPad* myPS3 = PS3NavFoot)
```

If you wrote functions of your own that take a `PS3BT*`, change those too.

## 5. Start them in `setup()`

In `setup()`, just **before** the line `PS3NavFoot->attachOnInit(onInitPS3NavFoot);`, add:

```c
    PS3NavFoot->begin();
    PS3NavDome->begin();
    #ifdef USE_JOYCON
      joyFootImpl.setStickRange(JOYCON_STICK_RANGE);
      joyDomeImpl.setStickRange(JOYCON_STICK_RANGE);
      joyFootImpl.setGrip(JOYCON_GRIP);
      joyDomeImpl.setGrip(JOYCON_GRIP);
      joyPairing.begin([](JoyConPairEvent e)
      {
        const char* sound = JOYCON_SOUND_STARTED;
        switch (e)
        {
          case JCP_STARTED:         sound = JOYCON_SOUND_STARTED; Serial.println(F("JOYPAIR started: hold sync on a Joy-Con")); break;
          case JCP_PAIRED_ONE:      sound = JOYCON_SOUND_PAIRED;  Serial.println(F("JOYPAIR paired one, now the other")); break;
          case JCP_DONE:            sound = JOYCON_SOUND_DONE;    Serial.println(F("JOYPAIR done")); break;
          case JCP_TIMEOUT:         sound = JOYCON_SOUND_TIMEOUT; Serial.println(F("JOYPAIR timed out")); break;
          case JCP_HOLD_SYNC_AGAIN: sound = JOYCON_SOUND_AGAIN;   Serial.println(F("JOYPAIR hold sync again on that Joy-Con")); break;
        }
        Serial1.print(sound);
        Serial1.write(13);   // MarcDuino commands end with a carriage return
      }, JOYCON_PAIRING_MODE);
    #endif
```

The sounds go out on `Serial1`, the port Shadow uses for the dome MarcDuino. If yours is on another port, change those two lines.

## 6. Three small changes in the functions

**a. Read the controllers every loop.** In `readUSB()`, right after `Usb.Task();`, add:

```c
     DroidPad::updatePair(*PS3NavFoot, *PS3NavDome);

     #ifdef USE_JOYCON
       // While pairing, the droid does not move and the controllers are not read.
       joyPairing.update();
       if (joyPairing.active())
       {
           if (!isFootMotorStopped) { ST->stop(); isFootMotorStopped = true; footDriveSpeed = 0; }
           if (!isDomeMotorStopped) { SyR->stop(); isDomeMotorStopped = true; }
           return false;
       }
     #endif
```

If your sketch names the motor objects differently from `ST` and `SyR`, use your names.

**b. Skip the PS3 MAC check.** In `onInitPS3NavFoot()` the sketch only accepts the controller whose MAC address is in your settings. Those are your Navs' addresses, so it would drop every Joy-Con. Change:

```c
    if (btAddress == PS3ControllerFootMac || btAddress == PS3ControllerBackupFootMac)
```

to:

```c
    #ifdef USE_JOYCON
    if (true)
    #else
    if (btAddress == PS3ControllerFootMac || btAddress == PS3ControllerBackupFootMac)
    #endif
```

Do the same in `onInitPS3NavDome()` with its `PS3ControllerDomeMAC` line. With Joy-Cons, pairing is the gate: a Joy-Con only calls a droid it has been paired with.

**c. Move the over speed toggle.** A Joy-Con has no L3. In `ps3ToggleSettings()`, change:

```c
    if (myPS3->getButtonPress(L3) && myPS3->getButtonPress(L1) && isStickEnabled)
```

to:

```c
    #ifdef USE_JOYCON
    if (myPS3->getButtonPress(PS) && myPS3->getButtonPress(L2) && isStickEnabled)
    #else
    if (myPS3->getButtonPress(L3) && myPS3->getButtonPress(L1) && isStickEnabled)
    #endif
```

On the Joy-Con that is **stick click + ZL**. If you have other functions of your own on L3, move them to a free combination: L3 cannot be pressed on a Joy-Con.

## 7. Compile and upload

Compile for **Arduino Mega ADK**. If the compiler complains that `DroidPad` has no member called something, your sketch uses a `PS3BT` call this guide does not cover yet: open an issue with the error message and we will add it.

## 8. Pair, on the bench first

With the droid's motors powered **off**, or the wheels off the ground:

1. Power up. With no Joy-Con recorded, the droid opens a pairing session by itself and plays a sound.
2. Hold the small **sync button** on the rail of the **left** Joy-Con until it connects and vibrates. Then the **right** one.
3. From now on, press any button on each Joy-Con and it calls the droid back. After a disconnect, wait about ten seconds first.

To pair again later, hold **Minus + Capture** (left) or **Plus + Home** (right) for 5 seconds, or reset the Mega three times in a row. If the droid asks you to sync again, that Joy-Con still remembers a Switch: hold sync once more. The full guide, with the sounds and LEDs, is in [the ShadowMD Lite repo](https://github.com/Fernando-Beltran/ShadowMDLite/blob/main/JoyCon/README.md#pairing).

## 9. Check before you drive

- Both sticks move the right things: the **left** Joy-Con drives the feet, the **right** one turns the dome. Whichever connects first, they always end up that way.
- The button map: print [`CheatSheet_JoyCon.html`](https://github.com/Fernando-Beltran/ShadowMDLite/blob/main/JoyCon/CheatSheet_JoyCon.html). Minus and Plus are the old CROSS, Capture and Home the old CIRCLE, the stick click the old PS.
- **The watchdog.** Drive slowly, wheels off the ground, and turn the foot Joy-Con off with its sync button. The motors must stop.

## Going back to the Navs

Comment `USE_JOYCON` and uncomment `USE_PS3_NAV` in `ControllerConfig.h`, put your old library back (step 2), and upload. Everything else you changed keeps working with the Navs.
