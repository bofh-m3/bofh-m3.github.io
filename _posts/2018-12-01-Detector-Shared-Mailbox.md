---
layout: single
title: "CURE - Detector Shared Mailbox"
excerpt: "CURE - The homebrewed monitoring solution.  In this post I'll describe the steps for setting up a detector monitoring unread mail in a shared office365 mailbox."
date: 2018-12-01
comments: true
permalink: /Detector-Shared-Mailbox.html
tags:
  - cure
  - powershell
  - office365
  - exchange
  - saas
  - helpdesk
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
The company has been a long time user of Microsoft [Office365](/office365.html), and the IT team has a shared, impersonalized mailbox for handling subscriptions and stuff.
However, no one on the team is really inclined to monitor that mailbox for new, and potentially important email (that's actually the *reason* for building CURE in the first place), but still, sometimes important stuff is sent to that mailbox. Therefore, I decided to build a CURE detector for monitoring unread emails.
Please note! The mailbox is not a *shared mailbox* in the Exchange sense of the word. The user has a real office license since it's also used for other stuff. I call it "Shared Mailbox" since, well, we use it as a shared mailbox. 
In the future we will probably move it to become a true "shared mailbox", and at that point we'll need to update this detector.

I will describe the steps I did for setting up a CURE detector for an Office365 mailbox based on the *DetectorTeamplate.ps1* described in [CURE-Detector foundation](/CURE-Detector-foundation.html) and the instructions in [CURE-Detector-step-by-step](/CURE-Detector-step-by-step.html).
The whole detector script can be found [here](https://github.com/bofh-m3/CURE/blob/master/Detectors/SharedMailbox.ps1)


#### Approach
I decided to go with the EWS (Exchange Web Services) Managed API. I will consider moving to the REST API in future, but since I had previously worked with EWS it seemed easier for me to get going. 

#### LOCAL SETTINGS, FUNCTIONS AND DEPENDENCIES
```powershell
$detectorName = "Shared Mailbox"
$mailbox = "mailbox@company.fqdn"
$yellowDays = 2
$redDays = 5
$currentDate = (get-date)
Add-Type -Path "$rootPath\modules\Exchange\Web Services\2.2\Microsoft.Exchange.WebServices.dll"
function Get-UnreadMail {
  [CmdletBinding()]
  param ($MailboxName,[object]$Credential)
  $s = New-Object Microsoft.Exchange.WebServices.Data.ExchangeService -ArgumentList Exchange2010_SP1
  $s.Credentials = New-Object Microsoft.Exchange.WebServices.Data.WebCredentials -ArgumentList $Credential.UserName, $Credential.GetNetworkCredential().Password
  $s.Url = new-object Uri("https://outlook.office365.com/EWS/Exchange.asmx");
  $folderid= new-object Microsoft.Exchange.WebServices.Data.FolderId([Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::Inbox,$MailboxName) 
  $inbox = [Microsoft.Exchange.WebServices.Data.Folder]::Bind($s,$folderid)
  $fv = new-object Microsoft.Exchange.WebServices.Data.FolderView(2000)
  $fv.Traversal = "Deep"
  $ffname = new-object Microsoft.Exchange.WebServices.Data.SearchFilter+ContainsSubstring([Microsoft.Exchange.WebServices.Data.FolderSchema]::DisplayName,"Completed Items")
  $folders = $s.findFolders([Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::MsgFolderRoot,$ffname, $fv)
  $completedfolder = $folders.Folders[0]
  $iv = new-object Microsoft.Exchange.WebServices.Data.ItemView(2000)
  $inboxfilter = new-object Microsoft.Exchange.WebServices.Data.SearchFilter+SearchFilterCollection([Microsoft.Exchange.WebServices.Data.LogicalOperator]::And)
  $ifisread = new-object Microsoft.Exchange.WebServices.Data.SearchFilter+IsEqualTo([Microsoft.Exchange.WebServices.Data.EmailMessageSchema]::IsRead,$false)
  $inboxfilter.add($ifisread)
  $msgs = $s.FindItems($inbox.Id, $inboxfilter, $iv)
  if ($msgs.items)
  {
    $psPropertySet = new-object Microsoft.Exchange.WebServices.Data.PropertySet([Microsoft.Exchange.WebServices.Data.BasePropertySet]::FirstClassProperties)
    $psPropertySet.RequestedBodyType = [Microsoft.Exchange.WebServices.Data.BodyType]::Text;
    $s.LoadPropertiesForItems($msgs,$psPropertySet) | out-null
    $result=@()
    foreach ($msg in $msgs.items)
    {
      $cobj= "" | select @{n="Sender";e={$msg.From.ToString()}},@{n="Subject";e={$msg.subject.ToString()}},@{n="Body";e={$msg.body.text.ToString()}},@{n="Received";e={$msg.DateTimeReceived}},@{n="Status";e={$null}}
      $result += $cobj
    }
    return $result
  }
}
```
In the local settings part, I set the thresholds for amount of days for an email to have been left unread. On row 6 I load the EWS binary. On rows 7-38 I wrote a function to extract the unread email from the mailbox using the EWS API.

#### CONNECT AND COLLECT
```powershell
<# this is custom #>
try {$unreadMail = Get-UnreadMail -MailboxName $mailbox -Credential (Receive-Credential -SavedCredential $mailbox) -EA stop -WA silentlycontinue}
catch {
  $localEvent.descriptionDetails = $_
  $localEvent.status = 'grey'
  $localEvent.eventShort = "Unable to connect to $mailbox"
  $Disconnected = $True
}
```
This is the *CONNECT AND COLLECT* section of the script. It basically just runs the function "Get-UnreadMail" described in the *local settings* section. I have previously saved the mailbox credentials using *Save-Credential* function covered in [CURE-Detector foundation](/CURE-Detector-foundation.html).

#### ANALYZE
```powershell
<# this is custom #>
If (!$Disconnected)
{
  If ($unreadMail)
  {
    $report = @()
    ForEach ($m in $unreadMail)
    {
      if ($m.Received -lt $currentDate.AddDays(-$redDays)) {$m.Status = "red"}
      elseif ($m.Received -lt $currentDate.AddDays(-$yellowDays)) {$m.Status = "yellow"}
      else {$m.Status = "green"}
      if ($m.Subject.ToCharArray().count -gt 50) {$currentSubject = $m.Subject.Substring(0,47)+'...'}
      else {$currentSubject = $m.Subject.ToString()}
      if ($m.Body.ToCharArray().count -gt 200) {$currentBody = $m.Body.Substring(0,197)+'...'}
      else {$currentBody = $m.Body.ToString()}
      $report += "" | select `
       @{n="Sender";e={$m.Sender.ToString()}},`
       @{n="Subject";e={$currentSubject}},`
       @{n="Body";e={$currentBody}},`
       @{n="Received";e={$m.Received.ToString()}},`
       @{n="Status";e={$m.Status}}
    }
    $noRed = Get-ItemCount ($report | where {$_.status -like "red"})
    $noYellow = Get-ItemCount ($report | where {$_.status -like "yellow"})
    $noGreen = Get-ItemCount ($report | where {$_.status -like "green"})
    if ($noRed -gt 0) {$localEvent.status = "red"}
    ElseIf ($noYellow -gt 0) {$localEvent.status = "yellow"}
    Else {$localEvent.status = "green"}
    $localEvent.eventShort = "$noRed unread mail older than $redDays days, $noYellow unread mail older than $yellowDays days, $noGreen new mail"
    $localEvent.descriptionDetails = ($report | ConvertTo-Json)
    $localEvent.contentType = "json"
  }
  else 
  {
    $localEvent.descriptionDetails = "No unread mail in $mailbox"
    $localEvent.status = 'green'
    $localEvent.eventShort = "No unread mail in $mailbox"
  }
}
```
In the *ANALYZE* section of the script, on rows 4-32 I add the unread mail collected to a report PSCustomObject (so it's more easily converted to json). On rows 12-13 I cap the number of characters in the Subject to 50, and on rows 14-15 I cap the characters of the body to 200 in order to make the report more readable. 
On rows 9-11 the status (red/yellow/green) is set based on when the email was received and for how long it's been unread.

#### Result
Here's how the detector would look in the UI with a warning.

![Detector shared mailbox overview](/assets/images/detector-shared-mailbox-overview.png)

And if you click the headline you get the details of the unread mail.

![Detector shared mailbox details](/assets/images/detector-shared-mailbox-details.png)



*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*




