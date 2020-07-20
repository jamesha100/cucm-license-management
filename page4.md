## Article 4 - Saving Existing Device Ownership Data

Before we start updating any device ownership data it is important to save existing data in a form which may be used to restore the system back to it's pre-change state.

To do this we will use a Python script to submit a query to the CUCM API and store the returned data in a CSV file.

- [Article 1 - Introduction and CUCM Licensing Overview.](https://jamesha100.github.io/cucm-license-management/page1)
- [Article 2 - Basic SQL Commands to Monitor and Update Licensing.](https://jamesha100.github.io/cucm-license-management/page2)
- [Article 3 - Using the CUCM AXL API.](https://jamesha100.github.io/cucm-license-management/page3)
- Article 4 - Saving Existing Device Ownership Data

### SQL Query Used to Retrieve Device Ownership Data

The SQL query we will use to retrieve the device mapping data is shown below.
```
run sql select d.pkid,d.name as devicename, d.description,tm.name as devicemodel,d.fkenduser,eu.userid from device as d inner join typemodel as tm on tm.enum = d.tkmodel inner join enduser as eu on eu.pkid=d.fkenduser where d.tkclass = '1' and not (d.tkmodel = '72' or d.tkmodel = '645')
```
This query will retrieve the following data for any phone device that has an owner assigned.
- Device pkid.
- Device name.
- Device description.
- Device model.
- Device owner (fkenduser) - this maps to the pkid value in the *enduser* table.
- Device owner userid value.

Devices that do not have owners assigned (the fkenduser filed is set to NULL) will not be returned in the query. In reality only the device pkid and device owner values are needed to restore ownership settings. The other fields are collected to make the data readable by humans.

### The Python Script

