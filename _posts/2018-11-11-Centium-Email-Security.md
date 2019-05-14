---
layout: single
title: "Centium Email Security"
excerpt: "Email gateways, from software, to appliance, to SaaS."
date: 2018-11-11
comments: true
permalink: /Centium-Email-Security.html
tags:
  - centium
  - defendas
  - csis
  - infosec
  - soc
  - antigen
  - ironport
  - carbon_black
  - microsoft
  - office365
category:
  - rant
---
During my work with the company's [Security Governance](/Security-Governance.html), Michael, the Danish branch IT-manager at the time, pointed me to a company named Defendas. 
Defandas, I learned, was a break-out product company from the very well-respected Danish cyber security company CSIS. I had not heard of these companies before, being Swedish and therefore priding myself on not knowing what our little sibling Denmark was up to, but once I started to dig in to them, I really liked what I saw. They had a no bullshit approach, the website was very low on marketing buzzwords and I could just imagine the cyber nerds, sitting in their space-age chairs, looking at a wall full of monitors displaying world maps with blinking lights, real-time syslogs flickering by on some monitors, listening to loud, crazy industrial music, wearing long trench coats and gas masks, sporting purple dreads, ready to jump in to action at the sign of some state-sponsored hacking group activity, a suspected BGP route hijack or a botnet whirring to life.

![hacker dungeon](/assets/images/hacker-dungeon.jpg)

## History
Email gateways. I have come across a couple during my career, and to be honest, I've liked them! In the early years (around the y2k) when spam and virus started to become a real nuisance, I was forced to start looking for a solution. The golden standard back then was a product named Antigen by Sybari. 

![Sybari Antigen](/assets/images/sybari_antigen.png)

Antigen was installed directly on the exchange server. It used several external antivirus engines to scan emails real-time. It had automatic updates and was really easy to configure and manage. Over all it worked nicely! And the flood of viruses and spam was halted at the server, instead of having some god damned plugin to the end-point antivirus client, crashing and fucking things up on a daily bases until the user got frustrated and turned it off, just to immediately get infected by a poisoned email. 
But there was a draw-back with Antigen; it was performance hungry. Back then servers were [physical](/VMWare.html), and performance was *always* a problem for exchange, so a fair amount of over-sizing the environment was needed in order to make things run (somewhat) smoothly.
But when Microsoft bought Sybari (around 2004) the product quickly turned to shit. It was swiftly rebranded, repacked into "ForeFront Security" and subsequently destroyed. But the biggest problem was not MS mishandling of Antigen, it was its hunger for resources. 
So I started to look for alternatives.
Enter IronPort.

![IronPort logo](/assets/images/ironport_logo.gif)

It was a hardware appliance box, specially crafted to handle large amounts of email passing through it, and besides for the features found in Antigen, it also had novel stuff such as reputation-based filtering. I set up a PoC with a local vendor, it worked beautifully! I was thrilled! Finally the Exchange servers got some breathing space, and things ran really smoothly, email wise. The IronPort was not cheap, but gosh damnit it was worth every penny!
Things ran perfectly for the Swedish branch for a couple of years, while the other branch offices struggled along with ForeFront. When Cisco bought IronPort in 2007, the Cisco fanboys in the Danish branch finally got interested. 
"We could set up a central appliance and funnel all our email through it" the Danes proclaimed.
"Sure, we can use mine, I've had it humming along for two years now..." I declared.
"No we need to buy it through my special buddy cisco vender here in Denmark".
"Whatever".
Said and done, the Danes bought a new IronPort (I guess basically the same box but with a Cisco sticker on it) and set it up. As expected, it worked nicely, and the added load from the other branch offices was no problem for it to handle. 
We ran IronPort for maybe 4 more years, and really, I had nothing to complain about. But when we decided to go all Microsoft Cloud back in 2011 (exchange online having built in spam and virus protection, probably ForeFront/Antigen), the good old IronPort was finally put to rest. 

## Centium Email Security
As mentioned above, a couple of years ago, I had not heard of the Danish cyber security entity CSIS or its product branch, Defendas. The local IT-provider in Denmark at that time had put together a "security package" with components like Kaspersky end-point protection, DNS Security (a black-holing service) and Centium Email Security. Centium caught our interest (sure there's basically the same functionality out-of-the-box in Office365, but we wanted something with better control and reporting. Also it doesn't hurt to have a multi-layer approach to email security, it being the number one source of bad things and all...), and I approved a PoC to be set up. We were content with the PoC and decided to buy the security package offered to us (but replacing Kaspersky with Carbon Black Defense).

![Centium dashboard](/assets/images/centium_dashboard.png)

## Conclusion
I'm over all content with the service. It's a SaaS, it's been up (almost) all the time, there's not much maintenance. Much like IronPort and Antigen before that. 
But there's one major problem.
There's no public API. 
This is becoming more and more of a problem for us!
We could easily put some monitoring on it using [CURE](https://bofh-m3.github.io/categories/#cure), to get early indications of customer mail being caught in quarantine or outbreaks happening. 
*If* there were only a way to get the event data out! 
This may very well turn out to be a show-stopper for us. Let's see what the progress is when it's time for the next license refresh.

*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*

