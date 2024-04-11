---
title: 2. JSON to CSV Conversion
date: 2024-03-28 21:26:01 +1100
categories: [Showcase, Data Retrieval]
tags: [data retrieval, json]     # TAG names should always be lowercase
---

## Background

In the prior walkthrough {TODO: insert link here}, we have extracted data from Xero in the form of JSON files. However, JSON files utilize a nested dictionary data structure, where information is kept in key value pairs. Values can also further extend into new key value pairs. This inevitably renders it difficult for accountants to use such data, as we are more familiar with tabular data (e.g. Excel).

Looking at the top portion of the Xero trial balance obtained previously, you will notice that data is stored via nested dictionaries. We will need to normalize or flatten the JSON data for it to be of use.
```json
{

  "Id": "1c8cb66e-c7db-4bf4-9620-27926461ea5a",

  "Status": "OK",

  "ProviderName": "myApp",

  "DateTimeUTC": "\/Date(1712829803118)\/",

  "Reports": [

    {

      "ReportID": "TrialBalance",

      "ReportName": "Trial Balance",

      "ReportType": "TrialBalance",

      "ReportTitles": [

        "Trial Balance",

        "Demo Company (AU)",

        "As at 31 January 2024"

      ],

      "ReportDate": "11 April 2024",

      "UpdatedDateUTC": "\/Date(1712829803118)\/",

      "Fields": [],

      "Rows": [

        {

          "RowType": "Header",

          "Cells": [

            {

              "Value": "Account"

            },

            {

              "Value": "Debit"

            },

            {

              "Value": "Credit"

            },

            {

              "Value": "YTD Debit"

            },

            {

              "Value": "YTD Credit"

            }

          ]

        },

        {

          "RowType": "Section",

          "Title": "Revenue",

          "Rows": [

            {

              "RowType": "Row",

              "Cells": [

                {

                  "Value": "Sales (200)",

                  "Attributes": [

                    {

                      "Value": "e2bacdc6-2006-43c2-a5da-3c0e5f43b452",

                      "Id": "account"

                    }

                  ]

                },

                {

                  "Value": "",

                  "Attributes": [

                    {

                      "Value": "e2bacdc6-2006-43c2-a5da-3c0e5f43b452",

                      "Id": "account"

                    }

                  ]

                },

                {

                  "Value": "18286.00",

                  "Attributes": [

                    {

                      "Value": "e2bacdc6-2006-43c2-a5da-3c0e5f43b452",

                      "Id": "account"

                    }

                  ]

                },

                {

                  "Value": "",

                  "Attributes": [

                    {

                      "Value": "e2bacdc6-2006-43c2-a5da-3c0e5f43b452",

                      "Id": "account"

                    }

                  ]

                },

                {

                  "Value": "22636.00",

                  "Attributes": [

                    {

                      "Value": "e2bacdc6-2006-43c2-a5da-3c0e5f43b452",

                      "Id": "account"

                    }

                  ]

                }

              ]

            }

          ]

        }
```

## Goal

Our current goal is to flatten the JSON data above into a tabular format that can be consumed for further analysis. The transformed data should more or less resemble the following table.

![Sample data](assets/data_retrieval/data_retrieval2.png)
*Simplified data example*

## Steps

**Requirements**: local Python interpreter, basic Python knowledge.

JSON files require examination on a case-by-case basis, though the overall method will be the same.

For this particular example, we will have to analyze the data structure level by level to determine what we want.

We are interested in 1st level ```"Reports"```, 2nd level  ```"Rows"```, 3rd level ```"Rows"```, 4th level ```"Cells"```, 5th level ```"Value"```. If you follow this path, you can see that the first value would be ```"Sales (200)"```, which is the first general ledger account. You can then see that **for each item** under ```"Cells"``` at the 4th level, the data under concern is always stored in ```"Value"```.

* Therefore, the solution would be to loop through each item at the 4th level ```"Cells"``` and extract all ```"Value"``` values.

* However, note that at the 3rd level ```"Rows"```, the first item where ```"RowType": "Header"``` is does not contain a ```"Row"``` key at he 4th level - it does contain our headers though we will ignore them in this demonstration. As a result, we will need to introduce an ```IF``` logic check to exclude this header section.

* We can then store all values within a Python list via an ```append()``` method, and write all rows to a blank CSV file.

* Please change the file paths below into the applicable ones on your end.

```python
import json
import csv

# Load the JSON data
with open('C:/Users/roypa/Downloads/xero_output.json') as f:
    data = json.load(f)

# Initialize an empty list to store all rows
all_rows = []

# Iterate over each report
for report in data['Reports']:
    # Iterate over each section in the report
    for section in report['Rows']:
        # Check if the section has rows
        if 'Rows' in section:
            # Iterate over each row in the section
            for row in section['Rows']:
                # Append the row to the list
                all_rows.append(row)

# Convert the rows to CSV format
if all_rows:
    with open('C:/Users/roypa/Downloads/xero_output.csv', 'w', newline='') as csvfile:
        writer = csv.writer(csvfile)
        for row in all_rows:
            if 'Cells' in row:
                writer.writerow([cell.get('Value', '') for cell in row['Cells']])

```

If done correctly, you should be able to generate a CSV file displaying the following:

![output](assets/json_to_csv/json_csv1.png)

If we compare the CSV output directly with Xero's Trial Balance function, you will see they match without issues.

![comparison](assets/json_to_csv/json_csv2.png)

## Outcome

Our transformed tabular data can then either be configured to flow directly into other Excel workpapers as data inputs or into a corporate network drive as backup. There are many options that branch out from here, from automated reporting snapshots to self-refreshing Tableau dashboards or centralized data repositories etc - not to mention that some can bypass the need for CSV conversion entirely and directly use JSON data.

Our next example will put what we have we have learned in a different scenario, where we extract transaction / journal listings instead {TODO: insert link here}.
