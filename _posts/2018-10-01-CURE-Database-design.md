---
layout: single
title: "CURE - Database design"
excerpt: "CURE - Database design. The very simplistic database design we chose for our homebrewed monitoring solution."
date: 2018-10-01
comments: true
permalink: /CURE-Database-design.html
tags:
  - cure
  - design
  - monitoring
  - postgres
  - linux
category:
  - cure
---
Previous posts in this series:
- [CURE-Design](/CURE-Design.html)
- [CURE-Environment](/CURE-Environment.html)

## The design
As described in previous posts, we wanted to have stateless detectors that would bake source event data and then shove the analyzed event into a database. The UI would then display that event data in an ordered fashion by making request to the API exposing the detector events in the database. We had decided to use PostgreSQL on linux, mainly because it's OSS and run on Linux, but also because of its modern approach to datatypes (for example native json), which suited our needs.
After some debate back and forth we decided to have these three tables in the database.

#### inventoryTable
We needed a table to keep the settings for the detectors. In this table both instructions for the individual detector is fetched on run, but also other instructions for the UI on how to display the data for each detector.
A quick run-through of what data is to be held there:
- **detectorId:** The unique ID of each detector. Always useful to have.
- **detectorName:** The human friendly name of the detector
- **refreshRate:** The plan is to make different divs in the UI, and this would be the setting for how often to refresh each div. We imagine some detectors needing to update rarely and some more often, depending on the criticality of the system monitored.
- **detectorEnvironment:** Basically information on *where* the detector is running. In our current setup it would just be one single VM for all detectors, but that can change in the future.
- **heartbeats:** This time stamp is set by the detector on each run, in order for the UI to be able to alert if a detector has died.
- **snoozeTime:** This is the amount of time to snooze a detector. We can snooze a detector (make it green and suppress it from running for this amount of time) through a *snooze button* in the UI. 
- **area:** This is used to describe which area of responsibility within the IT department the detector may concern. For example, support tickets would be *helpdesk* area.
- **isActive:** This is wether the detector is active and should run, and also displayed in the UI, or not.

Below is the query to set up the table.
```sql
CREATE TABLE inventoryTable (
  detectorId SERIAL PRIMARY KEY,
  detectorName VARCHAR(50) UNIQUE,
  refreshRate INT,
  detectorEnvironment TEXT,
  heartBeatTS TIMESTAMP,
  heartBeatTimeOut INT,
  snoozeTime INT,
  area VARCHAR(50),
  isActive BOOLEAN
);
```
#### eventTable
Of course, we need a table to store the actual event data produced by the detectors. Below is an explanation of the types of data we store in the event table. We had a long debate on weather to have individual tables for each detector. Finally, I caved and agreed on putting all detector event data into one single table.
- **eventId:** Unique ID for each event posted
- **detectorId:** The ID of the detector posting the event
- **dateTime:** When the event was posted
- **status:** Color of the event (green/yellow/red or grey)
- **eventShort:** Short summery of the event, for example "3 alerts detected" or something like that.
- **snoozeTS:** When the event was snoozed in the UI
- **snoozedBy:** The person who pushed the snooze button.

This is how one single detector looks in the UI
![Detector](/assets/images/detector.png)

And the query to set up the table.
```sql
CREATE TABLE eventTable (
  eventID SERIAL UNIQUE,
  detectorId INT REFERENCES inventoryTable(detectorId) ON DELETE RESTRICT,
  dateTime TIMESTAMP,
  status VARCHAR(50),
  eventShort VARCHAR(512),
  snoozeTS TIMESTAMP,
  snoozedBy VARCHAR(50),
  PRIMARY KEY (eventID,detectorId)
);
```
#### eventDescriptionTable
Since each detector event is based on x rows of source data, we also wanted a way to look at the actual source events. This is done by clicking the headline of a detector in the UI and there you'll see the current source events that has been analyzed by the detector. A description of the data held in the table:
- **eventDescriptionId:** Unique ID for each description event posted
- **eventId:** The ID of the event to which the description belongs
- **contentType:** What data type the content of the message is, for example "string" or "json".
- ** descriptionDetails:** The actual details of the event.

This is how the details of a detector event would look in the UI, when you've clicked the headline of a detector in the landing page.
![Event description](/assets/images/event-description.png)

And the query to set it up.
```sql
CREATE TABLE eventDescriptionTable (
  eventDescriptionId SERIAL UNIQUE,
  eventId INT REFERENCES eventTable(eventId) ON DELETE RESTRICT,
  contentType VARCHAR(50),
  descriptionDetails TEXT,
  PRIMARY KEY (eventDescriptionID,eventID)
);```

In future posts I will describe more details on the setup of the detector environment, UI and API.

*The personal experiences, viewpoints and opinions expressed in this blog post are my own and in no way represent those of the company.*

