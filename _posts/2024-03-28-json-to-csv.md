---
title: 2. JSON to CSV Conversion
date: 2024-03-28 21:26:01 +1100
categories: [Showcase, Data Retrieval and Upload]
tags: [data retrieval, json]     # TAG names should always be lowercase
---

## How this impacts you

In our prior [walkthrough](https://www.roypan.cc/posts/trial-balance/), data extraction from Xero was performed resulting in JSON format files. JSON files employ a hierarchical structure consisting of nested dictionaries, organizing data into key-value pairs, with potential expansion into additional key-value pairs. This inherent structure presents challenges for accountants accustomed to tabular formats such as Excel.

Upon examination of the top segments of our Xero trial balance, it becomes evident that data is organized within nested dictionaries. To facilitate usability, it is imperative to normalize or flatten the JSON data structure.
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

Our objective is to transform the JSON data into a tabular format suitable for subsequent analysis. The desired outcome is to present the data in a structured table similar to the following:

![Sample data](assets/data_retrieval/data_retrieval2.png)
*Simplified data example*

## Steps

**Requirements**: Python

JSON files require examination on a case-by-case basis, though the overall method will be the same.

For this particular example, a methodical approach is necessary to flatten the data into a tabular format. Let's go through this step-by-step:

We will navigate to 1st level ```"Reports"```, 2nd level  ```"Rows"```, 3rd level ```"Rows"```, 4th level ```"Cells"```, 5th level ```"Value"```. Should you follow this path, you can see that the first value would be ```"Sales (200)"```, which is the first general ledger account. You can then see that **for each item** under ```"Cells"``` at the 4th level, the data under concern is always stored in ```"Value"```.

* Therefore, the solution would be to loop through each item at the 4th level ```"Cells"``` and extract all ```"Value"``` values.

* However, note that at the 3rd level ```"Rows"```, the first item where ```"RowType": "Header"``` lies does not contain a ```"Rows"``` key at the 4th level - it does contain our headers though we will ignore them in this demonstration. As a result, we will need to introduce an ```IF``` logic check to exclude this header section.

* We can then store all values within a Python list via an ```append()``` method, and write all rows to a blank CSV file.

* Please replace the file paths below with the applicable ones on your system.

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

If we compare the CSV output directly with Xero's Trial Balance function, you will see that they align seamlessly.

![comparison](assets/json_to_csv/json_csv2.png)

## Outcome

The transformed tabular data offers versatile integration options, such as direct incorporation into Excel workpapers or backup storage in a corporate network drive. From this point, various possibilities emerge, including automated reporting snapshots, self-refreshing Tableau dashboards, and centralized data repositories. 

In our next example, we will apply the acquired knowledge to a different scenario, focusing on extracting [transaction listings](https://www.roypan.cc/posts/transactions-api/).
