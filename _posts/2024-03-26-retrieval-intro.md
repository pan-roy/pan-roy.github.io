---
title: Data Retrieval via Xero API
date: 2024-03-27 21:26:01 +1100
categories: [Showcase, Data Retrieval and Upload]
tags: [data retrieval, introduction]     # TAG names should always be lowercase
---

## Background:

Accountants usually perform manual data retrieval for further processing. Data, whether it be in the form of various transaction listings or financial statements, is used as input for further calculations and subsequently presented as independent reports or uploaded back into the accounting system (e.g. journal entries). Should there be numerous systems and sources involved, accountants would need to dedicate time towards manual data extraction in the absence of custom integrations or interconnected ERP systems.

Automated reporting and analytics require input data to be available in the first place - models will be rendered useless if no data was fed in. Therefore, the first step towards our analytics setup would be automated data extraction.

![Sample report](assets/data_retrieval1.png)
*Simplified report example*

![Sample data](assets/data_retrieval2.png)
*Simplified data example*

## Goal:

Our current goal is to create a Python script that automatically extracts financial data from an accounting system and setup the relevant initial authentication procedures required.

Xero, one of the more popular cloud-based accounting software solutions, will be used this demonstration given its availability of API documentation. Note that with the proper API access setup, logic utilized here would be applicable to other accounting environments, as long as documentation and support from IT is present.

Visit the [_Xero Main_](https://www.xero.com/au/) and [_Xero API_](https://developer.xero.com/documentation/api/accounting/overview) pages here.

![Sample Xero](assets/data_retrieval3.png)
*Xero sample*

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
