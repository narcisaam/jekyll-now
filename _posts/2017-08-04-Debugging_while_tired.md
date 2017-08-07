---
layout: post
title: Hardware Setup
---

### Don't debug when you're tired!


Have you ever had a bug in your code that turned you from a rational, reasonable human being to an angry, paranoid person? I'm talking about those times when you're a 100% sure that your code is correct, so you start thinking there must be something very weird going on that causes it to crash. It gotta be something else! _Maybe the virtual machine is broken, maybe it's a compiler bug!_ When you're working with hardware it's even easier to find reasons. _This wire doesn't look really good, maybe I should replace it_. Well, to keep programmers from wondering and wandering aimlessly, debugging tools have been invented. But debugging tools come with a warning. “Not to be used when you're tired!” Haha! 

Yesterday, I was working on a new feature for my driver and I managed to somehow cause a kernel oops. I double checked the code - it seemed fine (of course it did!). I started investigating the problem, I analyzed the stack trace – not very helpful on its own. I disassembled the code to find where it crashed and after victoriously finding the offset, I thought: **no, it doesn't make any sense, there's nothing wrong with the code here.** It seems I got somehow overpowered by a very well known human response: denial! So, I wasted some time testing again some other parts of the code, rereading some documentation and searching the internet for similar bugs. At that point, it got pretty late, so I decided I'd take a break and start again in the morning.

Today, I arrived at the workplace, turned on the computer and started recollecting my unsuccessful attempts at fixing the bug. As I sat at my desk, thoughts were spinning in my head (this time, the right kind of thoughts) _What was that assembly code trying to tell me yesterday? Where was the code crashing?_ I launched vim and before the sound of my coffee pouring into the mug ceased, **I knew exactly what was wrong!** It took me several minutes to fix the problem that caused me to spent few hours on it just the day before. 

You imagine how angry I was at myself. Why didn't I check more carefully the instruction that was causing the crash? Well there's no point in blaming yourself, if you don't do it publicly, so there you go! Now you know how this post came about!
Well, every cloud has its silver lining, every mistake gives you a lesson to learn. Today I've learned not to get stuck on the same problem for too long and not to **debug when I'm tired**

#### P.S. I felt pretty miserable as I was failing to find the cause of the crashes the other day, so shout out to that guy who was having a hard-to-find bug as well and described his problem on a programmers' website, starting his post with “Am I going crazy?”. You didn't help me fix my problem, but you made my day better!
