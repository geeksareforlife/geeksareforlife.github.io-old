---
layout: post
title: "Buzzer Project"
date: 2014-02-23
thumb: "/assets/images/2014/02/buzzer-thumbnail.jpg"
hero: "/assets/images/2014/02/buzzer-hero.jpg"
excerpt: "Short deadlines are the mother of all inventions! Here I build a quiz-syle buzzer for work."
categories: [laser, work]
---
I foolishly agreed to build a University Challenge style buzzer system to use for a work quiz.  And it needs to be done by April.

So, what do I need to do?

* 8 buttons
* Lockout - first button wins
* Visual display of which button has won
* Sound indication of buzzer
* Reset for new question

Luckily, an Arduino has enough pins to do all the buttons and the LEDS and a reset button without having to mess around with shift registers and the like!

First decisions to make are about how the system will look.  I want each person to have their own buzzer, and not have to crowd around a box, so we need 8 big buttons.  These need to connect to a central box that will house the display and the reset button.  Sounds simple!

![First Prototype]({{ "/assets/images/2014/02/buzzer-prototype.jpg" | absolute_url }})

So it's time to prototype a circuit and write some code.  This actually only took a few hours at an Open Hack night at <a href="http://www.nottinghack.org.uk/">Nottingham Hackspace</a>.

The code is pretty simple, and I'll include that and all other source files in a later post (EDIT 2018-03-02: see the bottom of this post!).

You can hopefully see that I've decide to use cat5 cable to connect the central box to the buzzers - it's readily available at the hackspace!  I'm using some <a href="https://www.sparkfun.com/products/716">Sparkfun RJ45 breakouts</a> I had lying around for the connectors.

I decided to copy <a href="http://www.re-innovation.co.uk/web12/index.php/en/information/random/designing-laser-cut-enclosures">Doc Little's enclosure design</a>, so spent the next week ordering parts - some M3 hardware, arcade buttons and the female spade connectors to wire them up.

![Buttons!]({{ "/assets/images/2014/02/buzzer-buttons.jpg" | absolute_url }})

I also spent the week designing enclosures.  I settled on a small box for each button, and two groupings of four.  The button boxes are wired back to a small distribution box with the RJ45 connector in it.  These then link back (via a network cable) to the central box.

On my next trip to the hackspace, I cut one of the boxes for the buttons.  Unfortunately I only had the time to cut one, but it looks pretty awesome!  Just 7 more to go!

![Button number 1]({{ "/assets/images/2014/02/buzzer-buzzer.jpg" | absolute_url }})

My next trip to the hackspace will be one of two things:  Either cut more boxes and build them (if the laser is available) or work out how to replace the electronic buzzer with an awesome bell for more realistic University Challenge sound!

Someone found this bell in the materials boxes at the hackspace for me.  Have I mentioned that hackspaces are awesome?

![Bell]({{ "/assets/images/2014/02/buzzer-bell.jpg" | absolute_url }})

## Update 2018-03-02

It's been a while and clearly I didn't write a second post!

Unfortunately I was unable to get the bell working in time, but the buzzer system itself worked great!

I have now added all the files to [GitHub](https://github.com/geeksareforlife/buzzer) (finally) so feel free to check them out and reuse whatever you like in your projects.