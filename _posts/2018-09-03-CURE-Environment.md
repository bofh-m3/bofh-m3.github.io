---
layout: single
title: "CURE - Environment"
excerpt: "CURE - The environment. I would like to, but this is about the technical environment of CURE, the homebrewed monitoring system."
date: 2018-09-03
comments: true
permalink: /CURE-Environment.html
tags:
  - cure
  - environment
  - design
  - powershell
  - monitoring
  - postgres
  - windows
  - linux
  - flask
  - nginx
category:
  - cure
---
Yes, curing the environment would really be on anyone's to-do. But this post is about the [CURE](/CURE-Design.html) technical environment. As described in the previous post, CURE (Company Unified Real-time Events) is a homebrewed IT monitoring system.
Now let's dive in to the environment setup we chose.

## Detector
The system is designed so that all persistent memory is in the database. Therefore, we can have one or more hosts for running the actual Detectors. The Detectors are designed to be stateless. We envision them to be able to run on basically anything, and somewhere in the future they may be running serverless, for example as Azure functions.
As explained, a Detector is a script that collects event data from the target system, analyzes it (green/yellow/red), and shoves it into the database for presentation in the UI.
I decided to (until other needs arises) run all detector scripts on a Windows 2016 server in our internal data center. The majority of the Detectors will be written in PowerShell, utilizing windows native cmdlets, so we scrapped the idea of going powershell/.net core on Linux for now. This host can easily be replaced and/or extend with one or more Linux hosts in the future.
And, like I mentioned, even run as functions in the cloud.
In future posts I'll describe the setup of the Detector host in more detail.

## Database
We decided to go with PostgreSQL on Linux. We could not find one single reason to do Microsoft SQL for this use case, but plenty of reasons for Postgres. In fact, it's part of my personal mission to rid the company of all production MS SQL databases. The dev MSQL we will never be able to remove, but I'm fine with that.
I will not go into detail on setting up Postgres in this post, but the process is quite straight forward and there are plenty of guides out there. 
In order to interact with Postgres, you'll need a driver. Driver for windows is available [here](https://www.postgresql.org/download/windows/). When installed, you're able to interact with postgres, much in the same way as mssql.
I will describe the design of the database in future post(s).

## GUI
My excellent colleague [Micke](https://www.linkedin.com/in/mikael-öberg-82520742) and I discussed several different approaches for the API/UI. We (mainly him) wanted it to be Python, and we both required it to be OSS and hosted on Linux. I pushed for Django, but, in hindsight, am very pleased I let Micke convince me to go with Flask.
Regarding the GUI, Micke insisted we'd keep it simple with a little JavaScript and some CSS. No Node.js, no PHP and absolutely no JS Framework. 
So far, I think it was the right decision.
We decided to run the GUI on the same host as the postgres database and publish it with nginx.
In future posts, I will get Micke to write more details on the GUI parts of CURE.

## The environment
Part of the environment, of course is the **Development** environment. Neither me nor Micke are *proper* developers, and the thought of source control, deploy pipeline and stuff like coding styles scares us. 
However, we decided on some coding style ground rules, and also decided to put source control and deploy pipeline on the to-do. 
Future will tell if we actually get our shit together and set up a proper CI/CD... 

![CURE Environment](/assets/images/cure-environment.png)

## The process
When we had decided on the environment, we (me and Micke) went off in a building frenzy. Micke set up the 2 VMs needed, I designed the database with good input from Micke and our Finnish colleague Janne and coded the foundation for the Detector host. Janne also helped with the over all design, being a super senior developer and all, he was certainly a trustworthy source for approving the designs we made. Micke went to work with the UI and API, and I must say I'm very pleased with the result!
 
*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*

