---
layout: post
title: "Content Reconstruction"
date: 2018-07-30
---

If you're reading this, I'm assuming that you've read the paper X and have some familiarity with it. 

The focus of this post is on the _Content Representation/Reconstruction_ section of the paper, Section 2.1 to be precise. The authors use the title _Content Representation_ yet I've also included the word _Reconstruction_ here since that's what they did in Figure 1 and what I've attempted to do on my own as well. 

Throughout working on this (sub) project, I ran into lots of issues. There's so much that happens between the description in the paper and a fully-fledged implementation that I've never had to consider before. While I don't consider what I've done to be exhausitive, even just considering Content Representation, it's quite thorough and I've decided to talk about the most significant _gotchas_ I ran into. As some of these might be obvious to some of you readers, I've shifted the _gotchas_ into a separate post and through this post I'll explicitly link to those mini write-ups. I hope that they help you avoid sinking as much time as I did into these issues. 

So at the very start of the project, I wrote up the following TODO list: 
- [ ] Obtain a pre-trained vgg-19 network. 
- [ ] Push an image through an extract the outputs from the layers: 
* Conv1_2
* Conv2_2
* Conv3_2
* Conv4_2
* Conv5_2
- [ ] Push white noise through and extract outputs from the same layer. 
- [ ] Calculate the loss between the output from the image and the output from the white noise. 
- [ ] Backprop this loss back but do not update the weights. Instead update the white noise itself. 

This was essentially my translation of Section 2.1 into actionable items (the layers are the ones chosen in Figure 1 of the paper). It appeared relatively straight-forward to me and I didn't budget too long for it, perhaps a few hours. 
I was far-off the mark though to my credit, the TODO list in and of itself wasn't incorrect. There were just a whole lot of _gotchas_. 

At first, I was tempted to use the object detector networks located *here*. This might strike you as odd and it is. The only reason I considered this option was that I had previously worked with it and knew how to load the network with pre-trained weights and extract features from specific layers. I decided against it primarily because it's a more complicated network and the computational and memory overhead would likely be larger. 
So I took the more conventional route and looked at the *Tensorflow(TF)-Slim* repository (repo). While they have a very detailed README that describes how to use the repo, I was impatient and skipped most of it. From my experience with object detection networks, I knew that all I needed was the:
* Meta file which described the network's graph
* Checkpoint file which contained the pretrained network's weights

I found the latter in a *zip file*. I then learnt that I would not be able to find the former because these networks were trained and saved differently, using an older format. I was previously used to working with frozen graphs which is essentially a combination of the two files I was looking for. I looked around for how to obtain this for the vgg-19 files and came across this [link](https://github.com/tensorflow/tensorflow/issues/7172). 
Reading through it, I learnt that for what I was trying to do, I should just be loading the weights into the model code and didn't need to go as far as trying to create a frozen graph. So that's what I did, supplying a simple numpy (np) array as a input. 
Lo and behold! I obtained a response from a specific layer. Thanks to the tidy setup of the model file, it was quite easy to specify which layer I wanted features from.   

