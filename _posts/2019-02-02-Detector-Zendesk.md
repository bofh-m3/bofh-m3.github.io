---
layout: single
title: "CURE - Detector Zendesk"
excerpt: "CURE - The homebrewed monitoring solution. In this post I'll describe the steps for setting up a detector monitoring support tickets in Zendesk."
date: 2019-02-02
comments: true
permalink: /Detector-Zendesk.html
tags:
  - cure
  - powershell
  - zendesk
  - helpdesk
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
The company is using different ticket management tools in the different branch offices. In one of the branches we're using Zendesk, and of course I wanted to monitor it so there are no unanswered tickets laying around.

I will describe the steps I did for setting up a CURE detector for monitoring Zendesk tickets based on the *DetectorTeamplate.ps1* described in [CURE-Detector foundation](/CURE-Detector-foundation.html) and the instructions in [CURE-Detector-step-by-step](/CURE-Detector-step-by-step.html).
The whole detector script can be found [here](https://github.com/bofh-m3/CURE/blob/master/Detectors/Zendesk.ps1)

#### Approach
Zendesk has a very well documented set of [APIs](https://developer.zendesk.com/rest_api/docs/support/introduction) to interact with it's services, so the whole thing was pretty straight forward to set up. 

#### LOCAL SETTINGS, FUNCTIONS AND DEPENDENCIES
```powershell
$detectorName = "Zendesk"
$user = "myUserName/token"
$apikey = (Receive-Credential -SavedCredential "MyAPIKey" -Type ClearText)
$cred = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(("{0}:{1}" -f $user,$apikey)))
$baseurl = "https://myInstance.zendesk.com/api/v2/"
$unansweredRed = 4
$unansweredYellow = 2
function Get-RequesterName {
  param ($id)
  return ($users | where {$_.id -eq $id}).name
}
function Get-ZendeskData {
  [CmdletBinding()]
  param ($endpoint)
  [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
  $result=@()
  $fetch = Invoke-RestMethod -Headers @{Authorization=("Basic {0}" -f $cred)} -Uri "$baseurl$endpoint.json" -Method Get -ContentType "application/json"
  $result += $fetch.$endpoint
  while (![string]::IsNullOrEmpty($fetch.next_page))
  {
    $url=$fetch.next_page
    $fetch = Invoke-RestMethod -Headers @{Authorization=("Basic {0}" -f $cred)} -Uri $url -Method Get -ContentType "application/json"
    $result+=$fetch.$endpoint
  }
  return $result
}
```
In the *Local Settings* section, on rows 2-5 I define the settings specific for our Zendesk instance. On rows 6-7 I define the thresholds for when an unanswered ticket should turn yellow/red. On rows 8-11 is a simple function to map a user ID to a plain text name. On rows 12-26 is a function written by my colleague Micke to get data out of the different API endpoints. 
  
#### CONNECT AND COLLECT
```powershell
<# this is custom #>
try {$users = Get-ZendeskData -endpoint users -EA stop}
catch {
  $localEvent.descriptionDetails = $_
  $localEvent.status = 'grey'
  $localEvent.eventShort = "Unable to connect to $baseurl"
  $Disconnected = $True
}
If (!$Disconnected)
{
  try {$tickets = Get-ZendeskData -endpoint tickets -EA stop}
  catch {
    $localEvent.descriptionDetails = $_
    $localEvent.status = 'grey'
    $localEvent.eventShort = "Unable to connect to $baseurl"
    $Uncollected = $True
  }
}
```
In the *CONNECT AND COLLECT* section of the script I use the *Get-ZendeskData* function to retrieve all the users (rows 2-8) and all open tickets (rows 10-18). Since we're on a small subscription, we didn't want to hit the connect limit in zendesk, and therefore decided to just download *all* users each time in order to match a user id to a plain text name. Not very lean, but what to do.

#### ANALYZE
```powershell
<# this is custom #>
If (!$Disconnected -and !$Uncollected)
{
  $report = $tickets | where {$_.status -notlike "closed"} | select id,subject,`
    @{n="requester";e={(Get-RequesterName $_.requester_id)}}, `
    @{n="assignee";e={(Get-RequesterName $_.assignee_id)}}, `
    @{n="state";e={$_.status}}, `
    @{n="tags";e={$_.tags -join ','}}, `
    created_at,updated_at, `
    @{n="noanswer_hours";e={(Get-Workhours $_.updated_at)}},`
    @{n="status";e={$null}}
  
  foreach ($t in $report)
  {
    if (($t.noanswer_hours -gt $unansweredRed) -and ($t.state -like "new")) {$t.status = "red"}
    elseif (($t.noanswer_hours -gt $unansweredYellow) -and ($t.state -like "new")) {$t.status = "yellow"}
    else {$t.status = "green"}
  }
  [datetime]$latestupdate = ($tickets.updated_at | sort -Descending | select -Index 0)
  $noNew = Get-ItemCount ($report | where {$_.state -like "new"})
  $noRed = Get-ItemCount ($report | where {$_.status -like "red"})
  $noYellow = Get-ItemCount ($report | where {$_.status -like "yellow"})
  $noOpen = Get-ItemCount ($report | where {$_.state -notlike "solved"})
  if ($noRed -gt 0) {$localEvent.status = "red"}
  elseif ($noYellow -gt 0) {$localEvent.status = "yellow"}
  else {$localEvent.status = "green"}
  $localEvent.eventShort = "$noNew new issues, $noRed urgent issues, $noYellow late issues, $noOpen open issues, $($latestupdate.ToString())"
  $localEvent.descriptionDetails = ($report | sort status,noanswer_hours -Descending | ConvertTo-Json)
  $localEvent.contentType = "json"
}
```
In the *ANALYZE* section of the script, on rows 4-11, the *$report* object is populated with open tickets. On rows 5-6 the *Get-RequesterName* function is used to map a plain text name to the user id in the tickets, and on row 10, the *Get-Workhours* function described in this [post](/Detector-Redmine-Issues.html) is used to calculate the business hours passed since the ticket was created or replied to.
On rows 13-18 the *$report* object is looped to set status for each ticket and on rows 19-29, the over all status is set in the *$localEvent* object, preparing it to be shoved into the CURE database.

#### Result
Here's how the detector would look in the UI with some unanswered (late) tickets.
![Detector zendesk overview](/assets/images/detector-zendesk-overview.png)
And if you click the headline you get the details of the current open tickets.
![Detector zendesk details](/assets/images/detector-zendesk-details.png)


*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*

