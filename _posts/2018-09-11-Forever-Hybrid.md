---
layout: single
title: "Forever Hybrid"
excerpt: " I try to envision a future where the company I work for is Cloud Native."
date: 2018-09-11
comments: true
permalink: /Forever-hybrid.html
tags:
  - cloud
  - hybrid
  - compliance
  - infrastructure
  - legacy
  - datacenter
  - azure
  - microsoft
category:
  - cloud
---
I try to envision a future where the company I work for is Cloud Native. 
I don't mean born in the cloud, that ship sailed long before the name "Cloud" was coined. I mean actually taking the step to becoming full cloud. No local compute and storage, no infrastructure services, no CoLo, no self-managed [Data Center](/Consolidated-Data-Center.html).
I myself would love a future where my team could focus all resources on providing value to the company in the form of code, processes and services and not having to worry about hardware, vendors, models, maintenance, backup, Disaster Recovery, service deals, legacy tech and specialist consultants. 
I envision leading an IT department that is truly able to deliver without external help (except for the actual cloud providers of course), and not having to worry about keeping up to date with the latest firmware releases and vulnerabilities, maintaining deep knowledge on specific compute, storage and network equipment. 
But especially, I truly believe that the company would be able to find new business opportunities if we were to go full cloud.
We have most of the components in place to take the step.
![Cloud native meme](/assets/images/cloud-native-meme.jpg)
What's stopping us?

## Attitude
This should not be a problem. Most of our tools are already SaaS based, we already have development projects running fully in Azure, we've got Azure AD and office 365, and I have yet to meet a developer that *likes* the thought of going hat in hand to IT to request some VM or compute resources. IaaS/PaaS, if done right, would enable the developers to take charge of a bigger slice of the stack, and be less dependent on grumpy IT departments.
For the rest of the company (the non-developers), I can't imagine anyone disliking the idea of going full cloud. 
I can however imagine most of the non-developers not giving a crap.
Last, but not least, everyone on my team (IT) is pro cloud. That of course is a prerequisite if trying to go full cloud. 

## Needs
Like I mentioned, most of the services in use, and *all* of our most business-critical ones, are SaaS. The ones that are not SaaS are subject for demotion anyways. The (very few) that are left could probably be migrated to the cloud fairly easy.
I can't imagine a future where a service needs to be hosted OnPrem. And as long as I'm with the company, I'd fight implementing such a system nails and teeth.

## Legacy
There's 16 years of legacy. 
In our tech stack, yes, but more importantly in our way of working. A lot of the processes and tools the developers use today would need to be changed, migrated and/or redeployed.
Someone would need to go through 300+ VMs to decide on their individual faith.
This is a major hurdle, and something I, as an IT manager, cannot decide and execute on myself.
Being a company where the product is project hours, no one is really inclined to risk putting the production into a grinding halt...

## Infrastructure
Let's face it, a company is never truly rid of IT infrastructure. 
Infrastructure services such as AD/DHCP/DNS/ADFS can be moved to IaaS/PaaS and/or local network equipment.
The data center can be demoted when there are no longer any services running on the VMs, but we'll still need an ISP to provide services, and we'd still need firewalls, switches and WiFi for all locations. And I would like something to monitor these infrastructure components. Where do I place it?
Well you build some sort of extended LAN with the SaaS/IaaS, to run some resources that can access the internal stuff and do the monitoring and management of the equipment. 
Fair enough. 
But would that let us become *truly* cloud native, having Infrastructure services and hardware to maintain? Would that rid us of the need for expensive and difficult to get hold of specialist consultants?
A whole lot more then now, but never *fully*.

## Security and compliance
Being a Nordic (and therefore members of the EU) company with customers in public sector, we certainly need to care about compliance and information security. We have already put mechanisms in place through Security Governance and since we already have a heavy presence in the cloud, our tooling, policies and processes are already in place.
And since all major cloud providers (GCP, AWS, Azure) are building regions in the Nordics, we don't even have to care too much about data locality in the future. Data locality, the need to keep data within the nations' borders sometimes is a requirement in public sector.
I see no hindrance for us in going full cloud, at least from a security and compliance point of view. 

## Buy-in
As mentioned in *legacy* above, the biggest hurdle lies in sticking your neck out and changing a lot of legacy tooling and processes on the developer side.
Buy-in from all local- and group-level management is needed for this. 
It would be a change project, and we would need to calculate for loss of income.
To be able to pull this through without someone pulling the plug or reversing it at the first sign of trouble, total buy-in would be necessary.
I don't have total buy-in at the moment, and I would need to work up more energy to even try going out and getting it. 
It will probably take me some time to get there...

## Cost/Risk/Reward
Cost would most likely be one argument for (or against) going full cloud according to top-level management. The other arguments would be:
- New business opportunities 
- Efficiency
- Quality of the delivery
Regarding, cost, I don't see any savings to be made in our use case. We already have a good life cycle management and utilization of our Data Center. 
Regarding **risk** (that thing any good management group always *should* consider), except for what's been noted under *security and compliance* and *buy-in*, the only major risk I see is **vendor lock-in**. If we were to put all our eggs in the cozy Microsoft cloud basket, how the hell do we get out of there, once the basket no longer I cozy, but scary, expensive and restrictive?
Would we consider a multi-cloud approach? Possibly, but that would add to the cost and timeline for sure.

## Conclusion
I have decided to NOT do another major data center hardware refresh. I'm planning on letting the current batch of iron running in the CoLo end its life there.
Sayonara expensive boxes!
I will not miss your excessive need for care and attention!
Neither the sleepless nights you've cost us, patching, configuring and worrying you'll freeze up and die on us.
So that gives me apx 2 years to get the stuff I need (mainly buy-in and someone to help me pull the  train), to go "full cloud".
Better roll up the sleeves.


*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*

