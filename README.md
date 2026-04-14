# OrangeJukebox750
I want an open and modular DJ environment, like modular synthesis but then with tracks and computer based, but not screen-based. It should be adaptable to the needs of other users too. I just get really annoyed by Traktor and especially Pioneer/Rekordbox building a closed ecosystem, no no no no nooh.

# Background
I have had allready one shot at making such a piece of software. I made it in Supercollider, and have done a few performances with it. But the basic structure of supercollider scheduling happening in the language side and then communicating to the audio thread over OSC proved to be annoying, as was the debugging experience.

# Features
Here is a list of ideas I have of things I'd like to include in my setup, ar at least be able to experiment with.

- playing digital audio files with control of speed, and knowledge of the grid,
- custom looping and beatjump mechanism, i.e. looping a phrase halfway the phrase by using grid information,
- tactile interactions, using midi but also alternative controllers like a barcode scanner and a joystick, and DVS (digital vinyl),
- audio effects and strange sound generators,
- advanced control over track selection with filtering by playlist and tags, perhaps using assistance from LLM,
- surround sound,
- on the fly parameter modulation,
- on the fly rearranging of effects.

# Philosophy
Whereas my previous approached rested on the idea that Supercollider was the one environment in which I could build all my ideas. I now try to find more established environments which suite a specific purpose, and tie them nicely together. So all the heavy lifitng is done by established tools, and I make small custom parts to let everything work together the way I like it. I have also considered modifying the [Mixxx](https://mixxx.org/) project. But this is not a direction which makes me enthusiastic, since a lot of time will be spent in finding out why the made things as they did (with a lot of legacy code), instead of building something myself. I also considered writing a standalone application in JUCE, but this will probably take more time, and there is less that I can use from others, and furthermore it will be less easy for others to adapt it to their situation.

# Concept and target audience
The core concept is to have a lean DJ tool which embeds in a DAW, and which is open to hack to your own liking. The targeted the more nerdy users. The users who like to mod something to their own liking, and don't mind to spent some time to set up the system (given that the system is well thought out, [example](https://lidarr.audio/)), in order to get a tool which is simple to use, does the job, is stable, and you can actual perform with. No fancy cool looking plugin gimmick.

# Existing DJ VST's
- https://www.stagecraftsoftware.com/products/DJs/, https://djtechtools.com/2013/08/08/scratch-track-scratch-with-timecode-in-any-daw/, https://www.kvraudio.com/forum/viewtopic.php?t=513753;
- https://plugins4free.com/plugin/2865/;
- https://mspinky.com/;
- https://plugins4free.com/plugin/207/;

# Minimal Architecture
## Components
The system will consist of the following components:
- a VST host, I will go for Reaper myself because it is highly customizable and has good support for surround;
- a DJ-deck plugin (VST) with support for DVS written in [JUCE](https://juce.com/). We can hopefully port some of the [xWax](https://xwax.org/) code, and make something like [MsPinky](https://mspinky.com/). Initial support for Serato timecode (because then Rane Twelve mk2 can be used). It will have a very minimal gui. Tracks should be loadable using an OSC message. Multiple plugin instances should be loaded on multiple tracks to allow for a multi-deck DJ experience;
- a Python program:
    - music library, supporting interaction using a barcode scanner (HID), communicating the selected track via OSC to the DJ deck VSTs. Initially support for Traktor NML using [traktor NML utils](https://github.com/wolkenarchitekt/traktor-nml-utils). Use [Beets](https://github.com/beetbox/beets) for track management;
    - other high level noncritical-timing task;
- music management takes place in third party software such as Rekordbox, Traktor, or [Lexicon](https://www.lexicondj.com/about).

## Communication
- All audio timing is done internally in the DAW / VST;
- All non-timing critical information is done using [OSC](https://opensoundcontrol.stanford.edu/index.html), this includes the cummuncation between the library and the dj decks. OSC is supported by many audio tools including [JUCE](https://juce.com/tutorials/tutorial_osc_sender_receiver/) and [Python](https://github.com/attwad/python-osc). Each deck has its own port on which it listens to OSC messages. The track loading message should include a description of the grid of the track.
- Ableton Link is used to modify the master clock from the VST, and can also be used to align other software later on (i.e. lighting).
- It is not so easy to make plugin [communicate](https://forum.juce.com/t/shared-variables-among-different-plugins/65687/12) directly to each other, variables are not exposed and its getting common for DAWs to sandbox plugins.
- There is "state information" which the plugin exposes to the daw. But the DAW only treats this as a single blob of information and cannot understand whats in there. It's used to save a snapshot of the current state of the plugin, for closing and opening the daw project.

## Overview
An overview of all the basic components looks like this:
![overview of minimal system](Architecture/Slide1.PNG)

# DJ deck plugin
## Architecture
All the mixing tools we leave to other plugins and DAW infrastructure. The DJ tool we develop consists of a DJ deck, of which one can use many. The DJ deck plays a long audio file, keeps it in sync, and allows for jumping to other positions, scratching and nudging. The plugin is controller oriented, meaning that gui design is less emphasized. The music library is seperate.

## Features
Basically the dj deck plugin plays an audio file from a given starting point at a given tempo. 

Crucial features:
- The DJ deck is fed with information on the grid of the track.
- It should be able to jump to other points in the track without glitches. We need two players between which we crossfade in order to avoid glitches. Looping is build as a function on top of beatjumping. 
- There are two tempo modes: internal and external. In external mode the tempo of the track is synced to the tempo of the daw (this information is natively available in the daw-plugin framework), in internal mode in runs by itself.
- The daw can be synced up with the track, i.e. the track can be set as master. This requires the plugin to tell the master the tempo. This is not natively supported. A solution could be to use Ableton Link.
- Using que points, at minimum one que point should be supported.
- Pitch bending support, for nduging a record forward or backward.
- jog wheel smooth speedy scrolling through track (so rotating the disk speeds up the record, without jumping). 
- Play, pause, que, tempo, beatjumping and pitchbending should be exposed to the daw, in such a way that it can be midi mapped in a meaningfull way.

Advanced features:
- modify the grid;
- DVS support;
- surround sound support;

## GUI
Minimal:
- static information of track;
- dynamic information of track;
- OSC port number (setable);

Advanced:
- Waveform display with grid overlay;

## Building
We build the software using [Juce](https://juce.com/). Using the CMake built system (this is now generally recommended over Projucer). 

# Library
The library allows for browsing, filtering and selecting a track, and forwarding this to the DJ deck plugins.

## Gui
A simple GUI can be made with [tkinter](https://docs.python.org/3/library/tkinter.html). More advanced GUI packages are also available.

## Building
We build it in Python. We can use a tool like [PyInstaller](https://pyinstaller.org/en/stable/), to make it easy for other users to get the library to work.

# Channel strip
This is what you need as a DJ to mix.
- volume;
- gain;
- DJ style filter ([DJMFilter](https://splice.com/plugins/4442-djmfilter-vst-au-by-xfer-records) or [Platone DJ Filter](https://platonestudio.com/product/dj-filter/));
- 3 band DJ style EQ ([DJEQ](https://deadducksoftware.github.io/free_effects_equalisers.html#djeq));
- crossfading utility 

# Extended Architecture
If we have any time left.
## Low Priority Features
- The DJ decks can sync the master playback to their current playback using ableton link; 
- Support for [DJ XML](https://djxml.com);
- Modulation signals and midi effects using Reaper JFSX and 14 bit MIDI;
- Pathbay and modulation support using Python Reascript to dynamically assign routing and modulation, based on midi and osc input, from amongst others a mock-up patchbay. In the mock-up patchbay each output port send a unique digital identifier (could for example just be the pulsewidth), and each input identifies what he gets fed. That way you can use physical wires to rewire software. For modulation we implement the concept that of you hold a "assign" button, and then wiggle two knobs; the second knob gets mapped to the first knob, perhaps using [MiditoReaControlPath](https://forum.cockos.com/showthread.php?t=43741) or ReaControlMIDI, to create a kind of proxy which allows to inject a modulation signal on top of a controller mapping, i.e. this proxy layer would be mapped to the VST parameters, and you would map you controller to the proxy, but this proxy is also listening to another midi channel on control path, on which you send your modulation signal;
- Open-source python based DJ music library software, with a good GUI, support for tagging, waveform visualization, grids;
- Support for HID game controllers, using a hid2midi converter;
- Per deck minimal waveform visualization using for example a Raspberry Pi, with the main purpose of visualizing the coming 16 bars, to see if an important phrase transition occurs.
- Audio effects written in Faust or JSFX;

## Overview
![overview of extensive system](Architecture/Slide2.PNG)