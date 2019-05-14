---
layout: single
title: "CURE - Detector Remote Office"
excerpt: "CURE - The homebrewed monitoring solution. In this post I'll describe the steps for setting up a detector monitoring a remote company office."
date: 2019-01-02
comments: true
permalink: /Detector-Remote-Office.html
tags:
  - cure
  - powershell
  - dhcp
  - fortigate
  - slack
  - slm
  - off-shoring
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
The company has an [off-shoring](/SLM.html) office in Kharkiv, Ukraine. I wanted monitoring of the connectivity there in more detail, since they had no local service infrastructure, just a [FortiGate](/FortiGate.html) cluster. The basic network services (DHCP, DNS) was managed over IPSec back in the central [data center](/Consolidated-Data-Center.html).

I will describe the steps I did for setting up a CURE detector for monitoring a company remote office based on the *DetectorTeamplate.ps1* described in [CURE-Detector foundation](/CURE-Detector-foundation.html) and the instructions in [CURE-Detector-step-by-step](/CURE-Detector-step-by-step.html).
The whole detector script can be found [here](https://github.com/bofh-m3/CURE/blob/master/Detectors/RemoteOffice.ps1)

#### Approach
To be honest, the IT staff back at the off-shoring office is kind of clueless. They take every opportunity they can to blame us when stuff isn't working. So I needed a way to quickly verify that our equipment and services (firewall, ipsec, dhcp, dns) was up and running, and that we had clients in Kharkiv connected.
I decided to monitor on these two things:
- The latest DHCP lease
- The percentage of devices that responds to ping
Also, to be fair, there has been major problems with the current firmware for FortiGate, and the Kharkiv branch has had regular firewall crashes due to cpu maxing out at 100% for hours at the time. Fortunately, while waiting for FortiNet to get their shit together and release an updated firmware, we are able to reboot the firewall cluster in Kharkiv remotely to solve the problem temporarily.
So this detector, unlike the other CURE detectors, actually has actions in the script (notify on [slack](/Slack.html) and reboot firewall)!

#### LOCAL SETTINGS, FUNCTIONS AND DEPENDENCIES
```powershell
$detectorName = "Remote Office"
$dhcpserver = "myDHCPserver.fqdn"
$ScopeId = "10.125.100.0"
$username = "myUser"
$LeaseExpiresHours = 24
$upYellow = 15
$upRed = 5
$latestLeaseYellowHours = 2
$latestLeaseRedHours = 6
$autoRebootFirewallEnabled = $true
$FirewallIP = "10.125.100.1"
$FirewallUser = "myFortiUser"
$postSlack = $true
$ITStafSlackIDs = @("ABC123","ABC456")
$SlackWebHook = "https://hooks.slack.com/services/XXX/YYY/ZZZ"
$SlackChannel = "#MyChannel"
$SlackBotName = "CURE"
```
In the *Local Settings* section, on rows 2-5 I define the Microsoft DHCP server settings. On rows 6-9 I set the threshold values for when to turn the detector Yellow/Red in CURE. On rows 10-12 I have some firewall settings and on 13-17 are slack settings. 

#### CONNECT AND COLLECT
```powershell
<# this is custom #>
try {$leases = Invoke-Command -ComputerName $dhcpserver -ScriptBlock {Get-DhcpServerv4Lease -ScopeId $ScopeId} -Credential (Receive-Credential -SavedCredential $username) -ea stop}
catch {
  $localEvent.descriptionDetails = $_
  $localEvent.contentType = "text"
  $localEvent.status = 'grey'
  $localEvent.eventShort = "Unable to connect to $dhcpserver to collect active leases"
  $Disconnected = $True
}
If (!$Disconnected)
{
  $report = @()
  foreach ($l in $leases)
  {
    $report += $l | select @{n="Client";e={$l.HostName}},`
      @{n="IP";e={$l.IPAddress}},`
      @{n="LeaseExpires";e={$l.LeaseExpiryTime.ToString()}},`
      @{n="IsUp";e={(Test-Connection -ComputerName $l.IPAddress -Count 1 -Quiet)}}
  }
}
```
In the *CONNECT AND COLLECT* section of the script I just issue a remote powershell command to get the current lease information from the DHCP server. I then, on rows 66-73, loop through the leases and ping the devices, collecting the result in the *$results* object.

#### ANALYZE
```powershell
<# this is custom #>
If (!$Disconnected -and !$Uncollected)
{
  $IsUp = [math]::round((($report | where {$_.IsUp}).count / $report.count),2) * 100
  $latestLease = $leases.LeaseExpiryTime | sort -Descending | select -Index 0
  $latestLeaseHours = ((Get-Date) - $latestlease.AddHours(-$LeaseExpiresHours)).hours
  if (($IsUp -lt $upRed) -and ($latestLeaseHours -ge $latestLeaseRedHours)) {$localEvent.status = "red"}
  elseif (($IsUp -lt $upYellow) -and ($latestLeaseHours -ge $latestLeaseYellowHours)) {$localEvent.status = "yellow"}
  elseif (($IsUp -ge $upYellow) -or ($latestLeaseHours -lt $latestLeaseYellowHours)) {$localEvent.status = "green"}
  else {$localEvent.status = "grey"}
  $localEvent.eventShort = "$IsUp% of PCs responded, Latest DHCP-lease $latestLeaseHours hours ago."
  $localEvent.descriptionDetails = ($report | sort LeaseExpires -Descending | ConvertTo-Json)
  $localEvent.contentType = "json"
}
```
In the *ANALYZE* section of the script, on row 4, I calculate the percentage of the devices with active leases that are responding to ping. On row 6 I calculate hours passed since last lease. On rows 7-13 I fill the *$localEvent* object with status data to make it ready to shove into the CURE database.

#### COMPARE WITH LAST EVENT AND WRITE NEW EVENT
```powershell
If (Test-IfNewEvent $LocalEvent)
{
  Write-Event $LocalEvent
  if ($IsUp -eq 0)
  {
    if ($postSlack)
    {
      $payload = @{
       "channel" = $SlackChannel;
       "username" = $SlackBotName;
       "text" = "$(($ITStafSlackIDs | %{'<@' + $_ + '>'}) -join ', ') Company Remote Office seem to be down";
       "icon_emoji" = ":skull:"
      }
      Invoke-RestMethod -Method Post -Uri $SlackWebHook -Body ($payload | ConvertTo-Json)
    }
    if ($autoRebootFirewallEnabled)
    {
      $payload.text = "Auto reboot firewall is enabled. Initiate reboot sequence in t minus 120 sec"
      Invoke-RestMethod -Method Post -Uri $SlackWebHook -Body ($payload | ConvertTo-Json)
      sleep -s 120
      import-module Posh-SSH
      New-SSHSession -ComputerName $FirewallIP -Credential (Receive-Credential -SavedCredential $FirewallUser) -AcceptKey
      $session = Get-SSHSession -Index 0
      $stream = $session.Session.CreateShellStream("reboot", 0, 0, 0, 0, 1000)
      $stream.Write("execute reboot comment CURE`n")
      $stream.Write("y`n")
      $resultReboot = $stream.Read()
      Remove-SSHSession -Index 0
      $payload.text = "Firewall result: $resultReboot"
      Invoke-RestMethod -Method Post -Uri $SlackWebHook -Body ($payload | ConvertTo-Json)
    }
  }
}
```
Like I mentioned, usually the CURE detectors don't perform any actions, but in this case, the firewall being a bit wobbly, it made sense to also add some action! 
In the *COMPARE WITH LAST EVENT* section, on row 1, I validate that this is a new event compared to the latest event in the CURE database. If it's *True*, the script will validate if there are zero devices responding to ping on row 4 and continue to verify if to write a notification to slack (row 6) and reboot firewall (row 16).
On rows 8-14, a message is compiled and posted to slack.
On rows 18-30, the firewall is rebooted using the [Posh-SSH](https://www.powershellgallery.com/packages/Posh-SSH/2.0.2) module, and the results are posted to slack. This is of course a temporary solution, at least the automatic rebooting of the firewall. As soon as the new firmware has landed, that should be sorted, and the *$autoRebootFirewallEnabled* can be set to *$false* again.

#### Result
Here's how the detector would look in the UI with a warning.

![Detector remote office overview](/assets/images/detector-remote-office-overview.png)

And if you click the headline you get the details of current leases.

![Detector remote office details](/assets/images/detector-remote-office-details.png)


*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*


