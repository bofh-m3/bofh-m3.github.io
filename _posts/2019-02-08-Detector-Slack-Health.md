---
layout: single
title: "CURE - Detector Slack Health"
excerpt: "CURE - The homebrewed monitoring solution. In this post I'll describe the steps for setting up a detector monitoring Slack Health."
date: 2019-02-08
comments: true
permalink: /Detector-Slack-Health.html
tags:
  - cure
  - powershell
  - slack
  - saas
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
The company has used [Slack](/Slack.html) for messaging and collaboration for a long time (like 4 years now). And since it's to be considered our main communication tool (or maybe second to email, not sure) I wanted to have some health monitoring on it.

I will describe the steps I did for setting up a CURE detector for monitoring Slack Health based on the *DetectorTeamplate.ps1* described in [CURE-Detector foundation](/CURE-Detector-foundation.html) and the instructions in [CURE-Detector-step-by-step](/CURE-Detector-step-by-step.html).
The whole detector script can be found [here](https://github.com/bofh-m3/CURE/blob/master/Detectors/SlackHealth.ps1)

#### Approach
Like all born-in-the-cloud SaaS, Slack has a modern and easy to consume set of APIs. One could even argue they set the standard for how to do it right! And the Health API is no exception. Also, as frosting on the cake, the Health API is exposed without authentication, making it a breeze to work with!

#### LOCAL SETTINGS, FUNCTIONS AND DEPENDENCIES
```powershell
$detectorName = "Slack Health"
$okStatus = @("ok","resolved")
```
That's it! Two rows! In the *Local Settings* section I set the *$detectorName* and an array holding the statuses for *green*.
  
#### CONNECT AND COLLECT
```powershell
<# this is custom #>
try {$status = Invoke-RestMethod -Method Get -Uri "https://status.slack.com/api/current" -EA stop -WA silentlycontinue}
catch {
  $localEvent.descriptionDetails = $_
  $localEvent.status = 'grey'
  $localEvent.eventShort = "Unable to connect to Slack Status Current API"
  $Disconnected = $True
}

If (!$Disconnected)
{
  if ($status.status -notin $okStatus)
  {
    try {$history = Invoke-RestMethod -Method Get -Uri "https://status.slack.com/api/history" -EA stop -WA silentlycontinue}
    catch {
      $localEvent.descriptionDetails = $_
      $localEvent.status = 'grey'
      $localEvent.eventShort = "Unable to connect to Slack Status History API"
      $Uncollected = $True
    }
  }
}
```
In the *CONNECT AND COLLECT* section of the script, on rows 2-8 I connect to slack *current* endpoint to get the current overall health status. 
On rows 10-21, if *current* status is not *ok*, I collect the historical messages to get details of the current disruption(s).

#### ANALYZE
```powershell
<# this is custom #>
If (!$Disconnected -and !$Uncollected)
{
  if ($status.status -notin $okStatus)
  {
    $report = $history | where {$_.status -notlike "resolved"} | select id,`
     @{n="created";e={([datetime]$_.date_created).ToString()}},`
     @{n="updated";e={([datetime]$_.date_updated).ToString()}},`
     @{n="title";e={$_.title}},`
     @{n="type";e={$_.type}},`
     @{n="state";e={$_.status}},`
     @{n="services";e={$_.services -join ','}},`
     @{n="latest_details";e={$_.notes[0].body}},`
     @{n="status";e={$null}}

    $report.latest_details = $report.latest_details -replace "'"
    $report.title = $report.title -replace "'"

    foreach ($i in $report)
    {
      if ($i.type -like "outage") {$i.status = "red"}
      else {$i.status = "yellow"}
    }

    $localEvent.descriptionDetails = ($report | ConvertTo-Json)
    if ($status.type -like "outage") {$localEvent.status = 'red'}
    else {$localEvent.status = 'yellow'}
    $localEvent.eventShort = "$($status.title), $(([datetime]$status.date_updated).ToString())" -replace "'"
    $localEvent.contentType = "json"
  }
  else
  {
    $localEvent.descriptionDetails = ($status | ConvertTo-Json)
    $localEvent.status = 'green'
    $localEvent.contentType = "json"
    $localEvent.eventShort = "$($status.status), $(([datetime]$status.date_updated).ToString())"
  }
}
```
In the *ANALYZE* section of the script, on rows 4-30 I handle the scenario where there's active alerts. The details of the disruption(s) are put in a PSCustomObject and added to the *$report* object. On rows 25-29 the *$localEvent* object is populated, making it ready to be shoved into the CURE database. 
On rows 31-37 the scenario where there is no active alerts is handled.

#### Result
Here's how the detector would look in the UI with a couple of active alerts.
![Detector slack health overview](/assets/images/detector-slack-health-overview.png)
And if you click the headline you get the details of the alerts.
![Detector slack health details](/assets/images/detector-slack-health-details.png)


*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*

