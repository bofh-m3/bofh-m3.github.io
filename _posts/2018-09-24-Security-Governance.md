---
layout: single
title: "Security Governance"
excerpt: "Security policy and awareness. A subject so boring, it would set a rave going 19-year-old pumped full of amphetamines right to sleep within a couple of seconds. "
date: 2018-09-24
comments: true
permalink: /Security-Governance.html
tags:
  - infosec
  - policy
  - gdpr
  - soc
category:
  - infosec
---
The company had put [GDPR](/GDPR.html) on the top-level agenda. Since I was in the early stages of my personally defining project, [CDC](/Consolidated-Data-Center.html), Security (InfoSec) was shoe-horned in as part of that project by my boss.
Really, InfoSec had very little to do with the Data Center. I knew that. I guess my boss knew that. But still, it had to be done because of GDPR, so it appeared to her as a good a place as any to shove it in. Security into my CDC project that is, mind you.
So how the hell do you go about raising security awareness in a fairly large Digital Agency?

## The approach
I had already made up my mind about how *I* regarded security:
90% Behavior
10% IT tools
And to me, it's all about weighing risk and cost. Sure, we could lock down the place like fort knox, but that would hardly be a security level worth the cost, since we'd be out of business within hours.
Also, except for defensive (and offensive if you get that far) InfoSec tools and strategies, I consider stuff such as back-up and disaster recovery part of security. And physical security in the offices. And personnel security.
But what the hell, it's a digital agency, not the pentagon!
Let's focus on the biggest threats first!
And to my boss they were these:
1. Compliance (GDPR)
2. Theft
3. Disruption (malware etc.)

I myself would have placed them 1. Disruption, 5. Compliance and 999. Theft. But hey, maybe I'm just naive.
But I was obviously not able to be objective about the company and its risks. I have after all been here closing in on two decades now... 
So, I wanted someone external to give it to me straight:
How bad was the situation?
And where the hell do I look to find an InfoSec specialist nowadays?
Thanks to my longtime colleague [Fredrik](https://se.linkedin.com/in/kfpersson), relationships had already been established with an InfoSec company in Stockholm, and some products had been PoC'ed. 
So, one of the first steps was to buy an audit.
I cannot even begin to describe how underwhelmed I was by that audit!
Paying through the nose to have someone interview you while taking notes, only to give you a report reciting the exact words you have told them! To add insult to injury, I had to nag them daily for their crap report, being very explicit on how time was of the essence. And finally, 4 weeks after the set date, I received the not-even-good-for-toilet-paper report.
I encouraged them to go fuck themselves. 
Well it was not only because of their crappy so-called audit report, but the CEO of that company also tried to poach members of my team. When the first one declined, he actually went and asked the next one! In my book, you don't fucking try to steal your customers' employees! I've met a lot of slimy and incompetent providers over the years, but this one takes the cake. 

![Cartoon villain](/assets/images/cartoon-villain.jpg)

Anyway. 
We were back to square one on the audit front.
Again, Fredrik pulled someone from his sleeve. 
Turned out a close friend of his was into InfoSec. He had recently quit his job doing cyber security for OMX Nasdaq and was now trying to build his own company.
Perfect!
I considered the implications of hiring a friend to someone on my team to do security for us, but I concluded it should not be a problem.
Also, I had no other options, and the clock was ticking.
So, I asked Fredrik to set us up for a meeting.

## The preparation
Before we met up with our soon-to-be new InfoSec partner, the team and I had produced a document outlining the different attack vectors I could think of, and my suggested tooling for mitigation.

![Attacks and vectors](/assets/images/attacks-and-vectors.png)

I will not go into detail on all the areas and tooling suggestions I had put on paper, but I will share the headlines in the document for some nice buzzwords; MDM, Self-Service security and compliance, Application security, Vulnerability scanning, Anti-malware (FirstGen/NextGen/Hypervisor), Firewall UTM/NGFW, PenTests, Monitoring (Health, anomalies, forensics, trends), PII Detection, DLP, CASB, White hat disclosures (BugBounty), Wired/WiFi security, Authentication/Authorization, Perimeter security, Physical security, Data encryption, Hardware encryption, Threat Assessments, Backup and D/R, Outer perimeter protection, whaling, phishing, IT-personnel protection and auditing, targeted attacks, social engineering, whistle blowing.
The meeting went off well!
I felt I received what I wanted from the new InfoSec partner, and I scheduled them in for recurring sessions the upcoming 6 months, right there on the spot.

## The document
But, the IT part of security and compliance only being 10% of the whole, how do I address the remaining 90%? 
The behavior?
The people?
The *real* problem area?
Any IT system can be secured, but any human has the power to make it insecure again.
So, I did what every company does.
Policy.
And I did it knowing full and well that not a single living soul would read them and let alone comply to them. At least not willingly and without threat and/or encouragement.
But I decided the policies needed to be there in order to have something to point to when starting the carrot-and-stick-campaigns to implement them in the organization.
Said and done, I set about writing the policies.
I decided to do it with a twist; I decided to add policies *specifically* pointed at the IT department, to ensure we'd upheld out part of the solution over time!
I also decided, with encouragement from the delivery part of the organization, to make it "sellable", basically creating it in a way to also describe our environment and routines! That way, we could easily add the Security Governance document as an attachment in client negotiations and put parts of it up on the website to display to the world all the amazing security and compliance stuff we'd done!
I thought it was nifty!
But it was a difficult task, combining ~~bragging~~ informative material and actual, enforceable internal policies.
As you may understand dear reader (all two of you, hi mum) I cannot put the entire document here.
I would love to though, because I would really like to provide some other poor soul in my situation with a helping hand.
Drafting a Security Governance document is not an easy task, and I had to put in several days of effort and pouring my heart and soul into it. 
The hardest part was trying to convince myself this document would make a difference...
But I will however provide you with the headlines of the document:

*COMPANY SECURITY, COMPLIANCE AND BUSINESS CONTINUITY POLICIES*
_Contents_
1. CHANGES AND REVISIONS
2. ABOUT THIS DOCUMENT
3. TARGET AUDIENCE
4. SECURITY GOVERNANCE AT THE COMPANY
5. COMPANY SECURITY DESIGN
6. COMPANY SECURITY ORGANIZATION
7. COMPLIANCE WITH COMPANY POLICY
8. DATA CLASSIFICATION POLICY
9. ACCEPTABLE USE POLICY
10. DATA PROTECTION POLICY
11. PERSONAL DEVICES POLICY
12. NETWORK AND DATA CENTER POLICY
13. IDENTITY AND ACCESS POLICY
14. SOFTWARE AND SYSTEMS POLICY
15. AUDIT AND ACTION POLICY
16. TRAINING AND AWARENESS POLICY
17. GLOSSARY

Snore.
When the first draft was done, I handed it over to Fredrik (I had recently given him the role of "Head of SOC") and the InfoSec partner, so that they could lay the final touches on it and bake it into a release version.

## The implementation
A beautiful spring day back in early May, just in time for GDPR to go into full effect, the time had finally come to unleash the Security Governance policies upon our unsuspecting co-workers.
The boss, with my blessing, took the honors. 

![Unleash dogs](/assets/images/unleash-dogs.jpg)

She had already primed the top-management group and engaged a *communicator* to make the text in the communique as warm, fuzzy and exciting as possible.
The copy turned out good. Well as good as it gets, when talking about a subject so boring to most people, it would set a rave going 19-year-old pumped full of amphetamines right to sleep within a couple of seconds, if it was to be read to him. 
And I think a couple of our employees actually read it once it was posted up on the intranet!
That was a couple more than I would have thought!
So now, how do we reach the remaining 98%?
We decided on two parallel tracks:
- A road-show to all offices, armed with some nice slides, to go on stage and educate all employees on the recurring all-hands meetings.
- Make Security Governance part of the On-Boarding material for all new employees.
Awareness.
That's our mantra now.
It's the *single* most important thing when raising the *actual* security level.

## The outcome
Well, it's too soon to tell. We've yet to go on stage, and the On-Boarding material is still somewhat rough around the edges. But we'll get there.
Over all I think the work done so far has been good. 
But the job is not done.
Compliance never sleeps.


*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*

