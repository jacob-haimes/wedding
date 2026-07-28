---
title: 'Schedule'
date: 2026-07-18
type: landing

design:
  # Default section spacing
  spacing: "6rem"

sections:
  - block: markdown
    id: schedule
    content:
      title: Schedule
      text: |
        **Saturday, September 18**
        {style="padding-top: 2rem"}
        {{< table path="schedule_saturday.csv" header="true" >}}
        
        **Sunday, September 19**
        {style="padding-top: 2rem"}

        {{< table path="schedule_sunday.csv" header="true" >}}
    design:
      no_padding: true
      spacing:
        padding: [4rem, 0, 2rem, 0]
        margin: [0, 0, 0, 0]
---
