## Article 8 - Allocating Owners to Extension Mobility Devices

This article covers setting device device ownership information by using the association between the telephone number allocated to the end user in the *enduser* table and the Directory Number assigned to the primary line of telephone devices in the *device* table.

- [Article 1 - Introduction and CUCM Licensing Overview.](https://jamesha100.github.io/cucm-license-management/page1)
- [Article 2 - Basic SQL Commands to Monitor and Update Licensing.](https://jamesha100.github.io/cucm-license-management/page2)
- [Article 3 - Using the CUCM AXL API.](https://jamesha100.github.io/cucm-license-management/page3)
- [Article 4 - Saving Existing Device Ownership Data.](https://jamesha100.github.io/cucm-license-management/page4)
- [Article 5 - Restoring Saved Device Ownership Data](https://jamesha100.github.io/cucm-license-management/page5)
- [Article 6 - Clearing Device Ownership Data](https://jamesha100.github.io/cucm-license-management/page6)
- [Article 7 - Setting Device Ownership by Primary Extension](https://jamesha100.github.io/cucm-license-management/page7)
- Article 8 - Allocating Owners to Extension Mobility Devices

### SQL Query Used to Select User Data from enduser Table

The SQL query we will use to get user information from the *enduser* table is shown below.
```
SELECT DISTINCT eudm.fkenduser FROM enduserdevicemap AS eudm INNER JOIN enduser AS eu ON eudm.fkenduser = eu.pkid INNER JOIN device AS d ON eudm.fkdevice = d.pkid WHERE d.tkclass=254 ORDER BY eudm.fkenduser
```
The query selects the following information:
- **eudm.fkenduser** - the primary key for the user.
The query selects the pkid for users who have one or more Extension Mobility User Device Profiles associated with their user account in the *enduserdevicemap* table.

### SQL Query Used to Select Device Data from device Table

The SQL query we will use to get user information from the *enduser* table is shown below.
```
SELECT pkid, name FROM device WHERE allowhotelingflag = "t" AND tkclass = "1" ORDER BY name
```
The query selects the following information:
- **pkid** - the primary key for the device.
- **name** - the name assigned to the device.
The query selects the pkid for users who have one or more Extension Mobility User Device Profiles associated with their user account in the *enduserdevicemap* table.


### SQL Query Used to Set Device Ownership by Primary Extension

The SQL query we will use to set the device owner by primary extension is shown below.
```
UPDATE device SET fkenduser = "{updatedevicefkenduser}" WHERE pkid = "{updatedevicepkid}"
```


### The Python Script
The Python script shown below retrieves lists of users who have at least one Extension Mobility User Device profile associated with their account and devices for which the Extension Mobility feature is enabled. It then assigns owners to devices (each user will be the owner of one device).


The script then loops through the user data and sets the Owner ID (fkenduser) of any device whose line 1 extension matches the user *telephonenumber* value.xxxxxxxxxxxxxxxx

The script comprises the following high level steps:

1. Load information needed to connect to the target CUCM server from an ini file.
2. Connect to the CUCM AXL API using the settings from the ini file and download *pkid* for all users in the *enduser* table who have at least one Extension Mobility User Device Profile associated with their account - these are stored in a list.
3. Connect to the CUCM AXL API using the settings from the ini file and download *pkid* and *name* for all devices in the *device* table which have the Extension Mobility feature enabled - these are stored in a list.
4. Loop the the list of devices obtained in the previous step and set the *OwnerId* of the device to a user from the list of users obtained in step 2 - the user allocated is incremented as the loop progresses. 


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

def axlgetextmobusers(cookies):
    axlgetextmobuserssoaprequest = f'<?xml version="1.0" encoding="UTF-8"?><soapenv:Envelope' \
                                   f' xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"' \
                                   f' xmlns:ns="http://www.cisco.com/AXL/API/{version}"><soapenv:Header/>' \
                                   f'<soapenv:Body><ns:executeSQLQuery><sql>SELECT DISTINCT eudm.fkenduser FROM' \
                                   f' enduserdevicemap AS eudm INNER JOIN enduser AS eu ON eudm.fkenduser = eu.pkid' \
                                   f' INNER JOIN device AS d ON eudm.fkdevice = d.pkid WHERE d.tkclass=254 ORDER BY' \
                                   f' eudm.fkenduser</sql></ns:executeSQLQuery></soapenv:Body></soapenv:Envelope>'


    try:

        axlgetextmobuserssoapresponse = requests.post(f'https://{ipaddr}:8443/axl/', \
        data=axlgetextmobuserssoaprequest, headers=soapheaders, verify=False, cookies=cookies, timeout=9)

        axlgetextmobuserssoapresponse_dict = xmltodict.parse(axlgetextmobuserssoapresponse.text)

        axlgetextmobuserssoapresponse_subdict = axlgetextmobuserssoapresponse_dict['soapenv:Envelope']['soapenv:Body'] \
        ['ns:executeSQLQueryResponse']['return']['row']
        axlgetextmobuserssoapresponse_list = []
        for item in axlgetextmobuserssoapresponse_subdict:
            axlgetextmobuserssoapresponse_list.append(item['fkenduser'])

        return axlgetextmobuserssoapresponse_list

    except requests.exceptions.Timeout:
        print('IP address not found! ')
    except requests.exceptions.ConnectionError:
        print('DNS lookup failed!  ')

########################################################################################################################

def axlgetextmobdevices(cookies):
    axlgetextmobdevicessoaprequest = f'<?xml version="1.0" encoding="UTF-8"?><soapenv:Envelope ' \
                                     f'xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" ' \
                                     f'xmlns:ns="http://www.cisco.com/AXL/API/{version}"><soapenv:Header/>' \
                                     f'<soapenv:Body><ns:executeSQLQuery><sql>SELECT pkid, name FROM device WHERE ' \
                                     f'allowhotelingflag = "t" AND tkclass = "1" ORDER BY name</sql>' \
                                     f'</ns:executeSQLQuery></soapenv:Body></soapenv:Envelope>'

    try:

        axlgetextmobdevicessoapresponse = requests.post(f'https://{ipaddr}:8443/axl/', \
        data=axlgetextmobdevicessoaprequest, headers=soapheaders, verify=False, cookies=cookies, timeout=9)

        axlgetextmobdevicessoapresponse_dict = xmltodict.parse(axlgetextmobdevicessoapresponse.text)

        axlgetextmobdevicessoapresponse_subdict = axlgetextmobdevicessoapresponse_dict['soapenv:Envelope'] \
        ['soapenv:Body']['ns:executeSQLQueryResponse']['return']['row']

        axlgetextmobdevicessoapresponse_list = []
        for item in axlgetextmobdevicessoapresponse_subdict:
            axlgetextmobdevicessoapresponse_list.append(item['pkid'])

        return axlgetextmobdevicessoapresponse_list

    except requests.exceptions.Timeout:
        print('IP address not found! ')
    except requests.exceptions.ConnectionError:
        print('DNS lookup failed!  ')

########################################################################################################################

def axlallocatedeviceowner(cookies, updatedevicepkid, updatedevicefkenduser):
    axlallocatedeviceownersoaprequest = f'<?xml version="1.0" encoding="UTF-8"?><soapenv:Envelope ' \
                                        f'xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" ' \
                                        f'xmlns:ns="http://www.cisco.com/AXL/API/{version}"><soapenv:Header/>' \
                                        f'<soapenv:Body><ns:executeSQLUpdate><sql>UPDATE device SET fkenduser = ' \
                                        f'"{updatedevicefkenduser}" WHERE pkid = "{updatedevicepkid}"' \
                                        f'</sql></ns:executeSQLUpdate></soapenv:Body></soapenv:Envelope>'

    try:
        axlallocatedeviceownersoapresponse = requests.post('https://' + ipaddr + ':8443/axl/', \
        data=axlallocatedeviceownersoaprequest, headers=soapheaders, verify=False, cookies=cookies, timeout=9)

        return

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

    # Get list of users with an Extension Mobility profile allocated to them
    extmobusers_list = axlgetextmobusers(axlgetcookiesresult)
    extmobusersquantity = len(extmobusers_list)
    print()
    print("Extension Mobility Device and User Data")
    print("=======================================")
    print("The number of users with Extension Mobility profiles allocated is " + str(extmobusersquantity))

    # Get list of devices enabled for Extension Mobility
    extmobdevices_list = axlgetextmobdevices(axlgetcookiesresult)
    extmobdevicesquantity = len(extmobdevices_list)
    print("The number devices enabled for Extension Mobility is " + str(extmobdevicesquantity))

    # Ask user whether to allocate owners to Extension Mobility devices
    allocatedeviceowners = None
    while allocatedeviceowners not in ("yes", "no"):
        allocatedeviceowners = input("Allocate owners to Extension Mobility devices? (yes/no): ")
        if allocatedeviceowners == "yes":
            pass
        elif allocatedeviceowners == "no":
            print()
            sys.exit("Script has terminated - Goodbye!")
        else:
            print("Please enter yes or no.")

    if extmobdevicesquantity <= extmobusersquantity:
        updatetotalruns = extmobdevicesquantity
    else:
        updatetotalruns = extmobusersquantity

    updatecounter = 0

    while updatecounter < updatetotalruns:
        updatedevicepkid = extmobdevices_list[updatecounter]
        updatedevicefkenduser = extmobusers_list[updatecounter]
        allocatedeviceownerresult = axlallocatedeviceowner(axlgetcookiesresult, updatedevicepkid,
                                                              updatedevicefkenduser)
        updatecounter = updatecounter + 1
        print("Device " + str(updatecounter) + " updated.")

########################################################################################################################

if __name__ == "__main__":
    main()

```
