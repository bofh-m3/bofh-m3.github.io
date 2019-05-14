---
layout: single
title: "CURE - Detector Printers"
excerpt: "CURE - The homebrewed monitoring solution.  In this post I'll describe the steps for setting up a detector monitoring the company printers."
date: 2018-12-02
comments: true
permalink: /Detector-Printers.html
tags:
  - cure
  - powershell
  - printer
  - ricoh
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
The Stockholm branch office has a handful of printers connected to a single print server. The printers are spread across the office space on different floors and different parts of the premises. We needed a way to get an indication of the current toner status, to be prepared and make sure replacements were available *before* we ran out.
The printers are RICO NRG in different models (I, like most IT pros, pride myself on knowing very little about printers. I fucking hate printers, so I cannot, and I will not give more details about the machines in question).

I will describe the steps I did for setting up a CURE detector for printers based on the *DetectorTeamplate.ps1* described in [CURE-Detector foundation](/CURE-Detector-foundation.html) and the instructions in [CURE-Detector-step-by-step](/CURE-Detector-step-by-step.html).
The whole detector script can be found [here](https://github.com/bofh-m3/CURE/blob/master/Detectors/Printers.ps1)

#### Approach
Micke, my colleague had written script to parse out the current toner level readings from the printers' web interface (as per usual no API or CLI is available, that's kind of default for office machinery...), and together we made a CURE detector out of it. 
Luckily all models we have has the same crappy web interface, so we could re-use the parsing code at least. 
Here's an image of the stuff we wanted to retrieve from the printer web ui.
![detector printer web ui](/assets/images/detector-printer-web-ui.png)

#### LOCAL SETTINGS, FUNCTIONS AND DEPENDENCIES
```powershell
$detectorName = "Printers"
$printers = @{
  "Printer1.fqdn" = "MP C2003 - RICOH";
  "Printer2.fqdn" = "MP C2003Z - RICOH";
  "Printer3.fqdn" = "MP C2003 - RICOH";
  "Printer4.fqdn" = "MP C3003 - RICOH";
  "Printer5.fqdn" = "MP C306Z - RICOH"
}
$colors = @("black","cyan","magenta","yellow")
$remainingRed = 2
$remainingYellow = 15
```
In the *Local Settings* section, we defined the thresholds for a yellow/red warning, and also made a hash table to map the printer fqdn to a model so we easily could what type of toner to order.

#### CONNECT AND COLLECT
```powershell
<# this is custom #>
$report = @()
foreach ($printer in $printers.keys)
{
  $HTML=$null
  $result = "" | select @{n="printer";e={$printer}},`
   @{n="status";e={"grey"}},`
   @{n="model";e={$($printers.$printer)}},`
   @{n="black";e={$null}},`
   @{n="cyan";e={$null}},`
   @{n="magenta";e={$null}},`
   @{n="yellow";e={$null}},`
   @{n="message";e={$null}}
  $uri = "http://$($printer)/web/guest/sv/websys/webArch/getStatus.cgi#linkToner"
  try {$HTML = Invoke-WebRequest $uri -EA stop}
  catch 
  {
    $result.message = $_.ToString()
    $result.status = "red"
  }
  if ($HTML)
  {
    $tonerHTML = $HTML.ParsedHtml.body.getElementsByTagName('div') | Where {$_.classname -eq 'tonerArea'} 
    $n = 0
    foreach ($i in $tonerHTML.innerHTML)
    {
      if ($n -eq 0) {[int]$result.black = ($i -split 'width=') -split ' height=' | select -Index 1}
      elseif ($n -eq 1) {[int]$result.cyan = ($i -split 'width=') -split ' height=' | select -Index 1}
      elseif ($n -eq 2) {[int]$result.magenta = ($i -split 'width=') -split ' height=' | select -Index 1}
      elseif ($n -eq 3) {[int]$result.yellow = ($i -split 'width=') -split ' height=' | select -Index 1}
      else {$result.message = "too many colors"}
     $n++
    }
  }
  $report += $result
}  
```
In the *CONNECT AND COLLECT* section of the script we loop through the printers and collect html from the printers' web interface. If a printer is unresponsive the error and status will be written to the report. On rows 21-34 the HTML is parsed into toner readings for each color. It aint pretty but what to do when you only get html to work with.

#### ANALYZE
```powershell
<# this is custom #>
foreach ($r in $report)
{
  $remaining = $colors | %{$r.$_}
  if ($remaining | where {$_ -lt $remainingRed}) {$r.status = "red"}
  elseif ($remaining | where {$_ -lt $remainingYellow}) {$r.status = "yellow"}
  else {$r.status = "green"}
}
$noRed = Get-ItemCount ($report | where {$_.status -like "red"})
$noYellow = Get-ItemCount ($report | where {($_.status -like "yellow")})
$noGreen = Get-ItemCount ($report | where {$_.status -like "green"})
if ($noRed -gt 0) {$localEvent.status = "red"}
elseif ($noYellow -gt 0) {$localEvent.status = "yellow"}
else {$localEvent.status = "green"}
$localEvent.eventShort = "$noRed replace now, $noYellow replace soon, $noGreen OK"
$localEvent.descriptionDetails = ($report | sort status -desc | ConvertTo-Json)
$localEvent.contentType = "json"
```
In the *ANALYZE* section of the script, we loop trough the report to validate if any printer is low on any particular toner color. Status is set per printer, and the *localEvent* object is prepared, ready to be shoved into the database.

#### Result
Here's how the detector would look in the UI with a couple of printers low on toner.
![Detector printers overview](/assets/images/detector-printers-overview.png)
And if you click the headline you get the details of the unread mail.
![Detector printers details](/assets/images/detector-printers-details.png)



*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*
