#Faust2Webaudio



##What is Faust

![llustration by Harry Clarke for Goethes Faust](img/faust-classic.png)


[FAUST](http://faust.grame.fr/) (Functional Audio Stream) is a functional programming language specifically designed for real-time signal processing and synthesis. FAUST targets high-performance signal processing applications and audio plug-ins for a variety of platforms and standards.


Simply put, Faust lets one program dsp code once in a purely functional language, and compile it to various platforms including [max/msp](http://cycling74.com/products/max/), [supercollider](http://supercollider.sourceforge.net/), [audio unit](https://en.wikipedia.org/wiki/Audio_Units), [vst](https://en.wikipedia.org/wiki/Virtual_Studio_Technology), and more.



##What is the Web Audio Api?
![HTML 5](img/h5_logo.png)


###The [Web Audio Api](https://dvcs.w3.org/hg/audio/raw-file/tip/webaudio/specification.html) is a high-level JavaScript API for processing and synthesizing audio in web applications.


The Web Audio API comes with a number of natively compiled audio nodes capable of doing quite a bit of advanced synthesis.

You can check out Hongchan's [WAAX](https://github.com/hoch/waax) library for an example of extensive work being done with native nodes.


But What if you want something more?



I present to you
##The ScriptProcessor


The ScriptProcessor node allows allows individuals to create their own web audio nodes in pure JavaScript.  This allows individuals to extend the Web Audio Api with custom nodes.


Web Audio Libraries such as [Flocking](flockingjs.org) by Colin Clark and [Gibber](http://www.charlie-roberts.com/gibber/) by Charlie Roberts make extensive use of the ScriptProcessor node.


========
#WARNING
======== 

Currently native Web Audio nodes and ScriptProcessor nodes don't play so nicely together. As such, most implementations of Web Audio tend to pick one or the other.  

So what do you want, stability or the bleeding edge



##What does faust look like?

Below is an example of Noise Written in Faust 

```
random  = +(12345)~*(1103515245);
noise   = random/2147483647.0;
process = noise * vslider("Volume[style:knob]", 0, 0, 1, 0.1);
```


##What does a WebAudioNode look like?
Below is an example of White Noise taken from Flocking

```
flock.ugen.whiteNoise = function (inputs, output, options) {
    var that = flock.ugen(inputs, output, options);

    that.gen = function (numSamps) {
        var out = that.output,
            i;

        for (i = 0; i < numSamps; i++) {
            out[i] = Math.random();
        }

        that.mulAdd(numSamps);
    };

    that.onInputChanged = function () {
        flock.onMulAddInputChanged(that);
    };

    that.onInputChanged();
    return that;
};
```



##But Doesn't Faust Already compile to web audio?


##Sure


##But does it work?
[Current Faust2Webaudio Noise](examples/current-noise.html)



##Why did that break?


There is only one answer...


#JavaScript



##Before we start a flame war
![Flame War](img/flame_war.gif)


#I freaking love JavaScript


##But there are some things it can't do... 


#Like Integer Arithmetic



###asm.js and typed arrays to the rescue!

[![asm.js](/img/asmjs.jpg)](http://asmjs.org/)


asm.js is a strict subset of JavaScript that can be used as a low-level, efficient target language for compilers. The asm.js language provides an abstraction similar to the C/C++ virtual machine: a large binary heap with efficient loads and stores, integer and floating-point arithmetic, first-order function definitions, and function pointers.



##What does an asm.js
##WebAudioNode look like?
An example from the asmjs Flocking Branch

```JavaScript
flock.ugen.asmSin.module = function (stdlib, foreign, heap) {
    "use asm";

    var sin = stdlib.Math.sin;
    var pi = 3.14159;
    var out = new stdlib.Float32Array(heap);

    function gen (numSamps, freq, phaseOffset, mul, add, sampleRate, phase) {
        numSamps = numSamps|0;
        freq = +freq;
        phaseOffset = +phaseOffset;
        mul = +mul;
        add = +add;
        sampleRate = +sampleRate;
        phase = +phase;

        var i = 0;

        for (; (i | 0) < (numSamps | 0); i = i + 1 | 0) {
            out[i >> 2] = +(sin(phase + phaseOffset) * mul + add);
            phase = +(phase + (freq / sampleRate * pi * 2.0));
        }

        return +phase;
    }

    return {
        gen: gen
    };
};
```



##So this is all great, but I don't want to hand roll asm.js like a sucker...



##Introducing
[![Emscripten](/img/emscripten.jpg)](http://emscripten.org/)


Emscripten is an LLVM to JavaScript compiler. It takes LLVM bitcode (which can be generated from C/C++ using Clang, or any other language that can be converted into LLVM bitcode) and compiles that into JavaScript, which can be run on the web (or anywhere else JavaScript can run).