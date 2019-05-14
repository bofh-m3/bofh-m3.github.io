---
layout: single
title: "CURE - Detector Temperature"
excerpt: "CURE - The homebrewed monitoring solution. In this post I'll describe the steps for setting up a detector monitoring Temperature in the server room."
date: 2018-11-26
comments: true
permalink: /Detector-Temperature.html
tags:
  - cure
  - powershell
  - datacenter
  - temperature
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
We have a small server OnPrem, mainly for hosting network equipment and some backup servers. The server room has all the stuff a server room usually [do](/VMWare.html), but in a light version. One of the recurring problems has been cooling. We have installed a backup system for cooling, and it's set to kick in when the main system fails.
However, I needed a way to monitor ambient temperature in the server room, to get an indication *before* hardware started complaining about overheating. So some years ago (maybe 5 or 6) I bought a network attached thermometer, [HWg-STE](https://www.hw-group.com) that was really cheap and simple but suited my needs. That ethernet thermometer has been serving me well ever since! But I didn't want notifications in mail, I wanted it up on the CURE dashboard, so I set about writing a detector for it.

I will describe the steps I did for setting up a CURE detector for Temperature based on the *DetectorTeamplate.ps1* described in [CURE-Detector foundation](/CURE-Detector-foundation.html) and the instructions in [CURE-Detector-step-by-step](/CURE-Detector-step-by-step.html).
The whole detector script can be found [here](https://github.com/bofh-m3/CURE/blob/master/Detectors/Temperature.ps1)

#### Approach
Since there's no API or CLI available for this very basic device, only a web GUI is available, I had to resort to parsing html to get what I needed.

#### LOCAL SETTINGS, FUNCTIONS AND DEPENDENCIES
```powershell
$detectorName = "Temperature"
$thermo = "MyHWg-STE.fqdn"
$lowyellow = 18
$lowred = 16
$highyellow = 28
$highred = 30
$historyevents = 100
```
In the local settings part, I defined the thresholds (high and low) for when to make the detector turn yellow and red.

#### CONNECT AND COLLECT
```powershell
try {$fetch = Invoke-RestMethod -Method Get -Uri "http://$thermo/index_m.asp" -EA stop -WA silentlycontinue}
catch {
  $localEvent.descriptionDetails = $_
  $localEvent.status = 'grey'
  $localEvent.eventShort = "Unable to connect to $thermo"
  $Disconnected = $True
}
```
This is the *CONNECT AND COLLECT* section of the script. I decided to collect the current readings data from the mobile version of the GUI, since it was cleaner and easier to parse.
Please Note! This kind of cheap and early IoT devices are usually very unsecure! Therefore, I have placed it in an isolated network, since it doesn't need access to anything except for incoming traffic!

#### ANALYZE
```powershell
<# this is custom #>
If (!$Disconnected)
{
  $report = Get-ODBCData -query "SELECT datetime,status,eventshort FROM $eventTable WHERE detectorId = $($localevent.detectorID) ORDER BY eventId DESC LIMIT $historyevents"
  $report | Add-Member -MemberType NoteProperty -Name "reading" -Value $null
  foreach ($i in $report)
  {
    if ($i.eventshort -like "Current temperature is *") {$i.reading = $i.eventshort -replace 'Current temperature is '}
  }
  $temp = ($fetch -split '<td  id=alse215><div class="value" id="s215">' | select -Index 1) -split '&nb' | select -Index 0
  $localEvent.descriptionDetails = ($report | select reading,@{n="date";e={$_.datetime.ToString()}},status,eventshort) | ConvertTo-Json
  $localEvent.contentType = "json"
  $localEvent.eventShort = "Current temperature is $temp"
  if (($temp -lt $lowred) -or ($temp -gt $highred)) {$localEvent.status = 'red'}
  elseif (($temp -lt $lowyellow) -or ($temp -gt $highyellow)) {$localEvent.status = 'yellow'}
  elseif (($temp -ge $lowyellow) -and ($temp -le $highyellow)) {$localEvent.status = 'green'}
  else {$localEvent.status = 'grey'}
}
```
On row 4 I extract the 100 previous readings from the CURE database. I wanted to have that in the report, to easily be able to see what the temperature reading was during the last handful of hours. 
On rows 5-9 I extract the previous readings values and add them into the "reading" property.
On row 10 I parse the current reading value from the html collected in the *CONNECT AND COLLECT* section. 
On rows 10-17 I fill the *localEvent* object with the results, ready to be shoved into the database.

#### Result
Here's how the detector would look in the UI with a warning.
![Detector temperature overview](/assets/images/detector-temperature-overview.png)
And if you click the headline you get the details of the historic readings.
![Detector temperature details](/assets/images/detector-temperature-details.png)



*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*
