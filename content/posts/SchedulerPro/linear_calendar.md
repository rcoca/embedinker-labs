---
title: Two cases of previewing busyness. Linear Calendar
date: 2026-08-24
status: published
canonical_url: 
tags:
  - SchedulerPro
  - GoogleSheets
  - GoogleCalendar
  - AutomaticScheduling
distribution:
  linkedin:
    status: pending
    payload_snippet: This feature suggests building Linear Calendars to contain busyness by design.
    link_posted: ""
  reddit:
    status: pending
    target_subreddits:
      - cpp
      - systems
    link_posted: ""
  grimm_network:
    status: pending
    thread_id: ""
---

For good use and visualization of the timeline design - we're providing 2 small tools - one for the Project Tasks - and one for Proposed Schedule: Linear calendars disguised as gsheet formulas.

## Video Preview
{{< youtube id="VkH6lcPUBug" autoplay="false" >}}

The input data is the originar source of busyness - crowded calendar and too many commitments in too little time, no space to breathe. 


**Project Tasks - Roadmap Design**

1. Create a table for the Linear Calendar
2. Add dates ranges to track busyness for
3. Add formulas selecting projects for the heading dates range
```txt
=UNIQUE(FILTER($B:$B, ($D:$D <= MAX(K$2:X$2)) * (IF($E:$E="", $D:$D, $E:$E) >= MIN(K$2:X$2)) * ($B:$B <> "")))
```
4. Add occupancy formula - then drag it over the table region
```txt
=IF(SUMPRODUCT(($B:$B=$J3)*(IF($D:$D="", $E:$E, $D:$D)<=K$2)*(IF($E:$E="", $D:$D, $E:$E)>=K$2)*(($D:$D<>"")+($E:$E<>"")))>0, "█", "")
```
5. Repeat for the other dates ranges - updating formulas accordingly
```txt
=UNIQUE(FILTER($B:$B, ($D:$D <= MAX(K$12:X$12)) * (IF($E:$E="", $D:$D, $E:$E) >= MIN(K$12:X$12)) * ($B:$B <> "")))
=IF(SUMPRODUCT(($B:$B=$J13)*(IF($D:$D="", $E:$E, $D:$D)<=K$12)*(IF($E:$E="", $D:$D, $E:$E)>=K$12)*(($D:$D<>"")+($E:$E<>"")))>0, "█", "")

=UNIQUE(FILTER($B:$B, ($D:$D <= MAX(K$23:X$23)) * (IF($E:$E="", $D:$D, $E:$E) >= MIN(K$23:X$23)) * ($B:$B <> "")))
=IF(SUMPRODUCT(($B:$B=$J24)*(IF($D:$D="", $E:$E, $D:$D)<=K$23)*(IF($E:$E="", $D:$D, $E:$E)>=K$23)*(($D:$D<>"")+($E:$E<>"")))>0, "█", "")
```
## Video Preview
{{< youtube id="hBqKON6f2k8" autoplay="false" >}}
Linear Calendar from Proposed Schedule

As promissed - the busyness plot AKA Linear Calendar for the compute schedule.

1. create a new tab. add a table heading to hold the Linear Calendar.
2. add and squeeze date ranges. after all we want all to fit on a page - as overview
3. put formulas for projects spanning the date ranges
```txt
=UNIQUE(FILTER('Proposed Schedule'!$E:$E, ISNUMBER(MATCH('Proposed Schedule'!$A:$A, B$2:O$2, 0)), 'Proposed Schedule'!$E:$E <> ""))
=UNIQUE(FILTER('Proposed Schedule'!$E:$E, ISNUMBER(MATCH('Proposed Schedule'!$A:$A, B$12:O$12, 0)), 'Proposed Schedule'!$E:$E <> ""))
=UNIQUE(FILTER('Proposed Schedule'!$E:$E, ISNUMBER(MATCH('Proposed Schedule'!$A:$A, B$12:O$12, 0)), 'Proposed Schedule'!$E:$E <> ""))
```
4. put in the formulas that compute ccupancy
```txt
=IF(COUNTIFS('Proposed Schedule'!$E:$E, $A3, 'Proposed Schedule'!$A:$A, B$2) > 0, "█", "")
=IF(COUNTIFS('Proposed Schedule'!$E:$E, $A13, 'Proposed Schedule'!$A:$A, B$12) > 0, "█", "")
=IF(COUNTIFS('Proposed Schedule'!$E:$E, $A22, 'Proposed Schedule'!$A:$A, B$21) > 0, "█", "")
```
5. can add conditional formattign for non-empty cells - or leave them
