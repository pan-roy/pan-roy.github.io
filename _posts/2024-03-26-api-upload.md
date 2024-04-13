---
title: 4. Automated Manual Journalling
date: 2024-03-26 21:26:01 +1100
categories: [Showcase, Data Retrieval and Upload]
tags: [data upload, xero]     # TAG names should always be lowercase
---

## How this impacts you

Instead of unidirectional data retrieval, what if we aimed to automate the process of uploading financial data?

Xero does support CSV template upload for manual journals. However, what if we could skip the manual import process entirely?

![manual journal](assets/manual_journals/manual_journals1.png)
*"Manual" manual journalling*

![import csv](assets/manual_journals/manual_journals2.png)
*CSV import*

Through utilizing POST Manual Journals API calls, we have the capability to automate the posting of manual journals directly within Excel. This eliminates the need for accountants to log into their accounting system and manually navigate to the journal entry interface, a process susceptible to human error and time-consuming, particularly when multiple journal entries need uploading simultaneously.

## Goal

Our current objective is to establish an automation pipeline that empowers end-users to transmit journal entry details directly from Excel to Xero, eliminating the requirement to exit Excel or engage with the accounting system, specifically Xero, during this process.

![frontend](assets/manual_journals/manual_journals3.png)
*Simplified frontend example*

## Steps

**Requirements**: local Python interpreter, Excel, basic Python knowledge.

To accomplish our goal, we would need to execute several steps:

* Install and configure the xlwings library in Python, along with installing the xlwings Excel add-in.

* Create a Python script to establish a connection with Xero.

* Create a Python function to convert tabular data into JSON format.

* Create a VBA macro to trigger the Python scripts within Excel.

* Configure Power Query to refresh the journal entry table, if journal requires preparation prior to submission.

**Step 1: Install and configure the xlwings library in Python, along with installing the xlwings Excel add-in.**

Excel does not have native functionality in regard to running Python scripts. We therefore require an external Python library, xlwings, to facilitate the connection between Excel and Python.

To install xlwings, activate your virtual environment (refer prior tutorials{insert link here}), and then enter ```pip install xlwings``` in a PowerShell terminal. Should you encounter any additional missing packages during the following process, use the ```pip install``` command to install them. Enter ```pip list``` to check whether you have successfully installed xlwings.

For detailed documentation on xlwings, please visit [here](https://docs.xlwings.org/en/latest/).

We will now install the xlwings Excel add-in. Close all open Excel instances (if any) and enter ```xlwings addin install``` in your PowerShell terminal. Once complete, check whether the installation has been successful by opening a new Excel instance. You should see a new add-in at the end of your top ribbon. This will grant Excel the ability to trigger Python scripts.

![add-in](assets/manual_journals/manual_journals4.png)

**Step 2: Create a Python script to establish a connection with Xero.**

Similar to our prior procedures relating to trial balances and transaction listings, we can modify our pre-existing Python script's variables to post manual journals instead. Refer the script below:

```python
import requests
import webbrowser
import base64
import xlwings as xw
import pandas as pd

client_id = '<INSERT_ID_HERE>'
client_secret = '<INSERT_SECRET_HERE>'
redirect_url = 'https://xero.com/'
scope = 'offline_access accounting.transactions'
b64_id_secret = base64.b64encode(bytes(client_id + ':' + client_secret, 'utf-8')).decode('utf-8')


def XeroFirstAuth():
    # 1. Send a user to authorize your app
    auth_url = ('''https://login.xero.com/identity/connect/authorize?''' +
                '''response_type=code''' +
                '''&client_id=''' + client_id +
                '''&redirect_uri=''' + redirect_url +
                '''&scope=''' + scope +
                '''&state=123''')
    webbrowser.open_new(auth_url)
    
    # 2. Users are redirected back to you with a code
    auth_res_url = input('What is the response URL? ')
    start_number = auth_res_url.find('code=') + len('code=')
    end_number = auth_res_url.find('&scope')
    auth_code = auth_res_url[start_number:end_number]
    print(auth_code)
    print('\n')
    
    # 3. Exchange the code
    exchange_code_url = 'https://identity.xero.com/connect/token'
    response = requests.post(exchange_code_url, 
                            headers = {
                                'Authorization': 'Basic ' + b64_id_secret
                            },
                            data = {
                                'grant_type': 'authorization_code',
                                'code': auth_code,
                                'redirect_uri': redirect_url
                            })
    json_response = response.json()
    print(json_response)
    print('\n')
    
    # 4. Receive your tokens
    return [json_response['access_token'], json_response['refresh_token']]


# 5. Check the full set of tenants you've been authorized to access
def XeroTenants(access_token):
    connections_url = 'https://api.xero.com/connections'
    response = requests.get(connections_url,
                            headers={
                                'Authorization': 'Bearer ' + access_token,
                                'Content-Type': 'application/json'
                            })
    json_response = response.json()
    print(json_response)

    for tenants in json_response:
        json_dict = tenants
    return json_dict['tenantId']


# 6.1 Refreshing access tokens
def XeroRefreshToken(refresh_token):
    token_refresh_url = 'https://identity.xero.com/connect/token'
    response = requests.post(token_refresh_url,
                             headers={
                                 'Authorization': 'Basic ' + b64_id_secret,
                                 'Content-Type': 'application/x-www-form-urlencoded'
                             },
                             data={
                                 'grant_type': 'refresh_token',
                                 'refresh_token': refresh_token
                             })
    json_response = response.json()
    print(json_response)

    new_refresh_token = json_response['refresh_token']
    rt_file = open('C:/Users/roypa/Downloads/refresh_token.txt', 'w')
    rt_file.write(new_refresh_token)
    rt_file.close()

    return [json_response['access_token'], json_response['refresh_token']]


def convert_json():
    # Open the Excel workbook
    wb = xw.books.active

    # Get the Excel table
    table = wb.sheets[0].api.ListObjects("input")

    # Get the table data
    data = table.Range.Value

    # Extract the Narration from the first cell
    narration = data[1][0]  # Assuming the first row is headers and the narration is in the first column

    # Separate the headers and the data
    headers = data[0]
    data_rows = data[1:]

    # Convert the data to a pandas DataFrame
    df = pd.DataFrame(data_rows, columns=headers)

    # Convert DataFrame to desired JSON format
    json_data = {
        "Narration": narration,
        "JournalLines": df[["LineAmount", "AccountCode"]].to_dict(orient="records")
    }

    return json_data


# 6.2 Call the API
def import_journals(json_data1):
    old_refresh_token = open('C:/Users/roypa/Downloads/refresh_token.txt', 'r').read()
    new_tokens = XeroRefreshToken(old_refresh_token)
    xero_tenant_id = XeroTenants(new_tokens[0])

    get_url = 'https://api.xero.com/api.xro/2.0/ManualJournals'
    response = requests.post(get_url,
                            headers={
                                'Authorization': 'Bearer ' + new_tokens[0],
                                'Xero-tenant-id': xero_tenant_id,
                                'Accept': 'application/json'
                            },
                            json=json_data1)
    # Check if the request was successful
    if response.status_code == 200:
        return "Data successfully imported to Xero."
    else:
        return "Error importing data to Xero: " + response.text

    # xero_output = open('C:/Users/roypa/Downloads/xero_output.json', 'w')
    # xero_output.write(response.text)
    # xero_output.close()


# old_tokens = XeroFirstAuth()
# XeroRefreshToken(old_tokens[1])


# Call the convert_json function to get the JSON data
json_data3 = convert_json()
print(json_data3)
result = import_journals(json_data3)
print(result)
```

**Step 3: Create a Python function to convert tabular data into JSON format.**

Please ensure you have installed the pandas library by running pip install pandas in PowerShell.

The ```convert_json()``` function, as described above, utilizes the pandas library to convert tabular Excel data into dataframes. This data can then be manipulated for further processing and upload.

Assuming there is a defined table named "input" in the first sheet of our Excel workpaper, the function extracts the journal entries' relevant information such as account code, journal amount, and description. This data is formatted to meet Xero's manual journal requirements, where the journal is posted in the current open period. The function extracts the description from the first row (excluding headers), and adds the journal amount and account code.

The resulting dataframe is converted into a dictionary format using the ```to_dict()``` method, resembling the structure of JSON files. Finally, this converted data can be passed to the ```import_journals()``` function for upload. Refer to the script for further details.