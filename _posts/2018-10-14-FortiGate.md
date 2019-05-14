---
layout: single
title: "FortiGate"
excerpt: "Firewalls. Can't live with them, can maybe live without them in a near future. Maybe I'll just go and find my old bootable floppy with linux and iptables on it and say good bye to this NGFW crap..."
date: 2018-10-14
comments: true
permalink: /FortiGate.html
tags:
  - fortigate
  - firewall
  - infosec
  - soc
  - networking
  - cisco
  - datacenter
  - infrastructure
category:
  - rant
---
My first assignment with the company, almost 17 years ago, on my very first day of employment, was to go through the infrastructure of the newly bankrupt dotcom company, on whose remains the new company was to be built. I was to basically go down in the basement server room and put stickers on the infrastructure equipment that should be bought and extracted from the bankruptcy into the new company. 
How do you select what's important in an organization where you have no history or clue what the future holds? 
Well you ask IT. 
In a room a couple of floors up I found them, busy doing factory resets on dozens of access switches, preparing them to be sold off in bulk. They were reluctant to help, since they had just lost their jobs, but I turned on my puppy eyes and they agreed to browse through the list of servers and mark those of importance. 
One of the (many) pieces of equipment marked as important, was a firewall called Gnatbox. I had never heard of that before, I had only previous experience with the Cisco PIX. But yea, I agreed with the IT guys, the production firewall was to be considered one important part of the infrastructure and therefore should be marked with a sticker.

![Gnatbox ui](/assets/images/gnatbox-ui.jpg)


## FortiGate
The Gnatbox served me well for a couple of years, I was amazed by the straight forward-ness of the box and loved the fact the whole system ran from 3,5" floppy disk! But when it died I bought a Check Point and when that died I bought a WatchGuard, and when that died a bought a D-LINK. I was very satisfied with the D-LINK! The Chinese had really made a bang-for-the-buck box, and it was very straight-forward to work with, containing all the features needed. And since the company during this period was quite small, we had no need for advanced features or performance. 
But after the global financial crises of 08 had played out, the company started to grow rapidly. And I realized the company needed to step up its firewall game to meet the new demands. Cyber threats started to be real, and last, but not least, we had just signed up for a 1GBs internet connection (this was almost unheard of at the time, but Stockholm was really ahead in the broad band game, and 1GBs wasn't very much more expensive compared to 100MBs). 
Ok, so I needed a firewall solution with UTM capabilities, HA and high throughput. 
Where to look?
Cisco.
The IT crew in the Danish branch were die-hard Cisco fanboys. Well, also, [Microsoft](/Office365.html) fanboys. They were *not* progressive in their thinking, and if it wasn't a Cisco or MS sticker on it, it was crap in their eyes. They preached Cisco to me, they pleaded and threatened.

![cisco](/assets/images/cisco.jpg)

But I was tired of Cisco (and to be frank, also the Danish IT crew). I didn't like Cisco at all, their monopoly and omnipotence in the networking arena, their scare tactics and proprietary "standards" had left me repulsed. And once I had received an offer on the Cisco kit suiting my needs (UTM, HA and 1GBs interfaces) I almost fainted; the Cisco firewall offered would cost me more than my entire data center!
So, I started to look elsewhere.
Luckily, my new IT provider had just certified a couple of consultants on FortiGate and were eager to sell me their kit. I received an offer, and I kid you not, it was 1/10 of the cost compared to the Cisco box! Forti was trying to get a foot-hold in the Nordics at the time, so they had a drive where they basically threw in the second node in the HA cluster for free! Icing on the cake was the extreme discount on the UTM services bundle at the time. Also, my new IT provider was eager to get a Forti case to flaunt, so I imagine they transferred all margin on the hardware to me, just to secure me as a Forti customer.
Sold!

## The early years
It was dreamy! Having endured the horrors of the ASDM GUI, I was thrilled! 
The FortiGate GUI actually made sense to me! And once I got the hang of config being committed *instantly* (that was scary at first), I loved it! Also, the CLI, in my opinion, was groundbreakingly excellent. The navigation of the CLI mapped that of the GUI and it was really easy to get into.
And the features!
Boy did we get features! For example, VDOM was perfect to be able to host our daughter company and some sub-letters at the same hardware but keeping their networking separate. There was not a single thing missing, and suddenly I could create the HA network I had dreamt of in the last couple of years! 
And performance was just brilliant! ASICS humming away, offloading SSL and IPSec, crunching packets and doing antivirus without even catching a breath!
And to me, naming the documentation *Cookbook* felt very modern and refreshing at the time (of course nowadays I get vomit in my mouth when IT vendors refers to their documentation as cookbooks and recipes).

![forti cookbook](/assets/images/forti-cookbook.jpg)

I perceived Fortinet as being everything Cisco wasn't, modern, functional, forward-thinking, and I loved it!

## Lately
I'm on my third (or fourth?) generation of FortiGate, and they have served us mostly well so far. During the [CDC](/Consolidated-Data-Center.html) project I decided to stick with FortiGate and equip all branch offices and CoLos with HA clusters of varying models. It made sense to go with FortiManager and FortiAnalyzer in order to build one logical firewall covering the whole group and all its sites, centralizing management and maintenance. We considered pfSense for a while but found show-stoppers (can't recall which though). Forti had just launched their Azure marketplace appliance, and that was one of the reasons for us to go with them again, since we had just decided to let Azure be the company's primary IaaS/PaaS platform. Other compelling reasons were that we had already let other services integrate with Forti, such as Netskope, and therefore were somewhat stuck in their eco-system. Since we possessed a lot of knowledge in operating Fortigate, staff needing re-training if we were to shift provider, was not very attractive either.

![hands tied](/assets/images/hands-tied.jpg)

Jumping from 4.x to 5.x was a big step, and I must confess I have not yet gotten the hang of FortiManager. So far, I'm not very pleased with it, it's buggy and unintuitive.
There has been some really nasty bugs and breaking changes and we always need consultants to hold our hands when doing upgrades, reading the release notes *super carefully*. And for example the [back-door](https://www.theregister.co.uk/2016/01/12/fortinet_bakdoor/) thing a couple of years ago, where Forti lied and tried to cover it up, and the fact that *not a single* firmware release has been stable, but you need to wait *at least* 6 months to dare push the upgrade button, is not something to inspire confidence in the capabilities of their DEV and QA departments. 
The FortiClient, used for Dial-in SSL VPN, died on all our Macs when a new OSX was released a couple of years ago (we have 70% macs on the client side), and it took Forti 6 fucking months to get their act together and produce an update! Fortunately, we don't host many services that need VPN to be accessed, but for the few that do, we had an angry mob screaming at us daily since they were not able to work from home. We finally had to set up a sperate VPN service (PPTP) just to not get lynched while waiting for that Mac FortiClient update. 
Unacceptable.
And when confronted with all these bugs and scandals, the official Forti [defense](https://www.reddit.com/r/networking/comments/5fjity/tell_me_your_fortinetfortigate_horror_stories/) seem to always be "all other NGFW vendors are just as bad". I don't like this shift in attitude from them. 
They have become megalomaniac. 
They have become Cisco.
And another thing that really gets my panties in a twist is that you must create a developer account to get documentation on (and access to) their REST APIs! And it costs money! This is just not acceptable! I have already payed for this, just give me the fucking documentation! I don't care about being part of your "developer community", I don't need your crappy python SDK wrappers, I just want the god damned swagger to start automating my own stuff! Give it to me now!
I feel cheated. I had carefully put API access as a prerequisite when buying FortiGates for the whole god damned organization, and of course the sales material promised available, public, modern REST APIs. Not in my wildest dreams could I have thought that meant available, as in tucked away behind a pay-wall! 

![greedy](/assets/images/greedy.gif)

Speaking of cost, the performance-per-dollar ratio is no longer as compelling as before. Sure, you still get most bang for the buck with FortiGate for our use case, but I think their market strategy, trying to become the new Cisco shifting all kinds of crappy network equipment and services, has really seen the quality of their core product take a dive.
Sad.
Next time I equip a multisite company with firewalls, I will consider alternatives. But for now (not having done a lot of research I must confess) the competition seems "just as bad". The InfoSec business, and the firewall vendors in particular, seem to be covered in marketing snake oil and buzzword bingo at the moment, flaunting SDN and Fabric and whatnot but letting basic hygiene slip. Just make sure you have stability in the core product, it being a crucial (maybe *the* most crucial) part of the enterprise infrastructure! Win customers on stability and bang-for-the-buck! The number of gullible C-suites ready to swallow your marketing bullshit *will* run out sooner or later, and you'll be stuck with grumpy admins to try and shift your gear to. And let me assure you, these admins value stability, cost and being told the basic truth of the products they buy!
I long for the next Fortinet to come along and disrupt!
Maybe even Gnatbox will see a revival?
Or maybe, just maybe perimeter firewalls soon will be a thing of the past.
I start to think in the lines of "why the hell have UTM capabilities when everything is end-to-end encrypted nowadays, and therefore not available for DPI"?  SSL inspection you say? Fuck that, if I were to MITM our sessions, the developers would kill me! And I'd probably break 50% of the SaaS services out there! 

![gnatbox logo](/assets/images/gnatbox-logo.png)

Maybe I'll just go and find my old bootable floppy with linux and iptables on it and say good bye to this NGFW crap...


*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*


