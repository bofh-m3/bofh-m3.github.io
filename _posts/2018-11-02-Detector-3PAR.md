---
layout: single
title: "CURE - Detector 3PAR"
excerpt: "CURE - The homebrewed monitoring solution. In this post I'll describe the steps for setting up a detector monitoring 3PAR alerts."
date: 2018-11-02
comments: true
permalink: /Detector-3PAR.html
tags:
  - cure
  - powershell
  - 3par
  - storage
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
As [covered](/3Par.html) previously, 3PAR is used for production storage in the company. Therefore, I of course wanted to monitor it! In our case we have two, one in Oslo and one in Stockholm.
I will describe the steps I did for setting up a CURE detector for 3PAR based on the *DetectorTeamplate.ps1* described in [CURE-Detector foundation](/CURE-Detector-foundation.html) and the instructions in [CURE-Detector-step-by-step](/CURE-Detector-step-by-step.html).
The whole detector script can be found [here](https://github.com/bofh-m3/CURE/blob/master/Detectors/3Par.ps1)

#### Approach
I have yet found no way to get event data out of 3PAR through an API. The only approach available, to my knowledge, is to use SSH and parse the event data. 
Ok, so I knew I needed a way for PowerShell to log in and get event data through SSH. I also knew that data had to be parsed, since it would not be a nicely structured object. 
I decided to go with Joakim Svendsen's [SSH-Sessions](https://github.com/EliteLoser/SSHSessions) module to be able to make a connection, run a command and snatch up the output. I had used it before and it had proven itself for this kind of tasks. I did not bother to compare the different options available for SSH in PowerShell, but I guess this module could easily be replaced by another approach. Anyways, I downloaded the SSH-Sessions module and put it in the "modules" directory on the Detector host.
Also, of course I needed to create a user on 3PAR-machines with permissions to SSH and to read event data.

#### LOCAL SETTINGS, FUNCTIONS AND DEPENDENCIES
```powershell
$detectorName = "3Par"
Import-Module $rootPath\modules\SSH-Sessions\SSH-Sessions.psm1
Add-Type -Path $rootPath\modules\SSH-Sessions\Renci.SshNet.dll
$username = "username"
$hosts = @("3par1.fqdn","3par2.fqdn")
$knownStatuses = @("Critical","Major","Degraded","Minor","Informational")
function Get-AlertObject {
  param (
      $ihost = $chost,
      $state = $null,
      $time = $(get-date -format 'yyyy-MM-dd HH:mm'),
      $severity = $null,
      $message = $null
    )
  $alertobject = "" | select `
  @{n="Host";e={$ihost}},`
  @{n="State";e={$state}},`
  @{n="Time";e={$time}},`
  @{n="Severity";e={$severity}},`
  @{n="Message";e={$message}}
  Return $alertobject
}
```
This is the detector specific settings I ended up with.
The function *Get-AlertObject* is the custom function I wrote to get a nicely formatted PSCustomObject based on parsed data from the 3PAR event.

#### CONNECT AND COLLECT
```powershell
<# this is custom #>
$alerts = @()
foreach ($chost in $hosts)
{
  $connect = $null
  $collect = $null
  <# connect #>
  try {$connect = New-SshSession -ComputerName $chost -Username $username -Password (Receive-Credential -SavedCredential $username -Type ClearText) -EA stop -WA stop}
  catch {
    $ErrMsg=$_
    $alerts += Get-AlertObject -state "ConnectionError" -severity "Critical" -message  $ErrMsg
    Continue
  }
  If ($connect -notlike "Successfully connected to $chost")
  {
    $alerts += Get-AlertObject -state "ConnectionError" -severity "Critical" -message  $connect
    Continue
  }
  <# collect alerts #>
  try {$collect = Invoke-SshCommand -Command 'showalert -n' -ComputerName $chost -quiet -EA stop -WA stop}
  catch {
    $ErrMsg=$_
    $alerts += Get-AlertObject -state "CollectionError" -severity "Major" -message  $ErrMsg
    Continue
  }
  If (!$collect -match "alerts") 
  {
    $alerts += Get-AlertObject -state "CollectionError" -severity "Major" -message  $collect
    Continue
  }
  <# parse alerts and add to array#>
  If ($collect -match "no alerts")
  {
    $alerts += Get-AlertObject -state "Operational" -severity "Informational" -message  "no alerts"
  }
  else
  {
    $collect = $collect -split 'Id          :'
    $collect = $collect | where {![string]::IsNullOrEmpty($_)}
    foreach ($a in $collect)
    {
      $a = ($a -split '[\r\n]') |? {$_}
      $cstate = ($a | select -index 1) -replace "State       : ",""
      $cdate = ($a | select -index 3) -replace "Time        : ",""
      $cseverity = ($a | select -index 4) -replace "Severity    : ",""
      $ctype = ($a | select -index 5) -replace "Type        : ",""
      $alerts += Get-AlertObject -state $cstate -time $cdate -severity $cseverity -message $ctype
    }
  }
}
```
This is the *CONNECT AND COLLECT* section of the script. The parsing of the 3PAR event data collected through SSH start on row 37. 
It ain't pretty, I know, but what to do when you don't get a proper interface to work with...

#### ANALYZE
```powershell
<# this is custom #>
if (!$alerts)
{
  $localEvent.descriptionDetails = "no result generated when connecting and collecting"
  $localEvent.status = 'grey'
  $localEvent.eventShort = "no result generated when connecting and collecting"
}
else
{
  $noCritical = Get-ItemCount ($alerts | where {$_.Severity -like "Critical"})
  $noMajor = Get-ItemCount ($alerts | where {($_.Severity -like "Major")})
  $noDegraded = Get-ItemCount ($alerts | where {($_.Severity -like "Degraded")})
  $noMinor = Get-ItemCount ($alerts | where {($_.Severity -like "Minor")})
  $noInfo = Get-ItemCount ($alerts | where {($_.Severity -like "Informational")})
  $noUnknown = Get-ItemCount ($alerts | where {$_.Severity -notin $knownStatuses})
  $noNew = Get-ItemCount ($alerts | where {$_.state -like "New"})
  if (($noCritical -gt 0) -or ($noMajor -gt 0))
  {
    $localEvent.status = "red"
    $localEvent.eventShort = "$noCritical Critical alarms, $noMajor Major alarms"
  }
  elseif (($noDegraded -gt 0) -or ($noMinor -gt 0))
  {
    $localEvent.status = "yellow"
    $localEvent.eventShort = "$noDegraded Degraded alarms, $noMinor Minor alarms"
  }
  elseif ($noUnknown -gt 0)
  {
    $localEvent.status = "grey"
    $localEvent.eventShort = "$noUnknown Unknown alarms"
  }
  else
  {
    $localEvent.status = "green"
    $localEvent.eventShort = "$noInfo Informational alarms"
  }
  $localEvent.eventShort += ". $noNew new alarms"
  $localEvent.descriptionDetails = ($alerts | ConvertTo-Json)
  $localEvent.contentType = "json"
}
```
In the *ANALYZE* section I basically just calculate the number of different event statuses, put the info in the *localEvent* object ready to shove into the database.

#### Result
Here's how the detector would look in the UI with unacknowledged alarms.

![Detector 3par overview](/assets/images/detector-3par-overview.png)

And if you click the headline you get the details of the events.

![Detector 3par details](/assets/images/detector-3par-details.png)



*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*


