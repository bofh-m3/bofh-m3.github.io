---
layout: single
title: "Consolidated Data Center"
excerpt: "I faced a fight-or-flight situation. My cozy everyday IT bubble was under attack. Should I move away or should I stay and fight?"
date: 2018-09-10
comments: true
permalink: /Consolidated-Data-Center.html
tags:
  - datacenter
  - azure
  - vision
  - project
  - career
  - leadership
  - infosec
  - me
  - cto
  - infrastructure
  - microsoft
category:
  - leadership
---
Having worked for the same company for a really long time (14 years at that time), and made my everyday life quite comfortable, I suddenly found myself facing a fight-or-flight situation a couple of years ago. 
A little background.
The company I work for (a fairly large digital agency) is Nordic. We have offices in Finland, Sweden, Norway, Denmark and off-shoring in the Ukraine. The customers are Nordic. The company is owned by a private investor and each country is its own, autonomous company. The countries are controlled by a holding company (previously only on paper, but that was about to change). 
The markets are local, and the countries have (with a few exceptions) never cooperated or shared customer assignments (even though that always has been, and still is, the vision of the owner). I was employed by the Swedish company and, like all other Swedish employees, had never cared too much about the other countries or the top management in holding for that matter.

![Company Structure](/assets/images/company_structure.png)

Even though IT always had a somewhat recurring relationship cross-border, we were autonomous entities, one IT department per country. The recurring interactions we had was mainly out of necessity since we shared a lot of the infrastructure services, such as AD, Exchange, ADFS etc. Every couple of years there was an initiative to increase our collaboration, but there was never anything but half-assed attempts at building something together. There were never any leadership for IT.
Until a couple of years ago.

## Top management
It all started when our Swedish CEO for 10 years announced his retirement. I kind of knew things was about to change, especially since his predecessor was announced (the former sales manager), but I decided to "wait-and-see". Unsurprisingly, within months, everyone in the management team had left, probably disgruntled with not having received their E after their C. I didn't care too much, I made face time with the new CEO and got his trust in order to retain stability for my precious IT department and thereby maintaining my cozy little bubble. But shortly after that, new top management in holding was announced. They went in hot with high voices and declared a lot was about to be changed. I had not cared too much about the previous iterations of top management, and they had not cared too much about IT, but I soon realized these new guys meant business. And their business was going to become by business, since IT all of a sudden was on their agenda!
I spent two months (I kid you not) contemplating my future with (or without) the company. I was very straight forward with my colleague (we were only 2 on IT in Sweden at the time) and told him I was either going to make a career move or leave the company.

## The move
Having had a rough first couple of interactions with (my soon to be future boss) the new Holding COO, I realized that this is what I had been missing; A strong Leader, full of ideas, pragmatic, outspoken, and possessing an understanding of IT. And most importantly, possessing the power and stamina to see things through! Ok, she is somewhat of bulldozer, and that was very intimidating at first, but after a couple of more meetings I had made up my mind. I was prepared to stay and fight!
All the ideas for IT I've had through the years, suddenly seemed possible to realize. I had gained the trust of the top brass, and I had made myself a rough roadmap to see these ideas spring to life. The new top management was the key I didn't know I had missed. So I pitched the first and most obvious thing: *Let's consolidate our data centers.*

## The vision
The consolidation of our datacenters was just part of my vision. A consolidation would force closer collaboration and I was fully aware that we would need someone in charge for all the IT departments. I wanted that someone to be me. I was confident I would make it work, and unlike all previous attempts, I *knew* what needed to be done in order to succeed. 
Let me be clear, my heart lies with the company. I **always** do what's best for it, even if it means personal sacrifices. And I knew there were lots of things we could improve within IT if we just put our resources together and worked in the same direction. So the decision to aim for becoming the group IT manager was not just about ego, I actually believed there was no one better suited to do it than me. 
And now I had it within my reach. 
But there were steps that needed to be taken.
I will not go into details on these steps, but they involved travelling to all offices, getting face-time and buy-in from all local management, getting all IT staff on-board (that was the hardest part), doing endless presentations and calculations, making myself available for all kinds of inter-company projects and initiatives. Making promises and keeping tabs. In hindsight I realize most top-level managers, unless they truly understand IT (they never do), don't care too much about the details of a plan. They care about the ability to deliver. 
I have taken that to heart.

## The design
I dubbed the project CDC and defined it through a series of meetings, presentations and diagrams. Budget and project plan were produced and approved. I attended the monthly top-management meetings to report on the progress. 
When the project started, we had 5 separate data centers. Since we have public sector customers in Sweden and Norway, where they demand the data to be kept within the nations' borders, we decided to make one logical data center with co-location facilities in Oslo and Stockholm. We also decided to implement fancy stuff such as NSX and replication on the storage layer to be able to do hot fail over between the locations. 
Also, early in the project, I was swept up in an inter-company group (Tech Leads), where the CTO for each country was assigned the task to define areas of cooperation and to decide on some common grounds. One of the decisions was to make Azure our go-to cloud platform. This made sense since we are a heavy Microsoft house, and already had an Azure + Office 365 tenant up and running.
I agreed to include Azure Governance in the CDC project.
Also, the group COO (my boss), wanted our InfoSec to improve *drastically*, mainly because of the looming threat of GDPR. I agreed to include Security Governance in the scope as well.
And, naturally I was invited to join the group GDPR initiative. It made sense and I accepted fully aware of how much work it would add (and how much focus it would steal from the *real* project).
I was also involved in some other projects and groups, such as implementing a new ERP for the whole company, but I'll write about that separately.

#### Scope
Scope creep. I knew it would come. I even gave it room in the budget and road map. I was open about it and even used it as a bargain chip. In a project-based organization everyone can relate and see it as a sign of great sacrifice if someone accepts to expand the holy Scope to suite someone else's needs. 
Never the less, the Scope was maintained within reason and plan. 

![CDC Scope](/assets/images/cdc-scope.jpg)

#### Components
The three main components of CDC were
1.	Infrastructure
2.	(Information) Security
3.	Azure
3 bullet points. 
6 months to complete. 
Super easy n'est-ce pas?
I plan to write more in detail about these components, but each of them of course deserves their own post... As explained above, I let the scope expand since it was in line with my overall vision for IT. But all the extra stuff (Azure, InfoSec, ERP, GDPR etc.) really cut in to my time and focus for the project. Having a second kid in the middle of this surly kept me busy 24/7...

#### The Data Center
Well, the actual Data Center. What was that all about? Below is an image to describe what the project was (at least initially) about. Hardcore, old-school IT stuff that I feel comfortable doing. Rackspace, Storage, SAN, Network, Compute and hypervisors. Ahh, this part felt easy.

![CDC Design](/assets/images/cdc-design.jpg)

## The outcome
Well it turned out pretty much as I envisioned. On the technical side I would have done some stuff differently if I had the chance to do it again. But on the human side I'm very pleased. I delivered a consolidated data center on budget and (almost) on time. I got all of IT on-board and behind me. I got my new role as group IT manager. I now am employed by Holding and no longer exposed to the whims of local management. I see measurable effects; Cost going down, user satisfaction and efficiency going up. And most importantly, IT is now truly a cross border team, working together, towards the same goal, sharing the same resources. And let me tell you, in all my years in the company, no other team has been able to do anything close to what my team has done! We have made good on the vision of our owner, at least for one small part of the company. And believe me, there has been many attempts before me, in many different departments, groups and constellations...
Yeay me!
Go team! 
Developers! Developers! Developers!

## P.S. 
My fears about the new top-management was well founded. In the last couple of years there has been a reorg frenzy, several management teams and CEOs has been replaced, and most senior staff has left. At the time of writing I can't tell if it's been a bad thing or not. Some indications for it actually being a *good* thing exist (the company may need to rejuvenate and adopt to market). Time will tell. 
But at least IT is stable, and no one in *my* team has left. Voluntarily at least...

*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*

