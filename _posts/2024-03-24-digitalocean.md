---
title: 6. Virtual Private Server Setup
date: 2024-03-25 21:26:01 +1100
categories: [Showcase, Virtual Private Servers]
tags: [vps, scripting]     # TAG names should always be lowercase
---

## How this impacts you

In our previous walkthroughs, we've focused on running scripts locally. However, what if we wanted to schedule scripts to run overnight, ensuring reports and data processing are complete first thing in the morning? What if we wanted to have a centralized repository of our scripts to avoid version control issues where different end-users have different script versions?

Our current solution of scheduling scripts via Windows Task Scheduler requires a dedicated, powered-on computer throughout the process. This setup poses challenges, as typical Windows consumer installations are often not configured for external access, complicating maintenance efforts. Moreover, concerns regarding power usage and uptime reliability arise, especially with increasing reporting requirements as businesses grow.

That is why in this walkthrough, we will be exploring virtual private servers.

Currently there are many solutions available on the market, from AWS EC2 to Azure Virtual Machines etc. However, these solutions are expensive and difficult to maintain, especially from our perspective as non-technical hobbyists. As a result, we will opt for a more developer-friendly solution, DigitalOcean, to setup a simple Linux virtual private server for overnight scripting.

It's important to note that in a professional enterprise environment, dedicated IT personnel are responsible for providing tailored solutions. However, the tutorials we've covered so far are primarily aimed at an audience with a blend of IT & accounting skills, focusing on proof of concept rather than large-scale, corporate-wide implementation.

Nevertheless, the principles and knowledge here are highly valuable for optimizing accounting processes and designing prototypes. These resources can also assist IT personnel in constructing more robust solutions tailored to specific corporate needs. In other words, our goal is to effectively bridge the gap between these two worlds of IT and accounting, to facilitate collaboration and to enable the development of solutions that meet the needs of both domains.

## Goal

