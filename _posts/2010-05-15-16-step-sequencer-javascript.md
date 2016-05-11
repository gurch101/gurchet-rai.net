---
layout: post
title: A 16-Step Sequencer in Javascript 
date: 2010-05-15
tags: javascript app
---
You can play with the sequencer [here](http://www.gurchet-rai.net/apps/sequencer).

I was bored today so I built a 16-step sequencer in javascript which uses sounds from 5 different instruments that I ripped from [Reason](http://www.propellerheads.se/products/reason/). I was planning on using the HTML5 tag for audio playback but codec support is inconsistent across browsers – Chrome, Firefox, and Opera support Ogg audio, Safari supports AAC through quicktime and IE doesn’t have audio tag support at all. Instead of encoding audio in a bunch of different formats or using flash, I decided to use SoundManager2 which wraps the HTML5 audio api and uses flash as a failsafe. Since this is just a toy project, I didn’t mix the audio as you’d get in a real sequencer so you’ll occasionally notice clipping and samples playing off-beat. I wouldn’t recommend HTML5 audio or the above library for time-sensitive playback since you can’t get very low latency.
