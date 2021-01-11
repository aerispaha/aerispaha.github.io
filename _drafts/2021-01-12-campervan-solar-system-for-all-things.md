---
layout: post
title: How we built a camper van solar system to power all our things
tags: Tech
---

![van stuff]({{site.baseurl}}/assets/img/van-smokey-mountains.jpg)

In the earliest weeks of 2020, my wife and I (along with our cat) decided to
leave our beloved studio apartment in Philadelphia for a life on the road.
The plan was to travel to the Texas-Mexico border to work within refugee camp.

The cat insisted on driving instead of flying which led us to the decision to
pack ourselves into a campervan. And after
several weeks of combing Craigslist, we found a humble, but very well-preserved,
1985 Dodge Roadtrek campervan.


#### Overview
- design process/decisions
  - design criteria:
    - energy demand (peak load, daily consumption)
    - battery supply for a 2 days
    - recharge duration
  - battery selection
    - huge decision that impacted a lot of other components
    - connecting in series, 24V inverter
  - panel selection  
- installation process
  - hacking the tesla module, finding the balance leads
  - panel installation: no new holes drilled in roof. VHB tape
  - splicing into 12V system: massive frustration with not understanding how
    cars use the chassis as a negative wire.
  - drilling into the water main
- Performance after installation
  - running AC in heat emergencies for cat
  - 40 amp breaker not sufficient, special order of 60 amp DC breaker shipped to
    flagstaff AZ orielly auto
