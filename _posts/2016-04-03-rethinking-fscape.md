---
layout: post
title: "Rethinking FScape and Audio Rendering"
description: ""
# category: ""
tags: [scala]
---
{% include JB/setup %}
{% include themes/twitter/default.html %}

## Preamble

I am currently working on a few new small pieces that are destined to be mosaic pieces in an upcoming exhibition "Imperfect Reconstruction" at the end of this year. One of them is based on a digitally processed video, and I have developed an idea of how sounds would go with this piece: Using a pool of synthetic sound structures "Mentasm" from an ongoing research in genetic programming of sounds, I have defined a simple process to pair an two elements X and Y from these pool as (X, Y) and (Y, X) and process these two tuples using [FScape's](https://github.com/Sciss/FScape) spectral patcher module.

Have done some manual rendering, what I would like to arrive at is a machine form of this. I imagine that in this exhibition there will be one or several rendering computers that generate ad-hoc sound structures. Ad-hoc means they could use physical world input, e.g. from a sensor, that will feed into the rendering process. That process does not necessarily have to run in so-called real-time, but "just in time". It means, an agent connected to this outside world input may issue jobs for the rendering computers. Those computers maintain a queue of jobs which they process with modules in the FScape sense, eventually injecting their output to the pool of sounds available for the real-time sound production.

While there is a simple library [FScapeJobs](https://github.com/Sciss/FScapeJobs), this is barely more than a hackish solution, sending FScape documents over the wire (OSC) and awaiting the result of the rendering. I have used FScapeJobs in several pieces, both installations and live performances. However, I would now like to integrate these processes more tightly with my ongoing effort of building a computer music framework [SoundProcesses](https://github.com/Sciss/SoundProcesses) and a graphical user interface [Mellite](https://github.com/Sciss/Mellite). FScape is a rather old software, having originally been written in Java 1.1 (then introducing Java 1.2 then 1.4 features, now compiling against Java 1.6) back in 2000/2001. One can imagine that the code base can be quite horrible. Here is a bit of my thought process of how a revision of FScape integrated into Mellite could look like.

## Decomposing

The first thing to note is that FScape modules are rather macroscopic when viewed from a signal processing perspective. Often the overall algorithm of a module is comprised of many individual steps and routines that, if they were dissembled, could greatly increase code reuse and the possibilty to quickly build entirely new modules.

Of course there are specific signal processing languages, e.g. [Faust](http://faust.grame.fr/about/) or [Nyquist](https://www.cs.cmu.edu/~rbd/doc/nyquist/index.html). But the model I'm currently thinking about is actually SuperCollider's UGen approach. UGens are black boxes with a number of inputs and outputs, and they can be instantiated with some static parameters such as the calculation rate. Let's review briefly the process mentioned above that I was using to generate some sounds for this video study: It is a simple spectral patch, using, sequentially, a forward STFT with large window size, a stream splitter, two percussion ops (an operator that manipulates the cepstral domain), a stream unitor, a volume adjustment, and an inverse STFT. At first sight, these would be five "FScape UGens". But just like on the client side, in [ScalaCollider](https://github.com/Sciss/ScalaCollider) we can have helper UGens that lazily expand to other UGens (eventually "real" ones), we can think of these as possible macros themselves. The analyser would decompose into the four steps "external input", "overlap-app-segmentation", "window application", "actual FFT". These could be the atoms of "FScape UGens".

Standard signal processing kits often have a rather simplistic view of signals, mostly they are analogous to real-time signals in that they are uniform and linearly progressing in time. FScape is fundamentally about the possibility to process in a non-linear fashion (although the mentioned example is indeed a linear one). Quite a few questions immediately pop up about the representation of the signals within this hypothetical new system:

- how to formalise random access? a UGen could declare
  - that it is something like a LTI (linear time invariant) process that simply processes whatever you throw at it
  - that is proactively supports random access from the "sink" side
  - that is requires linear progression; the system would then have to insert buffering/caching automatically if a "sink" wants random access
- push versus pull.
  - Mostly we think of push transport in signal processing, e.g. we start with an audio file and we throw chunks of data at the UGen until the file ends.
  - Yet, there may be cases where we want more of a functional "open music" kind of perspective, where the process is driven by a sink until a specific 
    condition occurs to terminate the process. Think of a `.contains` or `.takeWhile` collection view of a process
- channels
  - how to represent multi-channel streams?
  - Departing from the UGen metaphor, one might implement something like multi-channel expansion in SuperCollider
  - in fact, most FScape modules could be understood in this MCE fashion
  - however, this might be wasteful in terms of resources. We don't want to build auxiliary structures and filter coefficients etc.
    multiple times but only once. This would speak for building multi-channel support directly into the UGens
- numeric type
  - how about real versus complex?
- signal shape
  - how about one dimensional vectors versus multi-dimensional matrices?
  - how about chunking?
  - how about markers?

Of course, there are attempts to provide general representations and formats, e.g. [SDIF](http://sdif.sourceforge.net/standard/sdif-standard.html) or [NetCDF](https://www.unidata.ucar.edu/software/thredds/current/netcdf-java/CDM/index.html). In SPDIF, we have the model of time-sorted frames which are composed of matrices. On the other hand, it seems to have quite a few hard-coded assumptions such as the time format, doesn't have a category or string type etc.

## Chunks

Looking at the current FScape modules, most process sound in chunks. The spectral domain ones typically use the FFT frame as unit, the time domain ones usually use an arbitrary chunk size such as 8K sample frames. It would be great if we could define UGens that automatically support both types of chunks, so we don't have to write everything twice for FFT data and time domain data.

Since the processes will be decomposed, we might be needing to carry meta-data along the streams, e.g. FFT frequency scale information or time stamps. Perhaps there is a dictionary per stream or a dictionary per chunk. UGens could read or "inject" that meta-data as they go along.

## Lucre/SoundProcesses

There are two layers involved with the integration into Mellite. The first is the representation of modules or UGens in terms of [Lucre](https://github.com/Sciss/Lucre). What kind of data type _is_ a UGen? And what kind of input and output data types does it require. If we look again at the same example, we find:

- artifacts or audio-cue objects for the up-most source
- integer and floating point numbers for scalar parameters
- category parameters (e.g. window type)
- boolean switches
- I/O signals -- these we do not want to persist, they are a transitory data type; but like `Proc`'s `Output` and attribute inputs, we need a notion of "ports"

Indeed, taking `Proc` as the model, as incomplete and questionable it still is, we could arrive at a view, where an FScape "module" is analogous to a `SynthGraph`. Then we have a simple builder from "FScape UGens" which are persisted agnostically as products as is the case with ScalaCollider UGens. We talk to the world through an outer type analogous to `Proc` with it's object attribute map and outputs, and we through special ugens such as `Attribute.kr`.

The problem with this approach might be that we have to have much more differentiated types than `GE` for input and output. That would warrant a "full Lucre type" for each UGen. We could also go the dynamic/attribute map way. So there is `UGen <: Obj` and the parameters are again just in the attribute map. Then we need heuristics like in the Wolkenpumpe case, not really an attractive solution.

## Mellite

In Mellite, we could then have view factories? A general factory analogous to [Wolkenpumpe](https://github.com/Sciss/Wolkenpumpe)? Or specific factories that really reconstruct the FScape module panels? A graphical patcher for UGen graphs? 

The important bit here is that we need to be able to patch any parameter into the rest of a workspace, that is the entire point about this endeavour.

The outputs would be like a "rendered audio cue", or simply a regular audio-cue-variable with the module in its object attribute map.

If we go all the way to graphical patchers (as I have the feeling we will when the folder views are accompanied by a spatial patcher view)... then we could "build interfaces" to each module within Mellite itself. Including parameter ranges specs and so forth.

## Sound Installation

If we assume that an architecture is implemented as outlined above, how does this work inside a "Mellite composition"? Sure we could trigger renderings from an `Action` object. We could place another `Action` object as call-back for when the output has been rendered, or we could compose the module in terms of a lazy audio-cue (a "self-updating" expression so to speak)?

For example, let's imagine we have a `Folder` and then we have a `Select` operation in an attribute map of a `Proc`, assuming that it selects a correct type, say an audio-cue. Then the rendering could just add the result to that folder, using a done-`Action`. That action could also bind the folder's maximum size.

I.e. at some point we have a _circuit breaker_, something that transitions from functional to procedural or vice versa.

