---
type: post
title: 'playing piano in linux'
date: '2021-04-26'
author: arges
tags:
- linux
- music
---

While on vacation I brought my small MIDI [keyboard][1] along so that the kids could easily practice piano. This keyboard does not generate sounds on its own, so it requires some other device to generate the sounds based on the MIDI output. I used my laptop running Linux to accomplish this with some minimal setup.

First you must plug the keyboard in via USB. If you launch the program and get sounds, great! If not you may have to manually connect MIDI via `aconnect`.

## MIDI Connection

To manually connect the MIDI device to the target application use `aconnect`. If you are using `qjackctl` or other sound/midi systems, this may work differently. 

To determine midi inputs I first ran the following:
```
$ aconnect -i
client 0: 'System' [type=kernel]
    0 'Timer           '
    1 'Announce        '
client 14: 'Midi Through' [type=kernel]
    0 'Midi Through Port-0'
client 24: 'LPK25' [type=kernel,card=2]
    0 'LPK25 MIDI 1    '
client 129: 'VMPK Output' [type=user,pid=280738]
    0 'out             '

```

To determine midi outputs I ran:
```
$ aconnect -o
client 14: 'Midi Through' [type=kernel]
    0 'Midi Through Port-0'
client 24: 'LPK25' [type=kernel,card=2]
    0 'LPK25 MIDI 1    '
client 128: 'VMPK Input' [type=user,pid=280738]
    0 'in   
```

To connect the LPK25 MIDI keyboard to VMPK for example, you can do:
```
aconnect 0:24 0:128
```

Another approach would be using `aconnectgui` which provides a graphical way to connect devices.
![Aconnect Gui](/images/aconnectgui.png)

## Software

You will also need software to play sounds. Below are a few options.

### VMPK

The Virtual MIDI Piano Keyboard or [VMPK][2] can generate MIDI sounds and display notes as they are played. One can install this via a normal package manager such as `apt install vmpk`.

![VMPK Keyboard](/images/keyboard.png)

### Qsynth

You can install `qsynth` using standard package managers. Then you must connect the MIDI device through the method above. It doesn't display notes, but has more flexibility in how the output sounds.

### WebMIDI on Chrome

If you are using chrome you may be able to use WebMIDI which allows you to use a webpage to play sounds. An example that works is using [Web MIDI Keyboard][3].

### PianoTeq

PianoTeq works on Linux machines and is a commercial product that has high quality keyboard samples. You can try a demo of the software to see how it works first.

[1]: https://www.akaipro.com/lpk25
[2]: https://vmpk.sourceforge.io/
[3]: https://www.onlinemusictools.com/kb/

