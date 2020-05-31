---
path: '20200531'
date: 2020-05-31T09:53:49.313Z
title: First build
description: 'Where I build a simple state machine for tracking form interactions. '
---
For sure I'm a vastly, vastly better programmer now than I was when I did it professionally. 

This seems paradoxical but it's the truth. This was my cycle before:

* Get a basic idea
* Write code (with no tests, this was the 90s after all)
* Watch it crash with frustration
* Debug it.
* Make simple changes that I'm convinced are perfect
* Watch it crash again
* etc

My first rudimentary but nontrivial Flutter app worked first time. Perhaps specifically because I work very hard to reduce stress in my life. Here's how things tend to go now:

* Get a basic idea
* Write out some notes for any complexity needed
* Think about it for a bit. Design it in my head. Ask "does this make sense?" "What about this scenario?" etc. 
* Write some code. 
* Assume it failed and review the code. Make the inevitable changes as I find bugs.
* Then hand simulate it. Walk through line by line. Fix more bugs. 
* Run it. 

It worked because I knew what every line of code did and when it'd be called. 

As I'm still figuring out how to do unit tests in Flutter, I don't have any tests at all, and that makes me uncomfortable. Also my code will be more testable if I can substitute out the rules. I'm presuming that's what BLoC is all about. Separate business logic and functionality and keep the UI clean from it. 

Onward!

![](/assets/screenshot_20200531-110515.png)
