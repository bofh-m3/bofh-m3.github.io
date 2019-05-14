---
layout: single
title: "CURE - Detector Office365 Health"
excerpt: "CURE - The homebrewed monitoring solution. In this post I'll describe the steps for setting up a detector monitoring our homebrewed Password Reminder solution."
date: 2019-01-14
comments: true
permalink: /Detector-Office365-Health.html
tags:
  - cure
  - powershell
  - office365
  - saas
  - microsoft
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
If you're familiar with [Office 365](/Office365.html) as an admin, you also are aware that all communication regarding current incidents with the service is done through *Sevice Status* in the admin portal. So, events communicated are basically custom per tenant and therefore only accessible when authenticated. I wanted to put the current service status up on the CURE display, in order for IT so quickly see the current status of the platform and all its services.

I will describe the steps I did for setting up a CURE detector for monitoring the company tenant Office 365 health based on the *DetectorTeamplate.ps1* described in [CURE-Detector foundation](/CURE-Detector-foundation.html) and the instructions in [CURE-Detector-step-by-step](/CURE-Detector-step-by-step.html).
The whole detector script can be found [here](https://github.com/bofh-m3/CURE/blob/master/Detectors/O365Health.ps1)

#### Approach
I found this [article](https://www.lazyexchangeadmin.com/2018/10/shd365.html) by Lazy Exchange Admin, that really helped me get going. Since health information is published per Office 365 tenant, I needed to authenticate to the company tenant through Microsoft Graph in order to get our custom health information, and the guide was very helpful in setting that up.
The [Office 365 Management APIs](https://docs.microsoft.com/en-us/office/office-365-management-api/) unfortunately, at the time of writing, is not very good to work with, and I had to put in way more effort and code to get what I wanted than anticipated from this simple task.

#### LOCAL SETTINGS, FUNCTIONS AND DEPENDENCIES
```powershell
$detectorName = "O365 Health"
$greenAlerts = @("FalsePositive","ServiceRestored","ServiceOperational","Service restored","False positive","Service operational")
$redSeverity = @("Critical","High","Sev0","Sev1")
$forceGreen = @("SomeIssueIDtoSupress1","SomeIssueIDtoSupress2","SomeIssueIDtoSupress3")
$ClientSecret = Receive-Credential -SavedCredential "MyO365Secret" -Type ClearText
$ClientID = "MyClientIDGUID"
$tenantdomain = "MyO365tenant"
function Get-O365Health {
  [CmdletBinding()]
  param ([Switch]$AllMessages,[string]$ID),[String]$ClientSecret,[string]$ClientID,[string]$tenantdomain)
  begin
  {
    $body = @{grant_type="client_credentials";resource="https://manage.office.com";client_id=$ClientID;client_secret=$ClientSecret}
    $oauth = Invoke-RestMethod -Method Post -Uri "https://login.microsoftonline.com/$($tenantdomain)/oauth2/token?api-version=1.0" -Body $body
    $headerParams = @{'Authorization'="$($oauth.token_type) $($oauth.access_token)"}
  }
  process
  {
    if ($AllMessages)
    {
      Return (Invoke-RestMethod -Uri "https://manage.office.com/api/v1.0/$($tenantdomain)/ServiceComms/Messages" -Headers $headerParams -Method Get)
    }
    elseif ($ID)
    {
      Return (Invoke-RestMethod -Uri "https://manage.office.com/api/v1.0/$($tenantdomain)/ServiceComms/Messages?ID=$ID" -Headers $headerParams -Method Get)
    }
    else
    {
      Return (Invoke-RestMethod -Uri "https://manage.office.com/api/v1.0/$($tenantdomain)/ServiceComms/CurrentStatus" -Headers $headerParams -Method Get)
    }
  }
}
```
In the *Local Settings* section, on rows 2-3 I define arrays with statuses that are to be considered "green" or "red". Since I have not been able to find such lists of statuses, I needed to compile it by heart and will probably need to update them continuously. On row 4 I keep an array of incident IDs to force green, it usually is problems with services we don't use, or incidents that doesn't concern us. I suspect this list will grow over time, and I also may need to move it a more dynamic approach, maybe even an "Acknowledge" button in the CURE UI. But that's for a future version. For now, we just add incident IDs to this array to suppress them.
On rows 5-7 I set the tenant specific stuff and on rows 8-32 there's a simple function to extract event data with the Microsoft Management API.
  
#### CONNECT AND COLLECT
```powershell
<# this is custom #>
try {$status = Get-O365Health -ClientSecret $ClientSecret -ClientID $ClientID -tenantdomain $tenantdomain -EA stop -WA silentlycontinue}
catch {
  $localEvent.descriptionDetails = $_
  $localEvent.status = 'grey'
  $localEvent.eventShort = "Unable to connect to O365 Health API"
  $Disconnected = $True
} 
```
In the *CONNECT AND COLLECT* section of the script I use my *Get-O365Health* function to retrieve the current Office 365 health status. 

#### ANALYZE
```powershell
<# this is custom #>
If (!$Disconnected)
{
  $incidents = $status.value | where {$_.status -notin $greenAlerts}
  if (!$incidents)
  {
    $localEvent.status = 'green'
    $localEvent.eventShort = "$(Get-ItemCount $status.value) services OK"
  }
  else
  {
    $report = @()
    $messages = Get-O365Health -AllMessages -ClientSecret $ClientSecret -ClientID $ClientID -tenantdomain $tenantdomain
    foreach ($i in $incidents.IncidentIds)
    {
      $report += ($messages.value | where {$_.id -eq $i}) | select `
        @{n="Id";e={$_.id}},`
        @{n="Title";e={$_.Title}},`
        @{n="State";e={$_.Status}},`
        @{n="Workload";e={$_.Workload}},`
        @{n="ActionType";e={$_.ActionType}},`
        @{n="Classification";e={$_.Classification}},`
        @{n="Feature";e={$_.Feature}},`
        @{n="ImpactDescription";e={$_.ImpactDescription}},`
        @{n="LastUpdatedTime";e={([datetime]$_.LastUpdatedTime).ToString()}},`
        @{n="StartTime";e={([datetime]$_.StartTime).ToString()}},`
        @{n="Severity";e={$_.Severity}},`
        @{n="Status";e={$null}}
    }
    foreach ($e in $report)
    {
      if (($e.Id -in $forceGreen) -or ($e.State -in $greenAlerts)) {$e.status = "green"}
      elseif ($e.Severity -in $redSeverity) {$e.status = "red"}
      else {$e.status = "yellow"}
    }
    $localEvent.descriptionDetails = ($report | sort status | ConvertTo-Json)
    $localEvent.contentType = "json"
    if ($report.status -contains "red") {$localEvent.status = "red"}
    elseif ($report.status -notcontains "yellow") {$localEvent.status = "green"}
    else {$localEvent.status = "yellow"}
    if ((($report | where {$_.status -notlike "green"}).Workload | select -Unique).count -le 4) {$localEvent.eventShort = "$((($report | where {$_.status -notlike "green"}).Workload | select -Unique) -join ', ')"}
    else {$localEvent.eventShort = "$((($report | where {$_.status -notlike "green"}).Workload | select -Unique).count) Workloads with issues"}
    if ($report.count -eq 1) {$localEvent.eventShort = "$($report.workload), $($report.LastUpdatedTime)"}
    else {$localEvent.eventShort += ", $(($report.LastUpdatedTime | sort -descending)[0])"}
  }
}
```
In the *ANALYZE* section of the script, on row 4, I filter out the incidents I want to process. On rows 5-9 I handle the scenario where there are no active alerts (yea right, hasn't happened so for, seems there is always something broken over at Redmond). 
On rows 11-45 I handle the scenario where there are active alerts. Since the API is kind of crappy, and I have not found a way to get filtering of incidents to work, I need to download *all* messages in order to extract the details of the active incidents collected. On rows 14-29 I loop through the *$incidents* and match it to an actual message, create an event object with the information I need and add it to *$report*.
On rows 30-35 I loop through the *$report* object to set statuses for each incident. On rows 36-44 I finally set the over all status and populate the *$localEvent* object, making it ready to be shoved into the CURE database.
Worth noticing, on rows 41-42 I truncate how many broken services to be displayed in the CURE UI. Since there sometimes are *many* broken services over at Redmond, I limit the number to spell out to 4 to not break the UI.

#### Result
Here's how the detector would look in the UI with a couple of active incidents (during the last 2 months, I have seen it green only once...)

![Detector o365health overview](/assets/images/detector-o365-health-overview.png)

And if you click the headline you get the details of the current incidents.

![Detector o365health details](/assets/images/detector-o365-health-details.png)


*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*





