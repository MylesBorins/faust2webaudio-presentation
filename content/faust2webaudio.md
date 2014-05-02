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



##Faust -> C++
So using the faust compiler (specifically the faust2-asmjs branch) we can compile from [faust](https://gist.github.com/TheAlphaNerd/c904b82dae7f26782d29#file-noise-dsp) to [C++](https://gist.github.com/TheAlphaNerd/c904b82dae7f26782d29#file-faust-noise-cpp) with the following command
```
faust -a minimal.cpp -i -uim -cn Noise \
 dsp/noise.dsp -o cpp/faust-noise.cpp
```


In order to get access to the various parts of a C++ class via emscripten we need to write a simple wrapper on top of our Noise class.  You can find a copy of the wrapper on [github](https://gist.github.com/TheAlphaNerd/c904b82dae7f26782d29#file-faust-wrapper-cpp)

This wrapper makes meta functions that allow you to create and operate on objects using pointers, as emscripten does not have fantastic support for working with C++ objects.

We then pass the wrapper through sed to template the class name and concatenate to the end of the file compiled above
```bash
sed -e "s/DSP/NOISE/g" -e "s/Dsp/Noise/g" -e "s/dsp/noise/g" \
cpp/faust-wrapper.cpp >> cpp/faust-noise.cpp
```


We can then compile the resulting C++ file to asm.js using emscripten with the following command
```bash
emcc -O2 cpp/faust-noise.cpp -o js/faust-noise-temp.js \
-s EXPORTED_FUNCTIONS="['_NOISE_constructor','_NOISE_destructor', \ 
'_NOISE_compute', '_NOISE_getNumInputs', '_NOISE_getNumOutputs',  \
'_NOISE_getNumParams', '_NOISE_getNextParam']"
```

You will notice the reference to exported functions above.  This is how we let emscripten know which functions are going to be accessible in JavaScript so that their variable names are not obfuscated by the optimizer.

This will generate the [following JavaScript file](https://gist.github.com/TheAlphaNerd/c904b82dae7f26782d29#file-faust-noise-js)


Finally we apply the JavaScript wrapper (again templated via sed) to the compiled JavaScript in order to wrap the exported functions and generate an object literal with a programming interface similar to that found in the native WebAudio nodes.

You can find the wrapper [here](https://gist.github.com/TheAlphaNerd/c904b82dae7f26782d29#file-faust-wrapper-js)


#Phew...


#But does it work?


#Sure,
#[take a listen](/demo/index.html)


##But it is unnecessarily complicated



###Managing memory in C++ while maintaining persistence when compiled to JavaScript can be tricky.  


###Specifically because faust represents inputs and outputs as a float **


###That means the input and output variables are in fact a pointer to an array of channels


##Each channel in the array is a pointer to a buffer


##Each buffer is an array consisting of the samples represented as a floating point number between -1 and 1.


###This means we need to be using malloc in JavaScript to create variables in the heap which will then be used when making function calls into the c++ virtual machine to generate samples.  

####These pointers will then need to be dereferenced, again in JavaScript, sample by sample to copy the buffers over to WebAudio


Further, every faust project has a different number of controllable parameters.  While Faust currently offers a way to create a data structure that maintains all of the information about these parameters, including name and pointer value.


Unfortunately emscripten does not offer stable support for transferring the values of this generated data structure into JavaScript land.  

As such a meta UI class needed to be created to generate a map, which is then iterated across via calls from JavaScript.  

This iteration has a manual stack create from JavaScript which then injects pointers to grep out the data that lives on the heap to be accessible as string literals and numbers in JavaScript.


![wat?](img/wat.jpeg)



##This seems unnecessarily complicated


##Why should we be spending time managing these pointers?


###How are we supposed to deal with the pointers in JavaScript land without creating lots of garbage?



##But didn't I say Faust is Functional?


##Indeed it is


###So why are am I going from 

Functional -> Object Oriented -> Functional


![Clever Girl](img/clever-girl.png)



##Emscripten comes with its own virtual machine as well as a number of compilation optimizations


##Simply put, a bunch of performance for free!


##That made it a tempting starting point


##As a primary research goal is to create very efficient web audio based unit generators


##While we do not yet see the optimizing effects in Chrome, we can already see the benefits of using asm.js in the Firefox



##DEMO TIME!!!



#In Conclusion


##Things are still not perfect


##The browser is still not the perfect place to do signal processing


##Everything in JavaScript lives in a single thread... 


##... which is gross


#BUT


###Things will only get better with time.
####and the code compiled with Faust2Webaudio will only become more performant as browsers continue to build out support for asm.js and come up with better ways to handle audio 
(like a seperate thread *cough cough*)



##Oh yeah...
###my code isn't perfect yet


###I still have not yet perfected statically linking multiple files, and as such every unit generator has its own instance of the emscripten virtual machine...


#YUK



###Further, the current way of gathering parameters is far from elegant.  This could be improved by developing a new architecture file for faust, improving support for data structures in emscripten, and also re evaluating the current way I toss things between JavaScript land and the virtual machine.



##Finally the current compilation process leaves a lot to be desired.


###I love shell scripts and sed as much as the next neckbeard, but this is definitely not an ideal development environment.

###Moving forward I would like to get all of the C++ templating to live in the faust compiler via a custom architecture file.  


###I would then like to get all JavaScript related compilation (with emscripten) and templating to be handled with a node.js based development stack.  Specifically Yeoman for scaffolding new projects and grunt for task automation


###My vision is that you could drop any Faust .dsp file into a folder and the tool chain should automate the rest for you.



##I truly believe that the web 
##is the ultimate open and free platform


##Wether or not you like JavaScript
###The code will run anywhere without requiring any setup


###This is an extremely powerful concept
####Wouldn't it be nice if the same ugen code could run on your macintosh latptop, your linux desktop, and your fridge?

###The web is only going to get faster, and support for audio is only going to get better.


###But if developers living on the bleeding edge don't take the platform seriously, it will only take longer for the web to catch up with native code.



#The Begining
<br>
###Faust2Webaudio
###[Myles Borins](http://thealphanerd.io) | [@the\_alpha\_nerd](https://twitter.com/the_alpha_nerd)