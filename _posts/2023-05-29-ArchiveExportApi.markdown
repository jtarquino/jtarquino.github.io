---
layout: post
title: "Archiving snapshots of reports from Power BI into OneLake"
date: 2023-05-29 09:25:00 -0000
categories: MicrosoftFabric Fabric PowerBI
---

I haven't written in a long time, however with the release last week of Microsoft Fabric and OneLake by our organization some ideas came to my mind and I want to give it a try.

Some of the new features that got my attention was the easy access from the Notebooks to
- Power BI auth token with [Microsoft Spark Utilities](https://learn.microsoft.com/en-us/azure/synapse-analytics/spark/microsoft-spark-utilities?pivots=programming-language-python)
- Write to OneLake with [Microsoft Spark Utilities](https://learn.microsoft.com/en-us/azure/synapse-analytics/spark/microsoft-spark-utilities?pivots=programming-language-python)
- [OneLake File Explorer](https://learn.microsoft.com/en-us/fabric/onelake/onelake-file-explorer) for Windows 
- Easy schedule of Notebooks 

Power BI has the [ExportTo Api](https://learn.microsoft.com/en-us/rest/api/power-bi/reports/export-to-file-in-group) which allows the export of any report in a workspace (either pbi or paginated).

Althought Power BI offers tipically a no code experience with [subscriptions](https://learn.microsoft.com/en-us/power-bi/collaborate-share/end-user-subscribe?tabs=creator) and the [Power Automate action](https://learn.microsoft.com/en-us/power-bi/collaborate-share/service-automate-power-bi-report-export)

I chose the Yes-code way and built a notebook to archive the export of a report every day (the full workbook ready to upload to Microsoft Fabric is available [here](https://github.com/jtarquino/jtarquino.github.io/blob/master/samples/ExportBlogPost.ipynb))

**The final result will be available in the lakehouse under Files/ExportedFiles**
![image](https://github.com/jtarquino/jtarquino.github.io/assets/8054158/283cf739-94a6-4139-89d8-5c58096816ab)

**Also available in Windows File Explorer**
![image](https://github.com/jtarquino/jtarquino.github.io/assets/8054158/bd1e5b72-0c25-4920-9a85-998da180db80)

**Now the walkthrough**

**1. Define the report to export** format and additional properties like filters (I used the [IT Spend Analysis sample](https://learn.microsoft.com/en-us/power-bi/create-reports/sample-it-spend)
One note of caution is that the page name for the api is the internal name not the same that you see in the report, you can use the [Get Pages Api](https://learn.microsoft.com/en-us/rest/api/power-bi/reports/get-page-in-group) or grab it from the url ![image](https://github.com/jtarquino/jtarquino.github.io/assets/8054158/62878080-ff2f-4d46-9f6b-61fcb8c4747b)

 
{% highlight python %}
import requests
import time
from notebookutils import mssparkutils
from datetime import datetime

# Set the workspace id, report id and export format

import requests
import time
from notebookutils import mssparkutils
from datetime import datetime

# Set the workspace id, report id and export format
workspace_id = "912e6fbb-75f5-40b4-ac17-7ea2a116e9b2"
report_id = "8832f89e-9610-4e10-a047-4ffbda6018ad"
export_format="PDF"

# Different than displayname, either use the GetPages api or copy from the url when the desired page is selected
page_name="ReportSection"

# Set additional export configuration
export_config = {
    "format": export_format,
    "powerBIReportConfiguration" :{
        "pages": [
            {"pageName": page_name}
        ],
        "reportLevelFilters": [
            {"filter":"Department/VP eq 'Carl Morris'"}
        ]
    },
}

# Set the header with the authentication token
auth_token = mssparkutils.credentials.getToken('pbi')
headers = {
    "Authorization": f"Bearer {auth_token}",
    "Content-Type": "application/json"
}
{% endhighlight %}

**2. Call the Power BI [ExportTo Api](https://learn.microsoft.com/en-us/rest/api/power-bi/reports/export-to-file-in-group)** to schedule the export
{% highlight python %}
# Schedule the export job in Power BI
export_url = f"https://api.powerbi.com/v1.0/myorg/groups/{workspace_id}/reports/{report_id}/ExportTo"
response = requests.post(
    url= export_url,
    headers=headers,
    json=export_config
)

# Check the response
if response.status_code == 202:
    print("Export request submitted successfully.")
    print (response.content)
else:
    print("Export request failed with status code:", response.status_code)
    print("Error message:", response.text)
{% endhighlight %}

**3. Check the export status** the Api follows an async pattern, after the job is scheduled is needed to verify the progress of if using the [check status api](https://learn.microsoft.com/en-us/rest/api/power-bi/reports/get-export-to-file-status) until the export is completed**
{% highlight python %}
# Check the export status periodically until completed
export_id =  response.json()["id"]
check_export_status_url = f"https://api.powerbi.com/v1.0/myorg/groups/{workspace_id}/reports/{report_id}/exports/{export_id}"
while True:
    response = requests.get(
        url=check_export_status_url,
        headers=headers
    )
    print(response.content)
    export_status = response.json()
    if export_status["status"] == "Succeeded":
        print("Export completed successfully.")
        break
    elif export_status["status"] in ["Failed", "Cancelled"]:
        print("Export failed or was cancelled.")
        break

    print("Export status:", export_status["status"])
    time.sleep(5)  # Wait for 5 seconds before checking again
{% endhighlight %}

**4. Save the file** the last step is to take the content and write it as a file to OneLake
{% highlight python %}
# get the content and save it to onelake
if export_status["status"] == "Succeeded":
    content_url=export_status["resourceLocation"]
    report_name=export_status["reportName"]
    report_extension=export_status["resourceFileExtension"]
    response = requests.get(
            url=content_url,
            headers=headers
        )
    # create directory
    directory = "Files/ExportedFiles"
    mssparkutils.fs.mkdirs(directory) 

    # build the path to store the file
    current_datetime = datetime.now()
    formatted_datetime = current_datetime.strftime("%Y-%m-%d %H:%M")
    filename = f"{formatted_datetime}_{report_name}{report_extension}"
    full_file_path = mssparkutils.fs.getMountPath(f"/default/{directory}/") + filename

    #save file
    with open(full_file_path, "wb") as file:
        file.write(response.content)
{% endhighlight %}
