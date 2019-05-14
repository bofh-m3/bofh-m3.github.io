---
layout: single
title: "CURE - Detector Zoho Desk"
excerpt: "CURE - The homebrewed monitoring solution. In this post I'll describe the steps for setting up a detector monitoring support tickets in Zoho Desk."
date: 2019-02-03
comments: true
permalink: /Detector-Zoho-Desk.html
tags:
  - cure
  - powershell
  - zoho
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
The company is using different ticket management tools in the different branch offices. In one of the branches we're using Zoho Desk, and of course I wanted to monitor it so there are no unanswered tickets laying around.

I will describe the steps I did for setting up a CURE detector for monitoring Zoho tickets based on the *DetectorTeamplate.ps1* described in [CURE-Detector foundation](/CURE-Detector-foundation.html) and the instructions in [CURE-Detector-step-by-step](/CURE-Detector-step-by-step.html).
The whole detector script can be found [here](https://github.com/bofh-m3/CURE/blob/master/Detectors/ZohoDesk.ps1)

#### Approach
Just like [Zendesk](/Zendesk.html), [Zoho Desk](https://desk.zoho.com/DeskAPIDocument) has a very well documented set of APIs.  The whole thing was pretty straight forward to set up. However, just like Zendesk, there are limits on the frequency and amount of API callas you're allowed to do. I will not go into detail of the system, but we've been running Zoho for some months now, mainly to compare it to Zendesk, and my verdict is that it's similar but less polished. The app is definitely not as good as Zendesk's. But the price for Zoho is *much* more attractive.

#### LOCAL SETTINGS, FUNCTIONS AND DEPENDENCIES
```powershell
$detectorName = "Zoho Desk"
$token = Receive-Credential -SavedCredential MyAPIKey -Type ClearText
$orgid = "ORGID123"
$url = "https://desk.zoho.com/api/v1/tickets?status=new,in progress,on hold,escalated,open&limit=99"
$header = @{'Authorization' = $token; 'orgId' = $orgid}
$unansweredRed = 4
$unansweredYellow = 2
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
```
In the *Local Settings* section, on row 4 I define the URL of the API endpoint I want to use, including a filter on the tickets based on their status, since I'm not interested in the closed ticket for this detector. On row 8 I force TLS 1.2 to avoid certificate errors on the Detector Host. 
  
#### CONNECT AND COLLECT
```powershell
<# this is custom #>
try {$tickets = Invoke-RestMethod -Uri $url -Headers $header -Method GET -ContentType "application/json" -ea stop}
catch {
  $localEvent.descriptionDetails = $_.ToString() -replace "'"
  $localEvent.status = 'grey'
  $localEvent.eventShort = "Unable to connect to Zoho"
  $Disconnected = $True
}
```
In the *CONNECT AND COLLECT* section of the script I just *GET* all the open tickets by querying using my *$url* defined in the *LOCAL SETTINGS* section. Simple as that.

#### ANALYZE
```powershell
<# this is custom #>
If (!$Disconnected -and !$Uncollected)
{
  $report = $tickets.data | where {$_.statusType -like "open"} | select ticketNumber,email,subject,createdTime,@{n="state";e={$_.status}},threadCount,customerResponseTime,@{n="lastThreadDirection";e={$_.lastThread.direction}},@{n="noReplyHours";e={$null}},@{n="status";e={$null}}
  foreach ($i in $report)
  {
    if ($i.state -like "On Hold") {$i.status = "green"}
    elseif (($i.lastThreadDirection -like "in") -or ($i.state -like "new"))
    {
      if ($i.threadCount -gt 1)
      {
        $i.noReplyHours = Get-Workhours $i.customerResponseTime
      }
      else
      {
        $i.noReplyHours = Get-Workhours $i.createdTime
      }
      if ($i.noReplyHours -ge $unansweredRed) {$i.status = "red"}
      elseif ($i.noReplyHours -ge $unansweredYellow) {$i.status = "yellow"}
      else {$i.status = "green"}
    }
    elseif (!$i.lastThreadDirection) {$i.status = "grey"}
    else {$i.status = "green"}
  }
  [datetime]$latestupdate = ($tickets.data.createdTime | sort -Descending | select -Index 0)
  $noNew = Get-ItemCount ($tickets.data | where {$_.status -like "new"})
  $noOpen = Get-ItemCount ($tickets.data | where {$_.statusType -like "open"})
  $noRed = Get-ItemCount ($report | where {$_.status -like "red"})
  $noYellow = Get-ItemCount ($report | where {$_.status -like "yellow"})
  $noGrey = Get-ItemCount ($report | where {$_.status -like "grey"})
  if ($noRed -gt 0) {$localEvent.status = "red"}
  elseif ($noYellow -gt 0) {$localEvent.status = "yellow"}
  elseif ($noGrey -gt 0) {$localEvent.status = "grey"}
  else {$localEvent.status = "green"}
  $localEvent.eventShort = "$noNew new issues, $noRed urgent issues, $noYellow late issues, $noOpen open issues, $($latestupdate.ToString())"
  $localEvent.descriptionDetails = ($report | sort createdTime -Descending | ConvertTo-Json)
  $localEvent.contentType = "json"
}
```
In the *ANALYZE* section of the script, on row 4, I extract the tickets, selecting the members I'm interested in, and add them to the *$report* object.
On rows 5-24 I loop through *$report' to set the status of the individual tickets. On row 8 I force the ticket status "On hold" to *green*. 
On rows 8-21 I handle the scenario where the last reply on the ticket is *inbound*, meaning it's something for Helpdesk to answer. The hours passed since the last reply is calculated and a status *red/yellow/green* is set.
On rows 25-37 the over all status is set, *$localEvent* object is populated with data, making it ready to be shoved into the CURE database.

#### Result
Here's how the detector would look in the UI with some unanswered (late) tickets.

![Detector zoho desk overview](/assets/images/detector-zoho-desk-overview.png)

And if you click the headline you get the details of the current open tickets.

![Detector zoho desk details](/assets/images/detector-zoho-desk-details.png)


*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*







