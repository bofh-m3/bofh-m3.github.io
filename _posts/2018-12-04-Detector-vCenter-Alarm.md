---
layout: single
title: "CURE - Detector vCenter Alarm"
excerpt: "CURE - The homebrewed monitoring solution. In this post I'll describe the steps for setting up a detector monitoring alarm in VMware vCenter."
date: 2018-12-04
comments: true
permalink: /Detector-vCenter-Alarm.html
tags:
  - cure
  - powershell
  - vmware
  - vcenter
  - datacenter
category:
  - cure
---
CURE - The homebrewed monitoring solution. Read more about it in my previous posts:
- [CURE-Design](/CURE-Design.html)
- [CURE-Environment](/CURE-Environment.html)
- [CURE-Database design](/CURE-Database-design.html)
- [CURE-Detector foundation](/CURE-Detector-foundation.html)
- [CURE-Detector-step-by-step](/CURE-Detector-step-by-step.html)
- [CURE-GUI](/CURE-GUI.html)

#### Background
The company has been a user of [VMware](/VMWare.html) for a long time, and naturally I wanted monitoring of the alarms generated in vCenter by CURE.

I will describe the steps I did for setting up a CURE detector for vCenter alarms based on the *DetectorTeamplate.ps1* described in [CURE-Detector foundation](/CURE-Detector-foundation.html) and the instructions in [CURE-Detector-step-by-step](/CURE-Detector-step-by-step.html).
The whole detector script can be found [here](https://github.com/bofh-m3/CURE/blob/master/Detectors/vCenterAlarm.ps1)

#### Approach
VMWare, through PowerCLI, has provided a set of binary cmdlets to get PowerShell to play nice with vCenter. There's a REST API as well, but at the time of writing it's *still* crappy, so I decided to go with PowerCLI in order to extract alarm data from the vCenter server.


#### LOCAL SETTINGS, FUNCTIONS AND DEPENDENCIES
```powershell
$detectorName = "vCenter Alarm"
$myServer = "MyvCenterServer.fqdn"
$knownStatuses = @("red","yellow","green")
Import-Module -Name VMware.VimAutomation.Core
```
In the *Local Settings* section, it's pretty straight forward. I load the module needed for this task (I have already installed PowerCLI on the detector host).

#### CONNECT AND COLLECT
```powershell
<# this is custom #>
try {$session = Connect-VIServer -Server $myServer -EA stop -WA silentlycontinue}
catch {
  $localEvent.descriptionDetails = $_
  $localEvent.status = 'grey'
  $localEvent.eventShort = "Unable to connect to $myServer"
  $Disconnected = $True
}
If (!$Disconnected)
{
  try {$rootFolder = Get-Folder "Datacenters" -EA stop -WA stop}
  catch {
    $localEvent.descriptionDetails = $_
    $localEvent.status = 'grey'
    $localEvent.eventShort = "Unable to collect data from $myServer"
    $Uncollected = $True
  }
}
```
In the *CONNECT AND COLLECT* section of the script I connect the vCenter server and collect the *Datacenters* root folder to extract any active alarms.

#### ANALYZE
```powershell
<# this is custom #>
If (!$Disconnected -and !$Uncollected)
{
  $report = @()
  foreach ($ta in $rootFolder.ExtensionData.TriggeredAlarmState)
  {
    $alarm = "" | Select-Object Entity, EntityType, Reported, Status, Time, Acknowledged, AcknowledgedByUser, AcknowledgedTime
    $alarm.Reported = (Get-View -Server $visrvr $ta.Alarm).Info.Name
    $alarm.Entity = (Get-View -Server $vcsrvr $ta.Entity).Name
    $alarm.EntityType = (Get-View -Server $visrvr $ta.Entity).GetType().Name
    $alarm.Status = $ta.OverallStatus
    $alarm.Time = $ta.Time
    $alarm.Acknowledged = $ta.Acknowledged
    $alarm.AcknowledgedByUser = $ta.AcknowledgedByUser
    $alarm.AcknowledgedTime = $ta.AcknowledgedTime
    $report += $alarm
  }
  if (!$report)
  {
    $localEvent.descriptionDetails = "No active alarms on $myServer"
    $localEvent.status = 'green'
    $localEvent.eventShort = "No active alarms"
  }
  else 
  {

    $noRed = Get-ItemCount ($report | where {($_.Status -like "red") -and (!$_.Acknowledged)})
    $noYellow = Get-ItemCount ($report | where {($_.Status -like "yellow") -and (!$_.Acknowledged)})
    $noGreen = Get-ItemCount ($report | where {($_.Status -like "green")})
    $noUnknown = Get-ItemCount ($report | where {$_.Status -notin $knownStatuses})
    $noAck = Get-ItemCount ($report | where {$_.Acknowledged})
    $localEvent.eventShort = "$noRed Red alarms, $noYellow Yellow alarms"
    if ($noRed -gt 0) {$localEvent.status = "red"}
    elseif ($noYellow -gt 0) {$localEvent.status = "yellow"}
    elseif ($noUnknown -gt 0) 
    {
      $localEvent.status = "grey"
      $localEvent.eventShort += ", $noUnknown Unknown alarms"
    }
    else
    {
      $localEvent.status = "green"
      $localEvent.eventShort = "$noAck Acknowledged alarms"
    }

    $localEvent.descriptionDetails = ($report | ConvertTo-Json)
    $localEvent.contentType = "json"
  }
} 
```
In the *ANALYZE* section of the script, on rows 4-17, I loop through the alarms from *$rootFolder.ExtensionData.TriggeredAlarmState* (if any), create an *$alarm* object for each alarm and use *Get-View* to populate that object with relevant data and add it to the *$report* array. I have stolen this code from the interwebz but I can't remember where so I wont be able to give credit...
On rows 17-22, the scenario where there's no active alarms is handled.
On rows 23-45 the *$localEvent* object is populated with data, preparing it to be shoved into the database.

#### Result
Here's how the detector would look in the UI with a couple of printers low on toner.
![Detector vcenter alarm overview](/assets/images/detector-vcenter-alarm-overview.png)
And if you click the headline you get the details of the detected alarms.
![Detector vcenter alarm details](/assets/images/detector-vcenter-alarm-details.png)



*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*
