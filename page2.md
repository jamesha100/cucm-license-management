## Article 2 - Basic SQL Commands to Monitor and Update Licensing

Using the CUCM administration web interface is ok for making small numbers of changes but it is not ideal for performing bulk operations such as updating device ownership. CUCM does provide the Bulk Administration Tool (BAT) which can be useful but often it is easier to make changes to the CUCM database by using SQL commands.
These can be executed directly by connecting to the CUCM console interface using SSH or they can be executed using the SOAP based AXL API.
This article provides an overview of using the CLI to execute commands with a special focus on those pertaining to ownership of devices.

- [Article 1 - Introduction and CUCM Licensing Overview.](https://jamesha100.github.io/cucm-license-management/page1)
- Article 2 - Basic SQL Commands to Monitor and Update Licensing.

### Accessing the CUCM CLI

The CUCM CLI may be accessed using an SSH client such as [Putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html).

Connect to the CUCM cluster publisher via SSH then enter the platform username and password - note that these are different from the credentials used to log into the CUCM administration interface. The snippet below shows a typical CLI output for a succesful login.

```
Command Line Interface is starting up, please wait ...

   Welcome to the Platform Command Line Interface

VMware Installation:
        2 vCPU: Intel(R) Xeon(R) CPU E5-2630 v3 @ 2.40GHz
        Disk 1: 80GB, Partitions aligned
        6144 Mbytes RAM

admin:
```
### CUCM Database Overview
CUCM uses an IBM Informix database to hold configuration data. This database is not directly accessible using technologies such as ODBC. The [CUCM 12.5 Data Dictionary](https://developer.cisco.com/docs/axl/#!12-5-cucm-data-dictionary) documents the tables, fields etc. that comprise the CUCM configuration database.

The key tables which will be used in this series of articles are:
- **device** - this table contains details of every device in the system. This includes devices such as SIP trunks, conference bridges, music on hold servers and Extension Mobility profiles as well as collaboration endpoints.
- **enduser** - this table contains details of end users configured on the system. It contains both local users and those synchronised from an LDAP directory.

### Running SQL Commands on the CUCM CLI

SQL commands may be entered on the CUCM CLI by entering `run sql` followed by the SQL statement (SELECT, INSERT, UPDATE, DELETE). For example, `run sql select name,description from device` will list the name and description of all records in the device table as shown below.

```
admin:run sql select name,description from device
name                                               description
================================================== ============================================================
MTP_Pub                                            MTP on Publisher
CFB_Pub                                            Software Conference Bridge on Publisher
ANN_Pub                                            Annunciator on Publisher
MOH_Pub                                            MOH Server on Publisher
Sales                                              Sales
Technical                                          Tech Hunt List
UCCX_889301                                        Technical-1
UCCX_889302                                        Technical-1
UCCX_889303                                        Technical-1
UCCX_889304                                        Technical-1
CUPS-SIP-Trunk                                     IM and Presence
VCS_Trunk                                          Trunk to Cisco VCS service
PROD-CLUSTER                                       SIP Trunk to Production Cluster
TCTJHAWKINS                                        James Hawkins Jabber for iPhone
TABJHAWKINS                                        James Hawkins Jabber for iPad
SEP07872776AB23                                    Extension Mobility 1
SEPA456C561D234                                    Extension Mobility 2
SEP0786AC561274                                    Extension Mobility 3
JHAWKINS-UDP                                       James Hawkins Extension Mobility
```

### Filtering SQL Results
The *device* table contains records for devices other than collboration endpoints. For the purposes of configuring licensing we are not interested in any non endpoint records so we need a query that filters out non-desired results. Luckily the device table includes a field *tkclass* that can be used to implement this filtering. *tkclass* contains numerical values from the *enum* field of the *typeclass* as shown below.
```
admin:run sql select enum,name from typeclass
enum name
==== ===================================
1    Phone
2    Gateway
4    Conference Bridge
5    Media Termination Point
7    Route List
8    Voice Mail
10   CTI Route Point
12   Music On Hold
13   Simulation
14   Pilot
15   GateKeeper
16   Add-on modules
17   Hidden Phone
18   Trunk
19   Tone Announcement Player
20   Remote Destination Profile
248  EMCC Base Phone Template
249  EMCC Base Phone
250  Remote Destination Profile Template
251  Gateway Template
252  UDP Template
253  Phone Template
254  Device Profile
255  Invalid
301  Interactive Voice Response
```
Updating the query to only return results where the value of *tkclass* is equal to "1"  returns only "phones" (this term encompasses video conferencing devices and softphones as well as traditional hardware phones). 
```
admin:run sql select name,description from device where tkclass = '1'
name                                               description
================================================== ============================================================
Technical                                          Tech Hunt List
UCCX_889301                                        Technical-1
UCCX_889302                                        Technical-1
UCCX_889303                                        Technical-1
UCCX_889304                                        Technical-1
TCTJHAWKINS                                        James Hawkins Jabber for iPhone
TABJHAWKINS                                        James Hawkins Jabber for iPad
SEP07872776AB23                                    Extension Mobility 1
SEPA456C561D234                                    Extension Mobility 2
SEP0786AC561274                                    Extension Mobility 3
```



