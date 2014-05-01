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


