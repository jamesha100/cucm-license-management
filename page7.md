## Article 7 - Setting Device Ownership by Primary Extension

This article covers setting device device ownership information by using the association between the telephone number allocated to the end user in the *enduser* table and the Directory Number assigned to the primary line of telephone devices in the *device* table.

- [Article 1 - Introduction and CUCM Licensing Overview.](https://jamesha100.github.io/cucm-license-management/page1)
- [Article 2 - Basic SQL Commands to Monitor and Update Licensing.](https://jamesha100.github.io/cucm-license-management/page2)
- [Article 3 - Using the CUCM AXL API.](https://jamesha100.github.io/cucm-license-management/page3)
- [Article 4 - Saving Existing Device Ownership Data.](https://jamesha100.github.io/cucm-license-management/page4)
- [Article 5 - Restoring Saved Device Ownership Data](https://jamesha100.github.io/cucm-license-management/page5)
- [Article 6 - Clearing Device Ownership Data](https://jamesha100.github.io/cucm-license-management/page6)
- Article 7 - Setting Device Ownership by Primary Extension

### SQL Query Used to Select User Data from enduser Table

The SQL query we will use to get user information from the *enduser* table is shown below.
```
SELECT pkid, userid, telephonenumber FROM enduser ORDER BY userid
```
The query selects the following information:
- **pkid** - the primary key for the user.
- **userid** - the CUCM user ID for the user.
- **telephonenumber** - the telephone number assigned to the user.

### SQL Query Used to Set Device Ownership by Primary Extension

The SQL query we will use to set the device owner by primary extension is shown below.
```
UPDATE device SET fkenduser = "fkenduser" WHERE pkid IN (SELECT fkdevice FROM devicenumplanmap WHERE fknumplan = (SELECT pkid FROM numplan WHERE dnorpattern = "telephonenumber") AND numplanindex = 1) AND tkclass = '1'
```


### The Python Script
The Python script shown below will retrieve the information listed below from the *enduser* table:
- **pkid** - this value is used to update the *fkenduser* value in the *devices* table.
- **userid** - this is used to report success/failure in the program log.
- **telephonenumber** - this is used to identify which devices to update with owner information.

The script then loops through the user data and sets the Owner ID (fkenduser) of any device whose line 1 extension matches the user *telephonenumber* value

The script comprises the following high level steps:

1. Load information needed to connect to the target CUCM server from an ini file.
2. Connect to the CUCM AXL API using the settings from the ini file and download *pkid*, *userid* and *telephonenumber* for all users in the *enduser* table - these are stored as an ordered list.


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
from csv import DictReader

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
            print('Request failed! - HTTP Response Code is ' + axlgetcookiessoapresponse)

    except requests.exceptions.Timeout:
        axlgetcookiesresult = 'Error: IP address not found!'

    except requests.exceptions.ConnectionError:
        axlgetcookiesresult = 'Error: DNS lookup failed!'

    return getaxlcookiesresult

########################################################################################################################

def axlgetenduserdata(cookies):
    # Set SOAP request body
    axlgetenduserdatasoaprequest = f'<?xml version="1.0" encoding="UTF-8"?>\
    <soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"\
     xmlns:ns="http://www.cisco.com/AXL/API/{version}"><soapenv:Header/><soapenv:Body><ns:executeSQLQuery><sql>\
    SELECT pkid, userid, telephonenumber FROM enduser ORDER BY userid</sql></ns:executeSQLQuery></soapenv:Body>\
       </soapenv:Envelope>'

    try:

        # Make SOAP request
        axlgetenduserdatasoapresponse = requests.post(f'https://{ipaddr}:8443/axl/',\
                                                          data= axlgetenduserdatasoaprequest, headers=soapheaders,\
                                                          verify=False, cookies=cookies, timeout=3)
        # Parse SOAP XML response to a dictionary
        axlgetenduserdatasoapresponse_dict = xmltodict.parse(axlgetenduserdatasoapresponse.text)
        # Remove unwanted parts of the dictionary to leave the data we require
        axlgetenduserdatasoapresponse_subdict = \
            axlgetenduserdatasoapresponse_dict['soapenv:Envelope']['soapenv:Body']['ns:executeSQLQueryResponse']\
                ['return']['row']
        pprint.pprint(axlgetenduserdatasoapresponse_subdict)

        return axlgetenduserdatasoapresponse_subdict

    except requests.exceptions.Timeout:
        print('IP address not found! ')
    except requests.exceptions.ConnectionError:
        print('DNS lookup failed!  ')


########################################################################################################################

def axlsetdeviceowner(cookies, fkenduser, telephonenumber):
    # Set SOAP request body
    axlsetdeviceownersoaprequest = f'<?xml version="1.0" encoding="UTF-8"?><soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:ns="http://www.cisco.com/AXL/API/{version}"><soapenv:Header/><soapenv:Body><ns:executeSQLUpdate><sql>UPDATE device SET fkenduser = "{fkenduser}" WHERE pkid IN (SELECT fkdevice FROM devicenumplanmap WHERE fknumplan = (SELECT pkid FROM numplan WHERE dnorpattern = "{telephonenumber}" AND fkroutepartition = (SELECT pkid FROM routepartition WHERE name = "{user_dn_partition}")) AND numplanindex = 1) AND tkclass = "1"</sql></ns:executeSQLUpdate></soapenv:Body></soapenv:Envelope>'

    try:

        # Make SOAP request
        axlsetdeviceownersoapresponse = requests.post(f'https://{ipaddr}:8443/axl/', data=axlsetdeviceownersoaprequest, headers=soapheaders, verify=False, cookies=cookies, timeout=9)

        # Parse SOAP XML response to a dictionary
        axlsetdeviceownersoapresponse_dict = xmltodict.parse(axlsetdeviceownersoapresponse.text)
        # pprint.pprint(axlsetdeviceownersoapresponse_dict)
        devices_updated = axlsetdeviceownersoapresponse_dict['soapenv:Envelope']['soapenv:Body']['ns:executeSQLUpdateResponse']['return']['rowsUpdated']
        return devices_updated

        '''
        # Remove unwanted parts of the dictionary to leave the data we require
        axlgetenduserdatasoapresponse_subdict = \
            axlgetenduserdatasoapresponse_dict['soapenv:Envelope']['soapenv:Body']['ns:executeSQLQueryResponse'] \
                ['return']['row']
        pprint.pprint(axlgetenduserdatasoapresponse_subdict)

        return axlgetenduserdatasoapresponse_subdict
        
    
        if '200' in str(axlsetdeviceownersoapresponse):
            # request is successful
            device_update = 'Success'
            pprint.pprint(axlsetdeviceownersoapresponse.text)
        else:
            # request is unsuccessful
            device_update = 'Failure'

        return device_update
        '''

    except requests.exceptions.Timeout:
        print('IP address not found! ')
    except requests.exceptions.ConnectionError:
        print('DNS lookup failed!  ')


########################################################################################################################

def main():
    # Import values for the CUCM environment from ini file using ConfigParser

    global ipaddr, version, axluser, axlpassword, user_dn_partition, soapheaders, cookies
    parser = ConfigParser()
    parser.read('cucm-settings.ini')

    ipaddr = parser.get('CUCM', 'ipaddr')
    version = parser.get('CUCM', 'version')
    axluser = parser.get('CUCM', 'axluser')
    axlpassword = parser.get('CUCM', 'axlpassword')
    user_dn_partition = parser.get('CUCM', 'user_dn_partition')
    soapheaders = {'Content-type': 'text/xml', 'SOAPAction': 'CUCM:DB ver=' + version}


    # Get cookies for future AXL requests
    axlgetcookiesresult = axlgetcookies(ipaddr, version, axluser, axlpassword)
    print(axlgetcookiesresult)

    if 'JSESSIONIDSSO' in str(axlgetcookiesresult):
        print('Cookies Retrieved')

    else:
        sys.exit('Program has ended due to error!')

    # Get end user data via AXL function and read into list of lists
    enduserdata_dict = axlgetenduserdata(axlgetcookiesresult)
    #pprint.pprint(enduserdata_dict)

    # Set device ownership via AXL function using telephonenumber to primary extension mapping

    for item in enduserdata_dict:
        fkenduser = (item['pkid'])
        userid = (item['userid'])
        telephonenumber = (item['telephonenumber'])
        # print(f'{fkenduser}, {userid}, {telephonenumber}')

        if str(telephonenumber) != 'None':
            devices_updated = axlsetdeviceowner(axlgetcookiesresult, fkenduser, telephonenumber)
            print(f'{fkenduser}, {userid}, {telephonenumber}, {devices_updated}')
        else:
            print(f'{fkenduser}, {userid}, {telephonenumber}, no phone number in user record')


    '''
    # Create filename for output file in format deviceownerdata-year-month-dom-hour-minute-second.csv
    now = datetime.datetime.now()
    outputfile = 'deviceownerupdatelog-' + now.strftime("%Y-%m-%d-%H-%M-%S") + '.csv'

    # Open output log csv file
    deviceownerdataoutputfile = open(outputfile, 'w', newline='')
    deviceownerdataoutputwriter = csv.writer(deviceownerdataoutputfile)
    deviceownerdataoutputwriter.writerow(['devicepkid', 'devicename', 'fkenduser', 'userid', 'Result'])

    # Open input CSV file and read device pkid and fkenduser values
    with open(deviceownerdatainputfile, mode='r') as csv_file:
        csv_dict_reader = DictReader(csv_file)
        for row in csv_dict_reader:
            devicepkid = row['devicepkid']
            devicename = row['devicename']
            fkenduser = row['fkenduser']
            userid = row['userid']

            # Call update device owner function
            update_status = axlupdatedeviceownerdata(axlgetcookiesresult,devicepkid,fkenduser)
            deviceownerdataoutputwriter.writerow([devicepkid,devicename,fkenduser,userid,update_status])
            if update_status == "Success":
                print(f'Success! - User "{userid}" set as the owner of {devicename}')
            else:
                print(f'Failure! - User "{userid}" not set as the owner of {devicename}')

    deviceownerdataoutputfile.close()
    '''

########################################################################################################################

if __name__ == "__main__":
    main()
```
