---
title: Data Retrieval via Xero API
date: 2024-03-27 21:26:01 +1100
categories: [Showcase, Data Retrieval and Upload]
tags: [data retrieval, introduction]     # TAG names should always be lowercase
---

## Background:

Accountants usually perform manual data retrieval for further processing. Data, whether it be in the form of various transaction listings or financial statements, is used as input for further calculations and subsequently presented as independent reports or uploaded back into the accounting system (e.g. journal entries). Should there be numerous systems and sources involved, accountants would need to dedicate time towards manual data extraction in the absence of custom integrations or interconnected ERP systems.

For instance, what if we had a report that took in data from an Excel table. How could we enable the table to pull data directly from an accounting system?

![Sample report](assets/data_retrieval1.png)
*Simplified report example*

![Sample data](assets/data_retrieval2.png)
*Simplified data example*

Automated reporting and analytics require input data to be available in the first place - models will be rendered useless if no data was fed in. Therefore, the first step towards our analytics setup would be automated data extraction. System API calls usually result in JSON data returned, which would be our starting point.

![Sample JSON](assets/data_retrieval4.png)
*JSON example*

## Goal:

Our current goal is to create a Python script that automatically extracts financial data from an accounting system and setup the relevant initial authentication procedures required.

Xero, one of the more popular cloud-based accounting software solutions, will be used this demonstration given its availability of API documentation. Note that with the proper API access setup, logic utilized here would be applicable to other accounting environments, as long as documentation and support from IT is present.

Visit the [_Xero Main_](https://www.xero.com/au/) and [_Xero API_](https://developer.xero.com/documentation/api/accounting/overview) pages here.

![Sample Xero](assets/data_retrieval3.png)
*Xero sample company*

## Actions:  

**Requirements**: existing Xero account, local Python interpreter, Windows, basic Python knowledge (though explanations will be provided).

### **Step 1: Setup OAuth2 between computer and Xero for unhindered access**

Assuming that you have registered a Xero account with either the default demo company or an actual company linked, we will need to establish a long-term connection between our local computer and Xero.

Click [here](https://www.youtube.com/watch?v=t0DgAMgN8VY&list=WL&index=3) should you be interested in a detailed video tutorial (courtesy of [Edgecate](https://www.youtube.com/@edgecate)).

Assuming Python has already been installed, we will need to install the relevant package dependencies required for our script to run. It is recommended that you create a virtual environment for easier dependency management.

To create a virtual environment, open a PowerShell terminal and navigate to the environment you wish to setup your scripts via the ```cd``` command. Example:

```PowerShell
cd C:\Users\roypa\Scripts
```

Use the following command to create a virtual environment. This will prevent conflicts should you choose to work on other Python projects. Do not include the placeholder angle brackets below:

```PowerShell
python -m venv <INSERT_NAME>
```

You can then activate the virtual environment via the ```activate``` file within the newly created environment folder. Replace the example path below with yours and press Enter:

```PowerShell
C:\Users\roypa\Scripts\reportenv\Scripts\activate
```

Assuming ```pip``` is installed by default along with Python, enter the following command to install required dependencies. Should you get any errors regarding missing dependencies while scripting later on, you can use ```pip``` to install them as well.

```PowerShell
pip install requests
```

Use ```pip list``` to check whether installation is successful. Sample output on my end, note that yours will have much less content:

```
(reportenv) PS C:\Users\roypa\Scripts> pip list
Package                   Version
------------------------- -----------
pandas                    2.2.1
pefile                    2023.2.7
pip                       24.0
proto-plus                1.23.0
protobuf                  4.25.3
psycopg2                  2.9.9
pyasn1                    0.6.0
pyasn1_modules            0.4.0
pywin32                   306
pywin32-ctypes            0.2.2
requests                  2.31.0
requests-oauthlib         2.0.0
rsa                       4.9
setuptools                69.2.0
uritemplate               4.1.1
urllib3                   2.2.1
uvicorn                   0.29.0
websocket-client          1.7.0
xlwings                   0.31.1
```

The following Python script will assist us in establishing a secure long-term connection without the need of frequent logins. However, prep work is required before the script can be used.

```Python
import requests
import webbrowser
import base64

client_id = <INSERT_CLIENT_ID>
client_secret = <INSERT_CLIENT_SECRET>
redirect_url = 'https://www.xero.com/'
scope = 'offline_access accounting.reports.read' # Change as required.
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


# 6.2 Call the API
def XeroRequests():
    old_refresh_token = open(<INSERT_FILE_PATH>, 'r').read()
    new_tokens = XeroRefreshToken(old_refresh_token)
    xero_tenant_id = XeroTenants(new_tokens[0])

    get_url = 'https://api.xero.com/api.xro/2.0/Reports/TrialBalance?Date=2024-01-31'
    response = requests.get(get_url,
                            # params={
                            #     "offset": "481"
                            # },
                            headers={
                                'Authorization': 'Bearer ' + new_tokens[0],
                                'Xero-tenant-id': xero_tenant_id,
                                'Accept': 'application/json'
                            })
    json_response = response.json()
    print(json_response)

    xero_output = open(<INSERT_FILE_PATH>, 'w')
    xero_output.write(response.text)
    xero_output.close()


old_tokens = XeroFirstAuth()
XeroRefreshToken(old_tokens[1])


# XeroRequests()
```

First we will need to login to [Xero Developer](developer.xero.com) and link our Python script. Refer the images below to create a new app:

![Xero Developer1](assets/data_retrieval5.png)

Choose any name for your app. Select "Web app" for integration type. Type "https://placeholder.com/" as the company URL for now. Type "https://xero.com/" as the redirect URL.

![Xero Developer2](assets/data_retrieval6.png)

Once created, go to Configuration in the sidebar and generate a secret. Store the secret in a .txt file for later use.

![Xero Developer3](assets/data_retrieval7.png)

* Step 2: Define API scope depending on data to retrieve. E.g. journals listing, trial balance, PnL.  

{insert Xero interface samples}  

* Step 3: Define parameters. E.g. dates, accounts, batches.  

{insert Xero interface samples}  

* Step 4: Manually run script on demand, or schedule via Windows Task Scheduler.  

{demonstrate Task Scheduler interface setup}  

## Outcome:  

Automated JSON data inflows to be used as entry point for further optimization/automation.  

{insert screenshots of JSON output}
