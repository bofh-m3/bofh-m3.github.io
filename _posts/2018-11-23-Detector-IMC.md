---
layout: single
title: "CURE - Detector IMC"
excerpt: "CURE - The homebrewed monitoring solution.  In this post I'll describe the steps for setting up a detector monitoring IMC alerts."
date: 2018-11-23
comments: true
permalink: /Detector-IMC.html
tags:
  - cure
  - powershell
  - imc
  - networking
  - datacenter
  - hpe
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
HP IMC (Intelligent Management Center, yes very lame name) is used for monitoring networking equipment at the company. IMC basically aggregates alerts and telemetrics from switches, firewalls etc. using SNMP. It of course is geared towards HPE equipment but can be hammered into shape for other vendors as well. So, at the company, IMC is the main source for networking related event data.
And there's a REST API available!
Ok, it's limited unless you pay for it, but hey, at least it exists!

I will describe the steps I did for setting up a CURE detector for IMC based on the *DetectorTeamplate.ps1* described in [CURE-Detector foundation](/CURE-Detector-foundation.html) and the instructions in [CURE-Detector-step-by-step](/CURE-Detector-step-by-step.html).
The whole detector script can be found [here](https://github.com/bofh-m3/CURE/blob/master/Detectors/IMC.ps1)

#### Approach
Back when I first started to interact with the IMC API, documentation was very sparse. That may have changed however. I found a crappy PowerShell module provided by HP, but it was almost useless to me, so I dissected it to get what I needed for my use case.
Also, I needed to create a user in IMC with permission to read event data.

#### LOCAL SETTINGS, FUNCTIONS AND DEPENDENCIES
```powershell
$detectorName = "IMC"
$cred = (Receive-Credential -SavedCredential "myIMCUser")
$imcserver = "myIMCHost.fqdn"
$uri = 'http://' + $imcserver + ':8080/imcrs/fault/alarm?start=0&size=1000&isAdmin=true'
$knownStatuses = @("Critical","Major","Minor","Info")
$ignore = @{
 "MyDevice1" = "NMS Performance Alarm";
 "MyDevice2" = "Another alarm type"
}
```
This is the detector specific settings I ended up with.
I decided to create an *ignore* hash table in order to easily be able to suppress alarms I didn't want to see.

#### CONNECT AND COLLECT
```powershell
<# this is custom #>
try {$alarms = Invoke-RestMethod -Uri $uri -method GET -Credential $cred -ContentType "application/xml" -EA stop -WA stop}
catch {
  $localEvent.descriptionDetails = $_
  $localEvent.status = 'grey'
  $localEvent.eventShort = "Unable to connect to $imcserver"
  $Disconnected = $True
}
```
This is the *CONNECT AND COLLECT* section of the script. The interface being REST, connect and collect is done in the same command (Invoke-RestMethod). 

#### ANALYZE
```powershell
<# this is custom #>
If (!$Disconnected)
{
  If (!$alarms.list.alarm)
  {
    $localEvent.descriptionDetails = "Unable to receive alarms from $imcserver"
    $localEvent.status = 'grey'
    $localEvent.eventShort = "Unable to receive alarms from $imcserver because the alarm array is empty"
  }
  else
  {
    $imcalarms = $alarms.list.alarm | where {($_.ackStatusDesc -match "Unacknowledged") -and ($_.alarmLevelDesc -notmatch "Info")  -and ($_.recStatusDesc -notlike "Recovered")} | select `
    @{n="Device";e={$_.deviceName}},`
    @{n="Severity";e={$_.alarmLevelDesc}},`
    @{n="Alarm";e={$_.alarmCategoryDesc}},`
    @{n="Time";e={$_.faultTimeDesc}},`
    @{n="Status";e={"grey"}},`
    @{n="Ignore";e={$false}}
    If (!$imcalarms)
    {
      $localEvent.descriptionDetails = "No active alerts on $imcserver"
      $localEvent.status = 'green'
      $localEvent.eventShort = "No active alerts on $imcserver"  
    }
    else 
    {
      foreach ($i in $imcalarms)
      {
        if ($ignore.$($i.Device) -like $i.Alarm) {$i.Ignore = $true; $i.Status = "green"}
        elseif ($i.Severity -like "Critical") {$i.Status = "red"}
        elseif ($i.Severity -like "Major") {$i.Status = "yellow"}
        elseif ($i.Severity -like "Minor") {$i.Status = "green"}
        else {$i.Status = "grey"}
      }
      $noCritical = Get-ItemCount ($imcalarms | where {$_.Severity -like "Critical"})
      $noMajor = Get-ItemCount ($imcalarms | where {($_.Severity -like "Major")})
      $noMinor = Get-ItemCount ($imcalarms | where {($_.Severity -like "Minor")})
      $noUnknown = Get-ItemCount ($imcalarms | where {$_.Severity -notin $knownStatuses})
      if ($imcalarms.Status -match "red") 
      {
        $localEvent.status = "red"
      }
      elseif ($imcalarms.Status -match "yellow") 
      {
        $localEvent.status = "yellow"
      }
      elseif ($imcalarms.Status -match "grey") 
      {
        $localEvent.status = "grey"
      }
      Else {$localEvent.status = "green"}
      $localEvent.eventShort = "$noCritical Critical alarms, $noMajor Major alarms, $noMinor Minor alarms"
      if ($noUnknown -gt 0) {$localEvent.eventShort += ", $noUnknown alarms with unknown severity"}
      $localEvent.descriptionDetails = ($imcalarms | ConvertTo-Json)
      $localEvent.contentType = "json"
    }
  }
}
```
In the *ANALYZE* section, on rows 12-18, I filter out the alarms I don't care about. On rows 27-34 I loop through the alarms (if any), validates weather to ignore it, and set a status. On rows 35-55 I set the over all status, populate *$localEvent*, making it ready to be shoved into the CURE database.

#### Result
Here's how the detector would look in the UI with unacknowledged alarms.
![Detector imc overview](/assets/images/detector-imc-overview.png)
And if you click the headline you get the details of the events.
![Detector imc details](/assets/images/detector-imc-details.png)



*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*



