---
title: Data Retrival via Xero API
date: 2024-03-27 21:26:01 +1100
categories: [Showcase, Data Retrieval and Upload]
tags: [data retrieval, introduction]     # TAG names should always be lowercase
---

## Background:  

Accountants usually manually retrieve data for further processing. Data, whether it be in the form of transaction listings, journal listings, financial statements or trial balances etc., is subsequently used as input for further calculations and then presented either as independent reports or uploaded back into the accounting system. First step towards optimization of existing procedures is automated data extraction.  

{insert accounting examples – one from Eightcap, one from RSM}  

## Goal:  

Current goal: to create script that automatically extracts financial data under concern from accounting system and setup initial authentication procedures to facilitate this.
Xero, one of the more popular cloud-based accounting software options, will be used for demonstration purposes given availability of detailed documentation regarding accounting APIs. Note that given proper API access, along with the proper authentication setup, the logic utilized here can be replicated across different accounting environments.  

{insert Xero APIs webpage and link}  

## Actions:  

Assumptions: existing Xero account, local Python interpreter  

* Step 1: Setup OAuth between computer and Xero for unhindered access.  

{insert link to OAuth tutorial, sample scripts, process screenshots}  

* Step 2: Define API scope depending on data to retrieve. E.g. journals listing, trial balance, PnL.  

{insert Xero interface samples}  

* Step 3: Define parameters. E.g. dates, accounts, batches.  

{insert Xero interface samples}  

* Step 4: Manually run script on demand, or schedule via Windows Task Scheduler.  

{demonstrate Task Scheduler interface setup}  

## Outcome:  

Automated JSON data inflows to be used as entry point for further optimization/automation.  

{insert screenshots of JSON output}
