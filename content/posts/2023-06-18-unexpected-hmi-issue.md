+++
title = 'An unexpected human-machine interaction peculiarity in an iOS app'
date = '2023-06-18'
draft = false
+++

We've got a bug. The pen tool didn't work on M2 iPad with the Wrist Protection setting enabled.

Our Wrist Protection was added long ago when we were making Remarks – the note-taking app based on our PDF library. Pen was the main tool in that app. In our testing, we found that quite often the palm laying on the iPad screen would produce some touches. Our code didn't know if a finger or a wrist had produced the `UITouch`, so those accidental touches made pen marks on the page bottom. Wrist Protection filters our touches with `majorRadius` larger than a threshold that we heuristically found out.

So, back to the nowadays bug. Please take a look at [the video demonstrating it](https://drive.google.com/file/d/1XOQHgbfw8xmffSimVNnyXKT-glBNv4SD/view?usp=sharing).

<!--more-->

I tried to reproduce the bug with no luck. I called Nastia, who reported the bug. She reproduced it on her device with ease. We debugged it on her machine. Pretty soon we found the line where our wrist protection discarded touches. Turned out that `majorRadius` at her device was larger than our threshold.

But Nastia didn't put her palm on the screen. And I couldn't reproduce it on my device.

And then I've got it! Look at the video again! Look how Nastia puts her fingers on the screen! She's got long nails, she has to put her finger flatly. I don't have that elegant nails – I use my fingertip. And that is where that big `majorRadius` comes from "at her device".

So, if you use `majorRadius`, be aware :-)
