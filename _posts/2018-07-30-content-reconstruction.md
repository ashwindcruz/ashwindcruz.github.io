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
Reading through it, I learnt that for what I was trying to do, I should just be loading the weights into the model code and didn't need to go as far as trying to create a frozen graph. So that's what I did, supplying a simple numpy (np) (*or was it a placeholder*) array as a input. 
Lo and behold! I obtained a response from a specific layer. Thanks to the tidy setup of the model file, it was quite easy to specify which layer I wanted features from. 

Since it's the image that I needed to optimize, it would have to be a variable instead of a placeholder. I made the switch and ran into a an error when I tried to restore the weights of the pretrained vgg-19 network. I thought TF was smart enough to compare what was in the checkpoint file and what was in the current graph and restore variables that overlapped. Instead, it tried to restore all the variables in my current graph using the information in the checkpoint and broke when it encountered the new input variable I introduced which was not in the original checkpoint. Thankfully I found [this link] which taught me that I could specify which variables I would be loading when I instantiated the saver. It was a little counterintuitive as I had initially thought of the TF Saver object as primarily for _saving_ models and while it was odd that it could also load models, it was even more odd that you had to specify what the _saver_ would be _loading_ when you created the saver! While the *link* I mentioned earlier should clarify how you restore specific variables easily, I've also written a *short snippet* to explain how to do it. 

Then another very odd error. Long story short, sometime during my attempts to fix the Saver issue, I changed the input variable type from float32 to float64. I can't recall exaclty what led to this decision but I imagine it happened in one of those 'if I just keep fiddling with various knobs, perhaps I'll stumble across the required fix'. This took a _very_ *very* long time to debug since the error was completely uninformative. Ironically, now when I see an uninformative error, one of my first checks is the types of the variables I'm using. If any of you are interested, you can find the *long* story here. 

Now that I had one new variable and one pretrained network, initialization was a two stage process:
*insert code snippet here*. 
Note that if you reversed the order of those two lines, the code would still run but it wouldn't be correct. 
Consider the case where the lines are indeed reversed. I initially thought that the restore function would initialize the variables with the desired values and then the more generic initializer would simply initializer whatever remained. Instead, the general initializer would initialize all variables, including ones that were better left alone! If you would like to see how I came across this, have a look *here*.
Also I realise that it's possible to specify with an initializer which variables it should target. I didn't go for this approach because I knew that while I currently just had the newly introduced input variable, before I was done, there would be plenty more new variables.  
Run the general initializer first and then restore variables which you have pretrained weights for. 

So I had an input, a network, and an output. Now I needed a loss to optimize. It would have to be between the real image's response and the response of the 'image' being optimized. 

One tip I have here when you are dealing with pushing images through networks is to start out with some dummy data. For example, just create a random numpy array in the shape of the expected image and treat that as input to your network. 
Pros:
* Easier to get the basic input you want. In this instance for example, you don't have to worry about downloading an image, ensuring you provide the correct path to it, resizing the image.
* For other use cases, it's easier to play around with the batch size. If you wanted to do that with real data, you might have to fiddle a bit with how input is read from your disk to ensure you're getting the right batch size.
Cons: 
* Format of the array will be different from a real images. In some ways obvious (distribution of pixels) and in other subtle (numpy arrays default to float, images to uint8). I'll elaborate more on this a little later. 

I used TF's assign operation to set the input variables to different values. First it was set to the 'real image' so I could collect the response to aim for. Then I set it to some white noise as this was the variable I wanted to train. Wrapped up the loss in an optimizer and tried to run it... success! The loss decreased. I changed the 'real image' from a fake numpy array to real data and the loss still seemed to go down which was a good sign. Obviously this time the loss started a lot higher since the our white noise input variable was a lot further from the real image. The white noise input variable I was optimizing came about from a numpy random array which by default gives us values between 0 and 1. I found that I could give this variable a 'helping hand' by multiplying it by 255 thereby making it a little closer to the real image at hand. This sped up training a little. 

It's almost never sufficient to look at the loss to determine if your model is working correctly. The great thing about working with images is that you can always visually inspect the progress along the way and it should make some sense. I viewed the optimized image at a few points during the training. It made very little sense. Something was off but I wasn't quite sure what. 