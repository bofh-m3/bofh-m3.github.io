---
layout: single
title: "CURE - Detector Password Reminder"
excerpt: "CURE - The homebrewed monitoring solution. In this post I'll describe the steps for setting up a detector monitoring our homebrewed Password Reminder solution."
date: 2019-01-10
comments: true
permalink: /Detector-Password-Reminder.html
tags:
  - cure
  - powershell
  - password
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
Some years ago, I wrote a Password Reminder solution to remind users their AD password would expire in X days. Ok, some may ask themselves "why the hell do you have passwords with expire date!?". Well, for a couple of reasons:
- To make sure users don't put their AD credentials into services and scripts (if they do it will fail when they quit)
- To guard ourselves from password stuffing attacks
I know it generally forces users to choose weaker passwords, but hey, that's what password managers are for!
Anyways, because of *reasons*, all AD passwords must be changed after 90 days. That causes headaches for the users, and also helpdesk, so the approach I chose was to:
1. Set up self-service [password management](/Detector-PWM-Health.html) that could be accessed remotely
2. Write a solution for sending out Password Reminders when the password was about to expire to users, on e-mail and on [Slack](/Slack.html).
The good thing with sending the password reminders on Slack, is that our Slack setup is *not* SSO, so the login to Slack is totally independent from AD. Therefore the user that, for example, is away on holiday and forgets to change his/her password, able to see the instructions in Slack, and set a new password, even though he/she has been locked out of Outlook. 

I will describe the steps I did for setting up a CURE detector for monitoring the company Password Reminder solution on the *DetectorTeamplate.ps1* described in [CURE-Detector foundation](/CURE-Detector-foundation.html) and the instructions in [CURE-Detector-step-by-step](/CURE-Detector-step-by-step.html).
The whole detector script can be found [here](https://github.com/bofh-m3/CURE/blob/master/Detectors/PasswordReminder.ps1)

#### Approach
I will not go into detail of the the Password Reminder setup, I might in a future post, but basically there's a script to populate an SQL table with static information, such as what AD users belongs to what Slack user. Also, there's a script to gather all AD user's with expiring passwords, send email and slack to them, and shove the result into an event table in the SQL database. The scripts are run daily, and each run gets it's unique BatchID.
It's from that event table I read the results with this Detector script, in order to catch if there are any failed notifications.
So, basically, it's just a matter of getting prepared event data out of the sql db, and show any errors/warnings in CURE.

#### LOCAL SETTINGS, FUNCTIONS AND DEPENDENCIES
```powershell
$detectorName = "Password Reminder"
$sqlServer = "MySQLServer.fqdn"
$sqlDb = "MyDB"
$sqlTable = "MyTable"
$logHistoryDays = 30
$hoursLastRunRed = 60
$hoursLastRunYellow = 40
$ignoreEmails = @("some.user1@somewhere","some.user2@company","some.user3@company")
function Connect-SQL {
  [CmdletBinding()]
  param ([string]$Server,[string]$Database)
  if ($SQLDefaultConnection) {$SQLDefaultConnection.Dispose()}
  $global:SQLDefaultConnection=New-Object System.Data.SqlClient.SQLConnection
  $SQLDefaultConnection.ConnectionString="Server=$Server;Database=$Database;Integrated Security=True;"
}
function Invoke-SQL {
  [CmdletBinding()]
  param ([string]$Query,[object]$Connection,[string]$SurpessExceptionString)
  if (!$Connection) 
  {
    if (!$SQLDefaultConnection) {Connect-SQL}
    $Connection=$SQLDefaultConnection
  }
  if (!$SurpessExceptionString) {$SurpessExceptionString="\*"}
  if ($Connection.state -notlike "open") {$Connection.Open()}
  $Command=$Connection.CreateCommand()
  $Command.CommandText=$Query
  try {$Result=$Command.ExecuteReader()}
  catch 
  {
    if ($_ -notmatch $SurpessExceptionString)
    {
      write-error $_
    }
    $Connection.Close()
  }
  if ($Result)
  {
    $Table = new-object "System.Data.DataTable"
    $Table.Load($Result)
    $Connection.Close()
    return $Table
  }
}
```
In the *Local Settings* section, on rows 2-4 I define the settings for connecting to my Password Reminder database. And since the database is in *Microsoft SQL* I needed to write functions to interact with that on rows 9-44. On row 5 I set how many days history to show in the report. On rows 6-7 I set thresholds for how many days since last run is ok before turning the detector yellow/red.
  
#### CONNECT AND COLLECT
```powershell
<# this is custom #>
try {Connect-SQL -Server $sqlServer -Database $sqlDb -ea stop}
catch {
  $localEvent.descriptionDetails = $_
  $localEvent.status = 'grey'
  $localEvent.eventShort = "Unable to connect to $passwordremindersqlsrvr"
  $Disconnected = $True
}

If (!$Disconnected)
{
  try {$latestBatch = Invoke-SQL -Query "declare @latest_id int; select @latest_id=MAX(batch_id) from $sqlTable; select batch_id from $sqlTable where batch_id=@latest_id" -ea stop}
  catch {
    $localEvent.descriptionDetails = $_
    $localEvent.status = 'grey'
    $localEvent.eventShort = "Unable to collect data from db $sqlDb table $sqlTable"
    $Uncollected = $True
  }
} 
```
In the *CONNECT AND COLLECT* section of the script I use my *Connect-SQL* and *Invoke-SQL* functions to retrieve the results from the latest run.

#### ANALYZE
```powershell
<# this is custom #>
If (!$Disconnected -and !$Uncollected)
{
  $startBatch = ($latestBatch[0].batch_id - $logHistoryDays)
  $endBatch = $latestBatch[0].batch_id

  $log = Invoke-SQL -Query "select * from $sqlTable where batch_id BETWEEN $startBatch AND $endBatch"

  $report = $log | where {$_.type -notmatch "BatchStart|BatchEnd"} | sort batch_id,date -descending | select `
    @{n="date";e={$_.date.ToString()}},`
    @{n="batch_id";e={$_.batch_id}},`
    @{n="type";e={$_.type}},`
    @{n="EmailAddress";e={$_.EmailAddress}},`
    @{n="Name";e={$_.Name}},`
    @{n="OU";e={$_.OU}},`
    @{n="outlook_notified";e={$_.outlook_notified}},`
    @{n="slack_notified";e={$_.slack_notified}},`
    @{n="state";e={$_.status}},`
    @{n="message";e={$_.message}},`
    @{n="acknowledged";e={$_.acknowledged}},`
    @{n="status";e={$null}},`
    @{n="ignore";e={$false}}

  $latestRun = ($log.date | sort -Descending | select -Index 0)
  $hoursSinceLastRun = ((get-Date) - $latestRun).totalhours

  foreach ($i in $report)
  {
    if (![string]::IsNullOrEmpty($i.acknowledged)) {$i.status = "green"}
    else
    {
      if ([string]::IsNullOrEmpty($i.acknowledged)) {$i.acknowledged = $false}
      if ($i.state -like "Error") {$i.status = "red"}
      elseif (($i.EmailAddress -in $ignoreEmails) -and ($i.state -like "warning")) {$i.status = "green"; $i.ignore = $true}
      elseif ($i.state -like "Warning") {$i.status = "yellow"}
      elseif (($i.state -like "Success") -or ($i.state -like "Info")) {$i.status = "green"}
      else {$i.status = "grey"}
    }
  }

  $noRed = Get-ItemCount ($report | where {$_.status -like "red"})
  $noYellow = Get-ItemCount ($report | where {$_.status -like "yellow"})
  $noGreen = Get-ItemCount ($report | where {$_.status -like "green"})
  $noGrey = Get-ItemCount ($report | where {$_.status -like "grey"})

  $localEvent.eventShort = "$noRed red events, $noYellow yellow events, $noGreen green events, $noGrey grey events, $($latestRun.ToString())"

  if (($noRed -gt 0) -or ($hoursSinceLastRun -gt $hoursLastRunRed)) {$localEvent.status = "red"}
  elseif (($noYellow -gt 0) -or ($hoursSinceLastRun -gt $hoursLastRunYellow)) {$localEvent.status = "yellow"}
  elseif ($noGrey -gt 0) {$localEvent.status = "grey"}
  else {$localEvent.status = "green"}

  $localEvent.descriptionDetails = ($report | ConvertTo-Json)
  $localEvent.contentType = "json"
}
```
In the *ANALYZE* section of the script, on rows 4-7 I collect the history for the amount of days defined in *$logHistoryDays*. On rows 9-22 I build the *$report* object based on the events. On rows 27-39 I set the status of each event and on rows 41-54 I set the over all status and populate the *$localEvent* object with details, making it ready to shove into the CURE database.

#### Result
Here's how the detector would look in the UI with a warning.

![Detector password reminder overview](/assets/images/detector-password-reminder-overview.png)

And if you click the headline you get the details of notifications that was sent.

![Detector password reminder details](/assets/images/detector-password-reminder-details.png)


*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*




