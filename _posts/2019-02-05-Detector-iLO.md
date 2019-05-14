---
layout: single
title: "CURE - Detector iLO"
excerpt: "CURE - The homebrewed monitoring solution. In this post I'll describe the steps for setting up a detector monitoring alerts in HPE iLO."
date: 2019-02-05
comments: true
permalink: /Detector-iLO.html
tags:
  - cure
  - powershell
  - ilo
  - hpe
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
iLO (Integrated Lights Out), is HPEs take on out-of-bounds server management. iLO is basically a computer-on-a-card, running independently of the OS on the physical server, allowing admins to monitor and control the server independently of the Server OS. This of course is very handy, and one of the great features is that iLO maintains an *Integrated Management Log* that basically keep track of the hardware components in the server host, creating alerts when something fails. Independently of the Server OS.
Of course this is not something unique to HPE Servers, all server vendors has their OOBM solutions. But since the company is running Proliant server as hosts for our [ESX](/VMWare.html) cluster, I decided to put monitoring of our iLO logs up on display in CURE.

I will describe the steps I did for setting up a CURE detector for monitoring iLO logs based on the *DetectorTeamplate.ps1* described in [CURE-Detector foundation](/CURE-Detector-foundation.html) and the instructions in [CURE-Detector-step-by-step](/CURE-Detector-step-by-step.html).
The whole detector script can be found [here](https://github.com/bofh-m3/CURE/blob/master/Detectors/iLO.ps1)

#### Approach
There's a REST API available for iLO. That's good.
What's not good is the *documentation*. As per usual with HPE you must navigate fucking myriads of portals, logins and crap before you're able to get some decent information. And that information is usually contained within a *PDF*! What the hell HP, wake up and smell the 21st century! 
Some years ago, I had found some HPE official module named *HPRESTCmdlets* and was able to extract the *actual* REST calls needed to extract log data from iLO. 
However, those APIs had been discontinued in more resent versions of iLO, so I needed to exercise my googling skills once again. After a while a came across [HPERedfishCmdlets](https://www.powershellgallery.com/packages/HPERedfishCmdlets/1.0.0.2) that would suite my needs. I still didn't like the idea of loading a whole module just to make a simple REST call to extract the log data, but hey, I'm also lazy so I settled for using the official module.

#### LOCAL SETTINGS, FUNCTIONS AND DEPENDENCIES
```powershell
$detectorName = "iLO"
$hosts = @("MyiLOHost1.fqdn","MyiLOHost2.fqdn","MyiLOHost3.fqdn","MyiLOHost4.fqdn","MyiLOHost5.fqdn","MyiLOHost6.fqdn","MyiLOHost7.fqdn")
$user = "myUserName"
$logSources = @("/redfish/v1/Systems/1/LogServices/IML/Entries/")
```
In the *Local Settings* section, on row 2 I define an array with my target iLO hosts. On row 4 I created an array with *logSources*, thinking I could collect both the internal iLO logs and also the IML logs. That didn't work as I liked, so for now I only collect IML from the iLO hosts. 
  
#### CONNECT AND COLLECT
```powershell
<# this is custom #>
$report = @()
$sessions = @()
foreach ($h in $hosts)
{
  $cError = $null
  $cSession = "" | select `
    @{n="Name";e={$h}},`
    @{n="Session";e={$null}},`
    @{n="Message";e={$null}},`
    @{n="Status";e={$null}}

  try {$cConnect = Connect-HPERedfish -Address $h -Username $user -Password (Receive-Credential -SavedCredential $user -Type ClearText) -DisableCertificateAuthentication -EA stop}
  catch {
    $cError = $_.ToString()
    $report += "" | select `
      @{n="Target";e={$h}},`
      @{n="Source";e={"Cure"}},`
      @{n="Id";e={$null}},`
      @{n="Created";e={(get-date -Format "yyyy-MM-dd HH:mm:ss")}},`
      @{n="Type";e={"Connect"}},`
      @{n="Message";e={$cError}},`
      @{n="State";e={"Error"}},`
      @{n="Status";e={"red"}}
  }

  if (!$cError)
  {
    if (($cConnect.RootUri -like "https://$h/redfish/v1/") -and (![string]::IsNullOrEmpty($cConnect.'X-Auth-Token')) -and ($cConnect.Location -like "https://$h/redfish/v1/SessionService/Sessions/$user*"))
    {
      $cSession.Session = $cConnect
      $cSession.Status = "green"
      $sessions += $cSession
    }
    else
    {
      $cSession.Message = "Unknown connection status"
      $cSession.Status = "grey"
    }
  }
}

if ($sessions.Status -match "green")
{
  foreach ($s in $sessions)
  {
    if ($s.status -like "green")
    {
      foreach ($url in $logSources)
      {
        try {$cEntries = Get-HPERedfishDataRaw -Odataid $url -Session $s.session -DisableCertificateAuthentication -EA stop}
        catch {
          $errMsg = $_
          $report += "" | select `
            @{n="Target";e={$s.Name}},`
            @{n="Source";e={"Cure"}},`
            @{n="Id";e={$null}},`
            @{n="Created";e={(get-date -Format "yyyy-MM-dd HH:mm:ss")}},`
            @{n="Type";e={"Collect"}},`
            @{n="Message";e={$errMsg}},`
            @{n="State";e={"Error"}},`
            @{n="Status";e={"red"}}
          Continue
        }
        foreach ($m in $cEntries.members)
        {
          try {$cEvent = Get-HPERedfishDataRaw -Odataid $m.'@odata.id' -Session $s.session -DisableCertificateAuthentication -EA stop}
          catch {
            $errMsg = $_
            $report += "" | select `
              @{n="Target";e={$s.Name}},`
              @{n="Source";e={($url -split '/')[-3]}},`
              @{n="Id";e={($m -split '/')[-2]}},`
              @{n="Created";e={(get-date -Format "yyyy-MM-dd HH:mm:ss")}},`
              @{n="Type";e={"Collect"}},`
              @{n="Message";e={$errMsg}},`
              @{n="State";e={"Error"}},`
              @{n="Status";e={"red"}}
            Continue
          }
          $report += "" | select `
            @{n="Target";e={$s.Name}},`
            @{n="Source";e={$cEvent.Name}},`
            @{n="Id";e={$cEvent.Id}},`
            @{n="Created";e={$cEvent.Created}},`
            @{n="Type";e={$cEvent.EntryType}},`
            @{n="Message";e={$cEvent.Message}},`
            @{n="State";e={$cEvent.Severity}},`
            @{n="Status";e={$null}}
        }
      }
    }

    else
    {
      $report += "" | select `
        @{n="Target";e={$s.Name}},`
        @{n="Source";e={"Cure"}},`
        @{n="Id";e={$null}},`
        @{n="Created";e={(get-date -Format "yyyy-MM-dd HH:mm:ss")}},`
        @{n="Type";e={"Collect"}},`
        @{n="Message";e={$errMsg}},`
        @{n="State";e={"Error"}},`
        @{n="Status";e={"red"}}
    }
  }
}

else
{
  $localEvent.descriptionDetails = $report | convertto-json
  $localEvent.contentType = "json"
  $localEvent.status = 'grey'
  $localEvent.eventShort = "Unable to connect to any iLO hosts"
  $Uncollected = $True
}
```
Ok, this is just crazy! I thought at least by using a precompiled module, I wouldn't need to create as much code. But boy was I wrong! There's probably a more elegant approach, and in hindsight I regret not having spent more time extracting the *actual* REST call needed to get the log data out. 
But mainly I'm just pissed at HPE. Don't fucking hide shit in modules when there's a REST API! Just document it with Swagger or something and let everyone choose how to approach the APIs for solving *their* particular problem!
Anayways.
In the *CONNECT AND COLLECT* section of the script I need to loop through the *$hosts* and handle any connection errors separately by adding that particular host to the *$report* object. It's just crazy much coding and parsing needed to do this! Thanks for nothing HPE.
I don't even want to go into detail of the code, but basically, it leaves me with a *$sessions* object containing all the iLO hosts session tokens to be used for subsequent calls populating the *$report* object with IML data, ready to be analyzed.

#### ANALYZE
```powershell
<# this is custom #>
If (!$Uncollected)
{
  foreach ($i in $report)
  {
    if (!$i.status)
    {
      if ($i.state -match "Error|Critical") {$i.status = "red"}
      elseif ($i.state -match "Warning") {$i.status = "yellow"}
      elseif ($i.state -match "OK") {$i.status = "green"}
      else {$i.status = "grey"}
    }
  }
  [datetime]$latestupdate = ($report.created | sort -Descending | select -Index 0)
  $noRed = Get-ItemCount ($report | where {$_.Status -like "red"})
  $noYellow = Get-ItemCount ($report | where {$_.Status -like "yellow"})
  $noGreen = Get-ItemCount ($report | where {$_.Status -like "green"})
  $noGrey = Get-ItemCount ($report | where {$_.Status -notmatch "red|green|yellow"})
  $localEvent.eventShort = "$noRed Red alerts, $noYellow Yellow alerts, $nogreen Green alerts, $nogrey Grey alerts, $($latestupdate.ToString())"
  if ($noRed -gt 0) {$localEvent.status = "red"}
  elseif ($noYellow -gt 0) {$localEvent.status = "yellow"}
  elseif ($noGrey -gt 0) {$localEvent.status = "grey"}
  else {$localEvent.status = "green"}
  $localEvent.descriptionDetails = ($report | ConvertTo-Json)
  $localEvent.contentType = "json"
}
```
In the *ANALYZE* section of the script, on rows 4-13, I set the individual statuses of the alerts. Also, any iLO connection alerts generated in the *CONNECT AND COLLECT* section are skipped since they are already set to *red/grey*. 
On rows 14-25 I set the overall status, populate the *$localEvent* object with data, making it ready to be shoved into the CURE database.

#### Result
Here's how the detector would look in the UI with an IML warning.

![Detector ilo overview](/assets/images/detector-ilo-overview.png)

And if you click the headline you get the details.

![Detector ilo details](/assets/images/detector-ilo-details.png)


*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*

