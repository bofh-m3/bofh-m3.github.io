---
layout: single
title: "CURE - Detector PWM Health"
excerpt: "CURE - The homebrewed monitoring solution. In this post I'll describe the steps for setting up a detector monitoring PWM health."
date: 2019-01-06
comments: true
permalink: /Detector-PWM-Health.html
tags:
  - cure
  - powershell
  - pwm
  - password
  - infosec
  - soc
  - active_directory
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
The company needed a unified way to let users manage their passwords in Active Directory, so a couple of years ago I set up (PWM)(https://github.com/pwm-project/pwm), an open source password management solution written in java. PWM has some nice features for self-service password management that suited our needs very well.

I will describe the steps I did for setting up a CURE detector for monitoring health of the company PWM server based on the *DetectorTeamplate.ps1* described in [CURE-Detector foundation](/CURE-Detector-foundation.html) and the instructions in [CURE-Detector-step-by-step](/CURE-Detector-step-by-step.html).
The whole detector script can be found [here](https://github.com/bofh-m3/CURE/blob/master/Detectors/RedmineIssues.ps1)

#### Approach
Since there's an API, available without authentication, for getting health data from PWM, it was simple enough to get what I needed. I decided to also look at the *statistics* end-point, specifically *INTRUDER_ATTEMPTS_DAY*.

#### LOCAL SETTINGS, FUNCTIONS AND DEPENDENCIES
```powershell
$detectorName = "PWM Health"
$url = "https://myPWMServer.fqdn/pwm/public/rest"
$daysRed = 0.2
$daysYellow = 0.1
$ignoreConfigAlerts = @(
  "User Permission configuration for setting Modules ? Authenticated ? Guest Registration ? Guest Admin Permission issue: groupDN: DN '' is invalid.  This may cause unexpected issues."
  "Some other config error i want to supress"
)
function Get-PWMHealth {
  [CmdletBinding()]
  param ([string]$BaseURL)
  $header=@{"Content-Type"="application/json"}
  $curl=$BaseURL+'/health'
  $health=invoke-restmethod $curl -headers $header
  if ($health.error) {throw "$($health.errorCode) $($health.errorMessage)"}
  return $health.data
}
function Get-PWMStatistics {
  [CmdletBinding()]
  param ([int]$Days,[string]$statKey,[string]$statName,[string]$BaseURL)
  $header=@{"Accept"="application/json"}
  $body=@{}
  if ($Days) {$body.days=$Days}
  if ($statKey) {$body.statKey=$statKey}
  if ($statName) {$body.statName=$statName}
  $curl=$BaseURL+'/statistics'
  $stats=invoke-restmethod $curl -headers $header -body $body -ea stop
  if ($stats.error) {throw "$($stats.errorCode) $($stats.errorMessage)"}
  return $stats.data.eps
}
```
In the *Local Settings* section, on rows 3-4 I define the thresholds for *intruder attempts*, on rows 5-8 I have an array with *CONFIG* warnings I want to ignore (make green in CURE). On rows 9-30 I wrote 2 functions for interacting with the *health* and  *statistics* endpoints. I figured I could re-use them somewhere else, and it would be easier to maintain the detector script if they were easily accessable. 

#### CONNECT AND COLLECT
```powershell
<# this is custom #>
try {$health = Get-PWMHealth -BaseURL $url -EA stop}
catch {
  $localEvent.descriptionDetails = $_
  $localEvent.status = 'grey'
  $localEvent.eventShort = "Unable to connect to $url"
  $Disconnected = $True
}
try {$stats = Get-PWMStatistics -BaseURL $url -EA stop}
catch {
  $localEvent.descriptionDetails += $_
  $localEvent.status = 'grey'
  $localEvent.eventShort += "Unable to connect to $url"
  $Disconnected = $True
}
```
In the *CONNECT AND COLLECT* section of the script I use my *Get-PWMHealth* and *Get-PWMStatistics* functions to retrieve the current readings from PWM.

#### ANALYZE
```powershell
<# this is custom #>
If (!$Disconnected)
{
  $report = $health.records | select @{n="state";e={$_.status}},`
   topic,detail,@{n="status";e={$null}},@{n="ignore";e={$false}},@{n="severity";e={$null}}
  if ([int]$stats.INTRUDER_ATTEMPTS_DAY -ge $daysRed) {$intrusion = "WARN"}
  elseif ([int]$stats.INTRUDER_ATTEMPTS_DAY -ge $daysYellow) {$intrusion = "CAUTION"}
  elseif ([int]$stats.INTRUDER_ATTEMPTS_DAY -eq 0) {$intrusion = "GOOD"}
  else {$intrusion = "UNKNOWN"}

  $report += "" | select @{n="state";e={$intrusion}},`
   @{n="topic";e={"Intrusion"}},`
   @{n="detail";e={"INTRUDER_ATTEMPTS_DAY: $($stats.INTRUDER_ATTEMPTS_DAY)"}},`
   @{n="status";e={$null}},`
   @{n="ignore";e={$false}},`
   @{n="severity";e={$null}}

  foreach ($i in $report)
  {
    if (($ignoreConfigAlerts -contains $i.detail) -and ($i.state -like "CONFIG")) {$i.status="green";$i.ignore=$true; $i.severity=1}
    elseif ($i.state -like "WARN") {$i.status="red"; $i.severity=2}
    elseif (($i.state -like "CAUTION") -or ($i.state -like "CONFIG")) {$i.status="yellow"; $i.severity=1}
    elseif ($i.state -like "GOOD") {$i.status="green"; $i.severity=0}
    else {$i.status="grey"; $i.severity=4}   
  }

  $noRed = Get-ItemCount ($report | where {$_.status-like "red"})
  $noYellow = Get-ItemCount ($report | where {$_.status-like "yellow"})
  $noGreen = Get-ItemCount ($report | where {$_.status-like "green"})
  $noGrey = Get-ItemCount ($report | where {$_.status-like "grey"})
  $localEvent.eventShort = "$noRed Red alarts, $noYellow Yellow alerts, $noGreen Green alerts"
  if ($report.Status -match "red") {$localEvent.status = "red"}
  elseif ($report.Status -match "yellow") {$localEvent.status = "yellow"}
  elseif ($report.Status -match "grey") {$localEvent.status = "grey"; $localEvent.eventShort += ", $noGrey Grey alarts"}
  else {$localEvent.status = "green"}
  $localEvent.descriptionDetails = ($report | sort severity -Descending | ConvertTo-Json)
  $localEvent.contentType = "json"
}
```
In the *ANALYZE* section of the script, on rows 4-5 I create the *$report* object based on the *$health* reading. On rows 6-9 I check if there's any intrusion readings in *$stats* and on rows 11-16 I add a row to $report with the current intrusion state.
On rows 18-25 I loop through *$report* and set a status for each alert and on rows 27-37 I feed *$localEvent* with the results, preparing it to be shoved into the CURE database.

#### Result
Here's how the detector would look in the UI with a warning.

![Detector PWM health overview](/assets/images/detector-pwm-health-overview.png)

And if you click the headline you get the details of current alerts.

![Detector PWM health details](/assets/images/detector-pwm-health-details.png)


*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*



