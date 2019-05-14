---
layout: single
title: "CURE - Detector foundation"
excerpt: "CURE - The homebrewed monitoring solution. In this post I'll describe the foundation of the detectors written in PowerShell."
date: 2018-10-03
comments: true
permalink: /CURE-Detector-foundation.html
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

## Basic functions
I had previously built a monitoring solution, entirely in PowerShell, that outputted static html for consumption on a wall mounted monitor.
So, I had a pretty good idea on how to make the detector side of CURE as flexible and easy to maintain as possible, without wrapping to much of the functionality in hidden functions tucked away in modules. 
I knew there were some functions that would be used by *all* detectors running on the host, so I decided to start by writing those. I also needed a bunch of helper functions to run on the command line for administration of detectors. 

#### Credentials
First things first, since I'd have detector scripts running unattended on schedule I needed a secure and easy way to store credentials locally. I had previously written a handful of functions to do just that. And luckily, I had written them in such a way that they could be used on the console as well as in scripts, so I could just basically shove them all in a psm1 file and load them on startup.
The Credential functions can be found [here](https://github.com/bofh-m3/CURE/tree/master/PowerShellCredentialMgmt).
Mainly *Receive-Credential* is used by the Detectors, but also *Convert-CredentialToBasic* for APIs using basic auth, while *Save-Credential* can be used on the console for one-time saving of credentials to the Detector local disk, encrypted using the logged-in users keys.
I actually use these functions on my workstation as my personal password manager, so they're all loaded at startup on my machine.
On the detector host I put them all in *Credential.psm1* to be loaded at startup.

#### ODBC
Yes, each and every detector would need to interact with the postgres database. I decided to string together the commands needed to get or set data in postgres through PowerShell, into re-usable functions. The functions can be found [here](https://github.com/bofh-m3/CURE/tree/master/PowerShellODBC). Also these .ps1 files was shoved into the *ODBC.psm1* file and loaded at startup on the detector host.

## Detector functions
Since I wanted to administrate (create/edit/delete) detectors in a streamlined way I decided to write some functions to be used on the console for just that. The functions can be found [here](https://github.com/bofh-m3/CURE/tree/master/PowerShellDetector).
On the detector host I shoved them into *Detector.psm1*.

## CURE functions
I also saw a need for some functions specifically used by the detectors, basically stuff that each and every detector was gonna use. The script functions can be found [here](https://github.com/bofh-m3/CURE/tree/master/PowerShellCURE), and also they were collected into a psm1 file, *CURE.psm1*.

## Detector template script
Even though each detector is supposed to monitor very different systems and event sources, over basically any interface, I wanted to find a unified way of writing the detectors, to more easily navigate the detector scripts when they needed updating and/or bug-fixing. I gave it some thought and drew from experience and came up with this basic template file to be used when setting up a new detector.
#### DetectorTemplate.ps1
```powershell
$rootPath = "E:\CURE"

########################### LOAD MODULES AND SETTINGS #############################
(cat -path $rootPath\globalSettings.ini | out-string) | iex
Import-Module $rootPath\modules\Credential.psm1
Import-Module $rootPath\modules\CURE.psm1
Import-Module $rootPath\modules\ODBC.psm1
# Import-Module $rootPath\modules\Detector.psm1

################### LOCAL SETTINGS, FUNCTIONS AND DEPENDENCIES #####################
$detectorName = "My Detector"

############### GET DETECTOR SETTINGS AND VALIDATE CONNECTIVITY ####################
Try {$settings = Get-DetectorSettings $detectorName -EA stop}
Catch {Write-Log -scriptName $detectorName -errorMessage $_ -exit}
If (!($settings)) 
{
  Write-Log -scriptName $detectorName -errorMessage "$detectorName is not configured in database" -exit
}

####################### TEST IF SNOOZED OR NOT ACTIVE ##############################
If ($settings.isactive -ne 1) 
{
  Write-Warning "$detectorName is not activated. Run 'Set-Detector -name $detectorName -isActive $true' to activate the Detector"
  Sleep -s 10
  Exit
}
If (Test-IfSnoozed $settings.detectorid)
{
  Write-Host "$detectorName is snoozed"
  Sleep -s 10
  Exit
}

######################### CREATE DEFAULT LOCAL EVENT ###############################
$localEvent = Get-DefaultLocalEvent -DetectorName $detectorName -DetectorId $settings.detectorid

############################## CONNECT AND COLLECT ################################
<# this is custom #>
# try {Connect-TargetServer -server $myservername -EA stop -WA silentlycontinue}
# catch {
#   $localEvent.descriptionDetails = $_
#   $localEvent.status = 'gray'
#   $localEvent.eventShort = "Unable to connect to $myservername"
#   $Disconnected = $True
# }
# If (!$Disconnected)
# {
#   try {Collect-Data -EA stop}
#   catch {
#     $localEvent.descriptionDetails = $_
#     $localEvent.status = 'gray'
#     $localEvent.eventShort = "Unable to collect data from $myservername"
#     $Uncollected = $True
#   }
# }

################################### ANALYZE #######################################
<# this is custom #>
# If (!$Disconnected -and !$Uncollected)
# {
#   <Run-MyAnalyzer...>
# }

################# COMPARE WITH LAST EVENT AND WRITE NEW EVENT #####################
If (Test-IfNewEvent $LocalEvent)
{
  Write-Event $LocalEvent
}

############################## WRITE HEARTBEAT ####################################
Write-HeartBeat $settings.detectorid

################################### CLEANUP #######################################
<# this is custom #>
```
Basically, this is the template to be used by every detector, and even though the detectors do very different things using very different approaches, all of them, so far, has been able to follow this basic template.

## Global settings file
To make a future migrations and changes in the environment a little simpler, and also having one source of truth for some general settings, I decided to create a *globalSettings* file. This file is loaded by all detectors on startup.
#### globalSettings.ini
```powershell
$rootPath = "E:\CURE"

<### DATABASE SETTINGS #####>
$dbServer = "my-db-server.fqdn"
$dbName = "cure"
$dbUsername = "cureDbUser"
$inventoryTable = "inventoryTable"
$eventTable = "eventTable"
$eventDescriptionTable = "eventDescriptionTable"
```

## Putting it together
So here's a screen shot of the actual folder structure on the detector host.
![Detector folder structure](/assets/images/detector-folder-structure.png)
In the **detectors** folder I put all the individual detector scripts.
The **log** folder contains crash-logs.
**modules** contains *Credential.psm1*, *ODBC.psm1*, *Detector.psm1*, *CURE.psm1* and all other custom module files used by one or more future and/or present detector.
**other** is used for detector specific settings and/or data.
In **Tools** basically only the *DetectorTemplate.ps1* file resides.

Below is a table of all the general functions written and weather they are used by the detector and/or on the console.

| _Function_                | _Detector_ | _Console_ | _Module_        | _Description_                                                        |
|:--------------------------|:-----------|:----------|:----------------|:---------------------------------------------------------------------|
| Convert-CredentialToBasic | true       | true      | Credential.psm1 | Convert a saved credential to base64                                 |
| Receive-Credential        | true       | true      | Credential.psm1 | Get a saved credential in either PSCredential or PlainText           |
| Remove-Credential         | false      | true      | Credential.psm1 | Remove a saved credential                                            |
| Save-Credential           | false      | true      | Credential.psm1 | Save a credential to local machine                                   |
| Search-Credential         | false      | true      | Credential.psm1 | Search for a saved credential on local machine                       |
| Set-Credential            | false      | true      | Credential.psm1 | Update a saved local credential                                      |
| Get-ODBCData              | true       | true      | ODBC.psm1       | For retreiving data from postgres                                    |
| Set-ODBCData              | true       | true      | ODBC.psm1       | For inserting data into postgres                                     |
| Get-Detector              | false      | true      | Detector.psm1   | For maintaining detector settings in the console                     |
| Get-DetectorEvents        | false      | true      | Detector.psm1   | For maintaining detector settings in the console                     |
| New-Detector              | false      | true      | Detector.psm1   | For maintaining detector settings in the console                     |
| Remove-Detector           | false      | true      | Detector.psm1   | For maintaining detector settings in the console                     |
| Set-Detector              | false      | true      | Detector.psm1   | For maintaining detector settings in the console                     |
| Get-DefaultLocalEvent     | true       | false     | CURE.psm1       | For detector to receive a pre-formated object to add data to         |
| Get-DetectorSettings      | true       | false     | CURE.psm1       | For detector to get it's setting from database on startup            |
| Get-ItemCount             | true       | false     | CURE.psm1       | For detector to get correct item count on PSObjects with 1 occurence |
| Test-IfNewEvent           | true       | false     | CURE.psm1       | For detector to decide if it should write a new event to databse     |
| Test-IfSnoozed            | true       | false     | CURE.psm1       | For detector to check if snoozed and therefore not run               |
| Write-Event               | true       | false     | CURE.psm1       | For detector to write event data                                     |
| Write-HeartBeat           | true       | false     | CURE.psm1       | For detector to write heart beat to database                         |
| Write-Log                 | true       | false     | CURE.psm1       | For detector to write a crash-log locally                            |


In a future post I will give an example step-by-step for setting up a new detector, so hopefully all the stuff described in this post will make some sense!

*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*


