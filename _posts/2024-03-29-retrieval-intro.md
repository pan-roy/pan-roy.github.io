---
title: 1. Automated Data Retrieval via Xero APIs
date: 2024-03-29 21:26:01 +1100
categories: [Showcase, Data Retrieval]
tags: [data retrieval, xero]     # TAG names should always be lowercase
---

## Background

Accountants usually perform manual data retrieval for further processing. Data, whether it be in the form of various transaction listings or financial statements, is used as input for further calculations and subsequently presented as independent reports or uploaded back into the accounting system (e.g. journal entries). Should there be numerous systems and sources involved, accountants would need to dedicate time towards manual data extraction in the absence of custom integrations or interconnected ERP systems.

For instance, what if we had a report that took in data from an Excel table. How could we enable the table to pull data directly from an accounting system?

![Sample report](assets/data_retrieval/data_retrieval1.png)
*Simplified report example*

![Sample data](assets/data_retrieval/data_retrieval2.png)
*Simplified data example*

Automated reporting and analytics require input data to be available in the first place - models will be rendered useless if no data was fed in. Therefore, the first step towards our analytics setup would be automated data extraction. API calls from accounting applications typically result in JSON data being returned, which would serve as our starting point.

![Sample JSON](assets/data_retrieval/data_retrieval4.png)
*JSON example*

## Goal

Our current goal is to create a Python script that automatically extracts financial data from an accounting system and sets up the relevant initial authentication procedures required.

Xero, one of the more popular cloud-based accounting software solutions, will be used this in demonstration given its availability of API documentation. Note that with the proper API access setup, the logic used here could be applied to other accounting environments, as long as documentation and/or support from IT are present.

Visit the [_Xero Main_](https://www.xero.com/au/) and [_Xero API_](https://developer.xero.com/documentation/api/accounting/overview) pages here.

![Sample Xero](assets/data_retrieval/data_retrieval3.png)
*Xero sample company*

## Steps

**Requirements**: existing Xero account, local Python interpreter, Windows, basic Python knowledge (though explanations will be provided).

Assuming that you have registered a Xero account with either the default demo company or an actual company linked, you will need to establish a long-term connection between your local computer and Xero.

Click [here](https://www.youtube.com/watch?v=t0DgAMgN8VY&list=WL&index=3) should you be interested in a detailed video tutorial (courtesy of [Edgecate](https://www.youtube.com/@edgecate)).

Assuming Python is installed, you will need to install the relevant package dependencies required for the script to run. It is recommended that you create a virtual environment for easier dependency management.

To create a virtual environment, open a PowerShell terminal and navigate to the directory you wish to setup your scripts via the ```cd``` command. For example:

```powershell
cd C:\Users\roypa\Scripts
```

Use the following command to create a virtual environment. This will prevent conflicts should you choose to work on other Python projects. Do not include the placeholder quotation marks below:

```powershell
python -m venv "INSERT_NAME_HERE"
```

You can then activate the virtual environment via the ```activate``` file within the newly created environment folder. Replace the example's path below with yours and press Enter. You should see the virtual environment show up as a  prefix in your terminal command line once done.

```powershell
C:\Users\roypa\Scripts\reportenv\Scripts\activate
```

Assuming ```pip``` is installed by default along with Python, enter the following command to install required dependencies. Should you run into any errors regarding missing dependencies later on, you can use ```pip``` to install them as well.

```powershell
pip install requests
```

Use ```pip list``` command to check whether installation is successful. Below is sample output on my end, note that yours will have much less content:

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

Now we will need to login to [Xero Developer](https://developer.xero.com) to setup our app in Xero. Refer the image below to create one:

![Xero Developer1](assets/data_retrieval/data_retrieval5.png)

Choose any name for your app. Select "Web app" for integration type. Type _"https://placeholder.com/"_ as the company URL. Type _"https://xero.com/"_ as the redirect URL.

![Xero Developer2](assets/data_retrieval/data_retrieval6.png)

Once created, go to _Configuration_ in the sidebar and generate a secret. Store the client ID and secret in a .txt file for later use.

![Xero Developer3](assets/data_retrieval/data_retrieval7.png)

With the required prep work complete, we can now move to the following Python script that will enable us to establish a long-term connection without the need for repeated logins.

* File path examples have been provided for your reference - please replace them with the corresponding paths on your end.

* Please also replace the ```client_id``` and ```client_secret``` variable with those obtained in steps above.

* You will need to determine the scope variable based on the financial data you are interested in. In this demonstration, we will be pulling the trial balance as at 31 January 2024, which uses ```scope = 'offline_access accounting.reports.read'```. Please refer to the
[Accounting API Reports](https://developer.xero.com/documentation/api/accounting/reports) page in Xero for other options.

* Within the ```XeroRequests``` function below, you will notice that there is a ```get_url``` local variable. Refer to the API documentation in the prior bullet point for the relevant endpoint URLs. This will determine what action will be taken and what input parameters will be for the content requested (e.g. period end date for trial balances). In this instance we will be using ```'https://api.xero.com/api.xro/2.0/Reports/TrialBalance?Date=2024-01-31'```, where ```TrialBalance``` is the item requested, and ```Date=2024-01-31``` is the input parameter for our report date.

```python
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
    old_refresh_token = open('C:/Users/roypa/Downloads/refresh_token.txt', 'r').read()
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

    xero_output = open('C:/Users/roypa/Downloads/xero_output.json', 'w')
    xero_output.write(response.text)
    xero_output.close()


old_tokens = XeroFirstAuth()
XeroRefreshToken(old_tokens[1])


# XeroRequests()
```

![API details](assets/data_retrieval/data_retrieval8.png)
*API details*

After populating the variables above and running the script (via PowerShell etc.), you will be directed to a login page for end-user authorization. Login with your Xero account to obtain access.

![authorization](assets/data_retrieval/data_retrieval9.png)
*Authorization*

Once logged in, copy the response page URL back into the Python terminal and press Enter to obtain a refreshable access token under the designated folder.

![authorization page](assets/data_retrieval/data_retrieval10.png)
*Authorization link*

![access token](assets/data_retrieval/data_retrieval11.png)
*Access token*

![token file](assets/data_retrieval/data_retrieval12.png)
*Token .txt file*

Access tokens are only valid for approximately 10 minutes, after which access will be terminated - similar to how users are automatically logged out after periods of idle activity. However, the ```XeroRequests``` function automatically refreshes the token whenever an API call is made, and thus granting us long-term access.

You will now have to uncomment the ```XeroRequests()``` at the bottom of the script and comment out the ```XeroFirstAuth()``` and ```XeroRefreshToken(old_tokens[1])``` lines above. You will only need to run these 2 functions once to obtain your access token. You will only need ```XeroRequests()``` going forward. Once done, rerun the script to call the designated API. You should obtain a JSON file containing the data in the path designated. In this case, we now have our trial balance as at 31 January 2024.

![JSON file](assets/data_retrieval/data_retrieval13.png)

![output](assets/data_retrieval/data_retrieval14.png)
*Sample output*

This script can now be manually run on-demand, or alternatively you can use Windows Task Scheduler to schedule the script to run at certain intervals - this will be covered in future walkthroughs.

![task scheduler](assets/data_retrieval/data_retrieval15.png)

## Outcome

We now have automated JSON data inflows to be used as our entry point for further optimization/automation. The next step would be to convert JSON files received into a tabular format such as CSV files {TODO: insert link here}, for us to read the data easily and to transform data into useful insights.
