---
layout: single
title: "CURE - Detector Veeam Backup"
excerpt: "CURE - The homebrewed monitoring solution.  In this post I'll describe the steps for setting up a detector monitoring Veeam backups."
date: 2018-11-24
comments: true
permalink: /Detector-Veeam-Backup.html
tags:
  - cure
  - powershell
  - veeam
  - backup
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
As [covered](/Veeam.html) in a previous post, Veeam is used for backup of the company's [VMware](/VMWare.html) VMs.  

I will describe the steps I did for setting up a CURE detector for Veeam based on the *DetectorTeamplate.ps1* described in [CURE-Detector foundation](/CURE-Detector-foundation.html) and the instructions in [CURE-Detector-step-by-step](/CURE-Detector-step-by-step.html).
The whole detector script can be found [here](https://github.com/bofh-m3/CURE/blob/master/Detectors/VeeamBackup.ps1)

#### Approach
Luckily there are CmdLets (PowerShell native binary modules) available for Veeam. That makes the interaction easy when building a custom detector. 
I wanted to monitor the success of individual backup jobs, but also wanted to monitor that all jobs have actually ran within a set amount of time.
To get access to the Veeam CmdLets I installed the "management and console" tools on the Detector Host.

#### LOCAL SETTINGS, FUNCTIONS AND DEPENDENCIES
```powershell
asnp VeeamPSSnapIn
$detectorName = "Veeam Backup"
$veeamsrv = "MyVeeamServer.fqdn"
$HoursSinceJobLastRunYellow = 50
$HoursSinceJobLastRunRed = 80
$knownStatuses = @("Success","Failed","Warning")
$warningExceptions = @{
  "MyVM1" = "Changed block tracking cannot be enabled: one or more snapshots present*";
  "MyVM2" = "Unable to truncate Microsoft SQL Server transaction logs*";
  "MyVM3" = "Another warning to suppress*"
}
$JobNameFilter = "BACKUP*"
```
This is the detector specific settings I ended up with.
I decided to create an *$warningExceptions * hash table in order to easily be able to suppress warnings I didn't want to see. 
*$JobNameFilter* is to filter on jobs with a specific name prefix. In my case I don't care about some other one-off jobs and/or archive-jobs, so I've named the Veeam jobs I want to monitor "BACKUP-xxx".

#### CONNECT AND COLLECT
```powershell
<# this is custom #>
try {Connect-VBRServer -Server $veeamsrv -EA stop -WA silentlycontinue}
catch {
  $localEvent.descriptionDetails = $_
  $localEvent.status = 'grey'
  $localEvent.eventShort = "Unable to connect to $veeamsrv"
  $Disconnected = $True
}
If (!$Disconnected)
{
  try {$veeamjobs = Get-VBRJob -EA stop -WA silentlycontinue | where {($_.name -like $JobNameFilter) -and ($_.IsScheduleEnabled)}}
  catch {
    $localEvent.descriptionDetails = $_
    $localEvent.status = 'grey'
    $localEvent.eventShort = "Unable to collect jobs from $veeamsrv"
    $Uncollected = $True
  }
}
```
This is the *CONNECT AND COLLECT* section of the script. The connection is established with *Connect-VBRServer* and with *Get-VBRJob* I collect the backup jobs info. I filter on jobs prefixed with "BACKUP" and also only jobs that has an active schedule.

#### ANALYZE
```powershell
<# this is custom #>
If (!$Disconnected -and !$Uncollected)
{
  If (!$veeamjobs)
  {
    $localEvent.descriptionDetails = "There are no enabled jobs on $veeamsrv"
    $localEvent.status = 'grey'
    $localEvent.eventShort = "There are no enabled jobs on $veeamsrv"
  }
  else
  {
    $report = @()
    $failedjobs = $veeamjobs | where {([math]::Round(((get-date) - $_.Info.ScheduleOptions.LatestRunLocal).totalhours) -gt $HoursSinceJobLastRunYellow) -or ($_.Info.LatestStatus -notlike "Success")}
    if ($failedjobs)
    {
      foreach ($j in $failedjobs)
      {
        $Result = Get-VBRBackupSession | Where {$_.jobId -eq $j.Id.Guid} | Sort EndTimeUTC -Descending | Select -First 1
        $FailedVms=$Result.GetTaskSessions() | where {$_.status -notlike "success"}
        if (!$FailedVms)
        {
          $session=Get-VBRSession -Job $j -Last
          $message = ($session.Log | where {$_.Status -notlike "Succeeded"} | select -expand title) -join ''
          $report += "" | select @{n="JobName";e={$j.name}},@{n="VMName";e={$null}},@{n="State";e={$null}},@{n="Reason";e={$message}},@{n="HoursSinceLastRun";e={[math]::Round(((get-date) - $j.Info.ScheduleOptions.LatestRunLocal).totalhours)}},@{n="Status";e={$null}},@{n="Ignore";e={$false}}
        }
        else
        {
          $report += $FailedVms | select JobName,@{n="VMName";e={$_.Name}},@{n="State";e={$_.Status}},@{n="Reason";e={$_.info.Reason}},@{n="HoursSinceLastRun";e={[math]::Round(((get-date) - $j.Info.ScheduleOptions.LatestRunLocal).totalhours)}},@{n="Status";e={$null}},@{n="Ignore";e={$false}}
        }
      }
      foreach ($r in $report)
      {
        if (!$r.State)
        {
          if ($r.HoursSinceLastRun -gt $HoursSinceJobLastRunRed) {$r.State = "Failed"; $r.Reason = "Job not run in more than $HoursSinceJobLastRunRed hours"}
          elseif ($r.HoursSinceLastRun -gt $HoursSinceJobLastRunYellow) {$r.State = "Warning"; $r.Reason = "Job not run in more than $HoursSinceJobLastRunYellow hours"}
          else {$r.State = "Unknown"; $r.Reason = "Unable to get reason"}
        }
        if (!$r.VMName) {$r.VMName = "-"}
        if ($r.State -like "Failed") {$r.Status = "red"}
        elseif (($r.State -like "Warning") -and ($r.Reason -like $warningExceptions.$($r.VMName))) {$r.Status = "green"; $r.Ignore = $true}
        elseif ($r.State -like "Warning") {$r.Status = "yellow"}
        else {$r.Status = "grey"}
      }
    }
    else
    {
      $report = $veeamjobs | select @{n="JobName";e={$_.name}},@{n="VMName";e={"-"}},@{n="State";e={$_.Info.LatestStatus.ToString()}},@{n="Reason";e={$null}},@{n="HoursSinceLastRun";e={[math]::Round(((get-date) - $_.Info.ScheduleOptions.LatestRunLocal).totalhours)}},@{n="Status";e={"green"}},@{n="Ignore";e={$false}}
    }
    $noRed = (Get-ItemCount ($report | where {$_.Status -like "red"}))
    $noYellow = (Get-ItemCount ($report | where {$_.Status -like "yellow"}))
    $noGrey = (Get-ItemCount ($report | where {$_.Status -like "grey"}))
    $noEnabledJobs = (Get-ItemCount $veeamjobs)
    if ($noRed -ge 1) {$localEvent.status = "red"}
    elseif ($noYellow -ge 1) {$localEvent.status = "yellow"}
    elseif ($noGrey -ge 1) {$localEvent.status = "grey"}
    else {$localEvent.status = "green"}
    $localEvent.eventShort = "$noRed Red alerts, $noYellow Yellow alerts, $noGrey Grey alerts, $noEnabledJobs Enabled jobs"
    $localEvent.descriptionDetails = ($report | ConvertTo-Json)
    $localEvent.contentType = "json"
  }
}
```
Veeam objects turned out to be a bit tricky to work with. I had to dig through a lot of documentation and manually going through all properties of the objects to achieve what I wanted:
- If a job had failed, I wanted to extract which VMs had failed (if any).
- If a job had not run within a set number of hours, I wanted them to be yellow/red.
These prerequisites turned out to be a bit of a brain twister, hence the large amount of code.
Row 14-45 deals with the scenario that one or more jobs have not succeeded and/or not ben run within the set amount of time. On row 16-30 I extract the more exact reasons for the unsuccessful job (Get-VBRSession), and also, if available, the specific VMs that have failed and why (Get-VBRBackupSession). Warnings that should be suppressed is dealt with on row 41.
Row 46-49 deals with the desired outcome, namely all jobs having run successfully within the set time period.

#### Result
Here's how the detector would look in the UI with one failed job, and one job that has not run within the set amount of hours.

![Detector veeam overview](/assets/images/detector-veeam-overview.png)

And if you click the headline you get the details of the events including the VMs that have failed and why.

![Detector veeam details](/assets/images/detector-veeam-details.png)



*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*


