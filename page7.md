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
TBC

```
TBC
```
