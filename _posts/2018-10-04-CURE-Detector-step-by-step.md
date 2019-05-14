---
layout: single
title: "CURE - Detector step-by-step"
excerpt: "CURE - The homebrewed monitoring solution. In this post I'll describe the steps for setting up an example detector."
date: 2018-10-04
comments: true
permalink: /CURE-Detector-step-by-step.html
tags:
  - cure
  - powershell
  - monitoring
category:
  - cure
---
Previous posts in this series:
- [CURE-Design](/CURE-Design.html)
- [CURE-Environment](/CURE-Environment.html)
- [CURE-Database design](/CURE-Database-design.html)
- [CURE-Detector foundation](/CURE-Detector-foundation.html)

## CI/CD? Source Control?
Ok, I'll admit straight up, we have none of that stuff.
We're not *developers*, we're *ops*! So no thank you mam, we'll just source control via VM snapshots and backup, and we'll just develop new detectors and bug-fix right on the production environment. 
Yolo.
Well, having proper source control and a streamlined deploy pipeline *is* on the bucket list. But you know, being ops guys and basically finding that stuff unnecessary, I wouldn't hold my breath...

#### New detector
So I want to create a new detector to monitor some sort of system. How do I do? 
(1) RDP to the detector host
(2) Open a PowerShell console
(3) Load the necessary [functions and settings](/CURE-Detector-foundation.html) into that PowerShell console

```powershell
$rootPath = "E:\CURE"

########################### LOAD MODULES AND SETTINGS #############################
(cat -path $rootPath\globalSettings.ini | out-string) | iex
Import-Module $rootPath\modules\Credential.psm1
Import-Module $rootPath\modules\CURE.psm1
Import-Module $rootPath\modules\ODBC.psm1
Import-Module $rootPath\modules\Detector.psm1
```

(4) Create a new detector by running something similar to the below command. The *New-Detector* will copy the *detectorTemplate* into the *detectors* folder and set up all the stuff necessary in the database.
 
```powershell
New-Detector -name "My Example Detector"
```

(5) Open the *MyExampleDetector.ps1* script that was created and modify it to reflect what is to be done in this particular detector. The first thing to do however is to change *$detectorName* to reflect the name of the detector, in our case it should read

```powershell
$detectorName = "My Example Detector "
```

(6) Write whatever needs to be written to make the detector do its job. This is custom for all detectors.
(7) When you're done with your detector, enter the following command, optionally also doing custom settings for *-refreshRate*,  *-detectorEnvironment*, *-heartbeatTimeOut*, *-area* and *-snoozeTime* 

```powershell
Set-Detector -name 'My Example Detector' -isActive True
```

Here's a screenshot of the commands that have been run in the console.

![New detector commands](/assets/images/new-detector-commands.png)

(8) But of course the detector right now is just a dead script and won't run unless **scheduled**. So let's do that.
- Open **Task Scheduler** and create a *Basic Task*
- Name it for example "Detector My Example Detector"
- On *Task Trigger* select *daily*
- On *Daily* select *Recur every 1 day*
- On *Action* select *Start a program*
- On *Start a Program*, enter *powershell -file "E:\CURE\detectors\MyExampleDetector.ps1"*. The wizard will give you a warning about arguments, just click *yes* to let it fix it for you
- On *Summary* select **Open the Properties dialog** to make more settings
- On the *general* tab, select **Run whether user is logged on or not**
- On the *Trigger* tab, edit *Daily* trigger and/or create multiple triggers to suite you need. In this example I'll trigger the script to *Repeat task every 5 minutes*.
(9) Finally we have a new detector up and running, getting source events, analyzing them and shoving them into the database ready for consumption by the API/UI!


*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*


