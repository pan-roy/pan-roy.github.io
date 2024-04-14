---
title: 4. Automated Manual Journaling
date: 2024-03-26 21:26:01 +1100
categories: [Showcase, Data Retrieval and Upload]
tags: [data upload, xero]     # TAG names should always be lowercase
---

## How this impacts you

Instead of mere data retrieval in our previous tutorials, what if we aimed to automate the process of manual journaling?

Xero supports CSV template upload for manual journals. However, it still requires end-users to go through a manual process of logging in and downloading & uploading CSV templates.

![manual journal](assets/manual_journals/manual_journals1.png)
*"Manual" manual journaling*

![import csv](assets/manual_journals/manual_journals2.png)
*CSV import*

Through utilizing POST Manual Journals API calls, we have the capability to automate the posting of manual journals directly within Excel. This eliminates the need for accountants to log into their accounting system and manually navigate to the journal entry interface, a process susceptible to human error and time-consuming, particularly when multiple journal entries are uploaded simultaneously.

## Goal

Our current objective is to establish an automation pipeline that empowers end-users to transmit journal entry details directly from Excel to Xero, eliminating the requirement to exit Excel or engage with the accounting system, specifically Xero, during this process.

![front-end](assets/manual_journals/manual_journals3.png)
*Simplified front-end example*

## Steps

**Requirements**: local Python interpreter, Excel, basic Python knowledge.

To accomplish our goal, we would need to execute several steps:

* Install and configure the xlwings library in Python, along with the xlwings Excel add-in.

* Create a Python script to establish a connection with Xero.

* Create a Python function to convert tabular data into JSON format.

* Create a VBA macro to trigger the Python scripts within Excel.

**Step 1: Install and configure the xlwings library in Python, along with the xlwings Excel add-in.**

Excel does not have native functionality in regard to running Python scripts (excluding Microsoft's recent cloud-based "Python in Excel" addition). We therefore require an external Python library, xlwings, to facilitate the connection between Excel and Python.

To install xlwings, activate your virtual environment (refer prior tutorials{insert link here}), and then enter ```pip install xlwings``` in a PowerShell terminal. Should you encounter any additional missing packages during the following process, use the ```pip install``` command to install them. Enter ```pip list``` to check whether you have successfully installed xlwings.

For detailed documentation on xlwings, please visit [here](https://docs.xlwings.org/en/latest/).

We will now install the xlwings Excel add-in. Close all open Excel instances (if any) and enter ```xlwings addin install``` in your PowerShell terminal. Once complete, check whether the installation has been successful by opening a new Excel instance. You should see a new add-in at the end of your top ribbon. This will grant Excel the ability to trigger Python scripts.

![add-in](assets/manual_journals/manual_journals4.png)

**Step 2: Create a Python script to establish a connection with Xero.**

Similar to our prior procedures relating to trial balances and transaction listings, we can modify our pre-existing Python script's variables to post manual journals instead. Refer the script below:

<details>
<summary>Click here to expand</summary>

{% highlight python %}
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
{% endhighlight %}

</details>
&nbsp;

**Step 3: Create a Python function to convert tabular data into JSON format.**

Please ensure you have installed the pandas library by running ```pip install pandas``` in PowerShell.

The ```convert_json()``` function, as described above, utilizes the pandas library to convert tabular Excel data into dataframes. The data can then be manipulated for further processing and upload.

Assuming there is a defined table named "input" in the first sheet of our Excel workpaper (to be created), the function extracts the journal entries' relevant information such as account code, journal amount, and description. This data is formatted to meet Xero's manual journal requirements, where the journal is posted in the current open period. The function extracts the description from the first row (excluding headers), and adds all the listed journal amounts and account codes.

The resulting dataframe is converted into a dictionary using the ```to_dict()``` method, resembling the structure of JSON files. Finally, this converted data can be passed to the ```import_journals()``` function for upload. Refer to the script for further details.

**Step 4: Create a VBA macro to trigger Python scripts within Excel.**

We now need a front-end for the end-user to input data. Open up PowerShell or any terminal available (e.g. VS Code or PyCharm) and create a new xlwings Excel project via the command ```xlwings quickstart myproject```. You will need to activate your virtual environment first. You can also change the ```myproject``` to any name you would like.

* Once done, you should see a new folder with an Excel .xlsm workbook and a .py file in it. 

* Ignore the .py file for now and open the Excel file. Create a table and assign a defined name for the table "input".

* Add an VBA form button via the "Developer" tab in the top ribbon - you may need to enable it if absent. If so, on the File tab, go to Options > Customize Ribbon. Under Customize the Ribbon and under Main Tabs, select the Developer check box. Rename your button to "Upload Journal" or any other preferred name.

Your workbook should look like this (minus the data):

![front-end](assets/manual_journals/manual_journals3.png)
*Simplified front-end example*

Now we need to setup the VBA macro. Press Alt + F11 to open the VBA editor interface.

You will notice that there are existing modules created by xlwings. Go to Module 1 and copy the first ```SampleCall()``` sample macro, paste the copy below and change ```.main()``` to any function name you prefer - this will be the name for the upload function in our revised script (to be created below). In this case, the function will be called ```.upload()```.

```vb
Public Sub GenerateJSON()
    mymodule = Left(ThisWorkbook.name, (InStrRev(ThisWorkbook.name, ".", -1, vbTextCompare) - 1))
    RunPython "import " & mymodule & ";" & mymodule & ".upload()"
End Sub
```
Go back to the VBA button, right click and "Assign Macro". Choose ```GenerateJSON()```  as per the VBA macro above.

This will direct Excel to search within the .py file previously mentioned for the specified function. Now all we need to do is to copy over our script to said .py file. Do not change the .py file's name.

Open the newly created .py file within the xlwings project folder previously ignored. Copy over the contents in our prior Python script into this one. Copy over the dependencies as well. Do not overwrite any pre-existing code in the .py file. There are minor adjustments required - ```convert_json()``` has been renamed ```upload()``` and an additional call to ```import_journals()``` within the ```upload()``` function has been added. Refer the revised script below for the mentioned adjustments:

```python
import requests
import webbrowser
import base64
import xlwings as xw
import pandas as pd


def main():
    wb = xw.Book.caller()
    sheet = wb.sheets[0]
    if sheet["A1"].value == "Hello xlwings!":
        sheet["A1"].value = "Bye xlwings!"
    else:
        sheet["A1"].value = "Hello xlwings!"


if __name__ == "__main__":
    xw.Book("testproject.xlsm").set_mock_caller()
    main()

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
                             headers={
                                 'Authorization': 'Basic ' + b64_id_secret
                             },
                             data={
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


def upload():
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

    import_journals(json_data)


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
```

As mentioned above, the journal import is now contained within the ```.upload()``` function. This is due to the fact that we have setup our VBA code to call functions, but not to run entire Python scripts. Save the .py file and exit. The setup is now complete.

Go back to the Excel workpaper and populate the table with the accounts and amounts you wish to post - refer Xero's Chart of Accounts for details. Click on the "Upload" button and let the script execute.

![before](assets/manual_journals/manual_journals5.png)
*Before upload*

![after](assets/manual_journals/manual_journals6.png)
*After upload*

![journal details](assets/manual_journals/manual_journals7.png)
*Journal details*

You can see that the journal has been uploaded successfully. All that is left is for the financial accountant/manager to approve the journals.

Once approved, you can see that our manual journal has flowed through to all reports:

![transaction list](assets/manual_journals/manual_journals8.png)
*Account Transactions*

![journal list](assets/manual_journals/manual_journals9.png)
*Account Transactions*

![trial balance](assets/manual_journals/manual_journals10.png)
*Trial Balance*

![balance sheet](assets/manual_journals/manual_journals11.png)
*Balance sheet*

## Outcome

This solution offers significant scalability, enabling the user to automate the posting of multiple journals simultaneously. The upload process for accountants remains consistent regardless of the number of journals being uploaded. Additionally, this script can be replicated for use across different companies (e.g. subsidiaries) and projects (e.g. project accounting) if required.

Furthermore, the journal table within the Excel front-end spreadsheet can be connected to refreshable Power Query connections. With the appropriate configuration and integration of logic / calculations, a large number of journals can be automated, requiring only end-user review and approval. These tools will be further explored in subsequent tutorials.
