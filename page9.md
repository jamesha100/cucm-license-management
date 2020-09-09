## Article 9 - Saving Data for Devices With No Owner Assigned

After running the scripts in the previous articles all devices that should be allocated owners should have owners assigned. This script creates a CSV file listing devices without owners which may be reviewed by the system administrator. Note that some devices, such as shared area phones, conference phones and room based video units do not normally have users assigned. All softphone devices however should have owners. If any do not then they should be reviewed to check whether the user for which they were configured is still active on the system or whether the number assigned to the device's Line 1 does not match the *telephonenumber* in the user record. If the users of softphone devices are no longer configured on CUCM then the devices may be deleted to free up licenses.

- [Article 1 - Introduction and CUCM Licensing Overview.](https://jamesha100.github.io/cucm-license-management/page1)
- [Article 2 - Basic SQL Commands to Monitor and Update Licensing.](https://jamesha100.github.io/cucm-license-management/page2)
- [Article 3 - Using the CUCM AXL API.](https://jamesha100.github.io/cucm-license-management/page3)
- [Article 4 - Saving Existing Device Ownership Data](https://jamesha100.github.io/cucm-license-management/page3)
- [Article 5 - Restoring Saved Device Ownership Data](https://jamesha100.github.io/cucm-license-management/page5)
- [Article 6 - Clearing Device Ownership Data](https://jamesha100.github.io/cucm-license-management/page6)
- [Article 7 - Setting Device Ownership by Primary Extension](https://jamesha100.github.io/cucm-license-management/page7)
- [Article 8 - Allocating Owners to Extension Mobility Devices](https://jamesha100.github.io/cucm-license-management/page8)
- Article 9 - Saving Data for Devices With No Owner Assigned

### SQL Query Used to Retrieve Device Ownership Data

The SQL query we will use to retrieve the device data is shown below.
```
SELECT d.pkid,d.name AS devicename, d.description,tm.name AS devicemodel FROM device AS d INNER JOIN typemodel AS tm ON tm.enum = d.tkmodel WHERE d.tkclass = '1' AND NOT (d.tkmodel = '72' OR d.tkmodel = '645') AND fkenduser IS NULL ORDER BY devicemodel, devicename
```
This query will retrieve the following data for any phone device that has an owner assigned.
- Device pkid.
- Device name.
- Device description.
- Device model.

Only devices that do not have owners assigned (the fkenduser filed is set to NULL) will be returned in the query. 

### The Python Script
The Python script shown below will retrieve the data listed above and save it to a CSV file. The name of the CSV file includes date and time information so that new output files can be created without overwriting existing files.

The script comprises the following high level steps:

1. Load information needed to connect to the target CUCM server from an ini file. 
2. Connect to the CUCM AXL API using the settings from the ini file to retrieve and store cookies data. (this step is not strictly necessary as the script only makes one other connection to the AXL API. It is used for consistency with the other Python scripts used in this series of articles.
3. Connect to the CUCM AXL API, execute the SQL query detailed above. This will return an XML document containing the required data.
4. Extract the data from the XML document to an ordered dictionary.
5. Loop through the dictionary to extract the data from the dictionary and store in a list of lists.
6. Create the filename for the CSV file which includes the date and time of file creation.
7. Write header values to the CSV file first line.
8. Loop through the list of lists and write data for each individual device to a new line in the CSV file.

```
import sys
import requests
from urllib3 import disable_warnings
from urllib3.exceptions import InsecureRequestWarning
from configparser import ConfigParser
import datetime
import pprint
import xmltodict
import csv

disable_warnings(InsecureRequestWarning)


########################################################################################################################

def axlgetcookies(serveraddress, version, axluser, axlpassword):
    try:
        # Make AXL query using Requests module
        axlgetcookiessoapresponse = requests.get(f'https://{ipaddr}:8443/axl/', headers=soapheaders, verify=False,\
                                                 auth=(axluser, axlpassword), timeout=3)
        print(axlgetcookiessoapresponse)
        getaxlcookiesresult = axlgetcookiessoapresponse.cookies

        if '200' in str(axlgetcookiessoapresponse):
            # request is successful
            print('AXL Request Successful')
        elif '401' in str(axlgetcookiessoapresponse):
            # request fails due to invalid credentials
            print('Response is 401 Unauthorised - please check credentials')
        else:
            # request fails due to other cause
            print(f'Request failed! - HTTP Response Code is {axlgetcookiessoapresponse}')

    except requests.exceptions.Timeout:
        axlgetcookiesresult = 'Error: IP address not found!'

    except requests.exceptions.ConnectionError:
        axlgetcookiesresult = 'Error: DNS lookup failed!'

    return getaxlcookiesresult


########################################################################################################################

def axlgetdeviceownerdata(cookies):
    # Set SOAP request body
    axlgetdeviceownerdatasoaprequest = f'<?xml version="1.0" encoding="UTF-8"?>\
    <soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"\
     xmlns:ns="http://www.cisco.com/AXL/API/{version}"><soapenv:Header/><soapenv:Body><ns:executeSQLQuery><sql>\
     SELECT d.pkid,d.name AS devicename, d.description,tm.name AS devicemodel FROM device AS d INNER JOIN typemodel ' \
    f'AS tm ON tm.enum = d.tkmodel WHERE d.tkclass = \'1\' AND NOT (d.tkmodel = \'72\' OR d.tkmodel = \'645\') AND ' \
    f'fkenduser IS NULL ORDER BY devicemodel, devicename</sql></ns:executeSQLQuery></soapenv:Body>\
       </soapenv:Envelope>'

    try:

        # Make SOAP request
        axlgetdeviceownerdatasoapresponse = requests.post(f'https://{ipaddr}:8443/axl/',\
                                                          data=axlgetdeviceownerdatasoaprequest, headers=soapheaders,\
                                                          verify=False, cookies=cookies, timeout=3)
        # Parse SOAP XML response to a dictionary
        axlgetdeviceownerdatasoapresponse_dict = xmltodict.parse(axlgetdeviceownerdatasoapresponse.text)
        # Remove unwanted parts of the dictionary to leave the data we require
        axlgetdeviceownerdatasoapresponse_subdict = \
            axlgetdeviceownerdatasoapresponse_dict['soapenv:Envelope']['soapenv:Body']['ns:executeSQLQueryResponse']\
                ['return']['row']
        pprint.pprint(axlgetdeviceownerdatasoapresponse_subdict)

        axlgetdeviceownerdatasoapresponse_list = []

        for item in axlgetdeviceownerdatasoapresponse_subdict:
            devicerecord = []
            devicerecord.append(item['pkid'])
            devicerecord.append(item['devicename'])
            devicerecord.append(item['description'])
            devicerecord.append(item['devicemodel'])
            axlgetdeviceownerdatasoapresponse_list.append(devicerecord)

        return axlgetdeviceownerdatasoapresponse_list

    except requests.exceptions.Timeout:
        print('IP address not found! ')
    except requests.exceptions.ConnectionError:
        print('DNS lookup failed!  ')


########################################################################################################################

def main():
    # Import values for the CUCM environment from ini file using ConfigParser

    global ipaddr, version, axluser, axlpassword, soapheaders, cookies
    parser = ConfigParser()
    parser.read('cucm-settings.ini')

    ipaddr = parser.get('CUCM', 'ipaddr')
    version = parser.get('CUCM', 'version')
    axluser = parser.get('CUCM', 'axluser')
    axlpassword = parser.get('CUCM', 'axlpassword')
    soapheaders = {'Content-type': 'text/xml', 'SOAPAction': f'CUCM:DB ver={version}'}

    # Get cookies for future AXL requests
    axlgetcookiesresult = axlgetcookies(ipaddr, version, axluser, axlpassword)
    print(axlgetcookiesresult)

    if 'JSESSIONIDSSO' in str(axlgetcookiesresult):
        print('Cookies Retrieved')

    else:
        sys.exit('Program has ended due to error!')

    # Get Device Ownership Data

    deviceownerdata_list = axlgetdeviceownerdata(axlgetcookiesresult)
    print(deviceownerdata_list)

    # Create filename for output file in format devicenoownerdata-year-month-dom-hour-minute-second.csv

    now = datetime.datetime.now()
    outputfile = 'devicenoownerdata-' + now.strftime("%Y-%m-%d-%H-%M-%S") + '.csv'
    print(outputfile)

    # Write device ownership data to csv file
    deviceownerdataoutputfile = open(outputfile, 'w', newline='')
    deviceownerdataoutputwriter = csv.writer(deviceownerdataoutputfile)
    deviceownerdataoutputwriter.writerow(
        ['devicepkid', 'devicename', 'description', 'devicemodel'])
    for item in deviceownerdata_list:
        deviceownerdataoutputwriter.writerow(item)

    deviceownerdataoutputfile.close()


########################################################################################################################

if __name__ == "__main__":
    main()

```
#### The Output Data
The csv file output should look similar to that shown below - thanks to [Donat Studios](https://donatstudios.com/CsvToMarkdownTable) for providing a tool to convert CSV data to Markdown.

| devicepkid                           | devicename      | description              | devicemodel                             | 
|--------------------------------------|-----------------|--------------------------|-----------------------------------------| 
| f838da89-d68f-43fe-a3ea-59caf22caad0 | SEPE0D173E4AA6A | Demo                     | Cisco 7861                              | 
| 44482f47-e7d4-c1fe-3ae7-ed08b4f03366 | SEP00CAE541646B | James Test 8841          | Cisco 8841                              | 
| 2463e6c0-71e8-8d87-7526-01c981466bc7 | jeffh           |                          | Cisco IP Communicator                   | 
| 6eb51642-2723-8ceb-0431-6d1f70437fb8 | CSFJHAWKINS     | James Hawkins Jabber CSF | Cisco Unified Client Services Framework | 
