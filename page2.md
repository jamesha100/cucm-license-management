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

SQL commands may be entered on the CUCM CLI by entering `run sql` followed by the SQL statement (SELECT, INSERT, UPDATE, DELETE). For example, `run sql select pkid,name,description from device` will list the primary key, name and description of all records in the device table as shown below.

```
admin:run sql select pkid,name,description from device
pkid                                 name                                               description                                               
==================================== ================================================== ============================================================        
6107eb89-c3e2-48c9-8c14-5b92480814dd Auto-registration Template                         #FirstName# #LastName# (#Product# #Protocol#)             
23525453-9d43-4883-b079-0fa909dd860d MTP_Pub                                            MTP on Publisher                                          
f662853d-ea26-4ea4-bc28-6dbf589d0b8e CFB_Pub                                            Software Conference Bridge on Publisher                   
cb57bb48-2195-4b96-9f7d-8208f918f848 ANN_Pub                                            Annunciator on Publisher                                  
8e5e1179-1632-4fe6-977f-2f4a2e09b2ca MOH_Pub                                            MOH Server on Publisher                                                                                  
fdf9378d-7d70-4b28-477b-11d3e5536a28 UCCX_889301                                        Technical-1                                               
a802f3fe-fd3e-94ec-fafb-c451f4bb8b99 UCCX_889302                                        Technical-1                                               
2d7ad949-f76d-5155-8219-db380b027d1b UCCX_889303                                        Technical-1                                               
332be9f4-e7a3-2836-f5a7-f813cf321162 UCCX_889304                                        Technical-1                                                                    
d9884544-b512-affb-f32b-db9d4e861baa VCS_Trunk                                          Trunk to Cisco VCS service                                
6eb51642-2723-8ceb-0431-6d1f70437fb8 CSFJHAWKINS                                        James Hawkins Jabber CSF                                                      
5a5fbb7b-6b34-7a81-332f-e8bf39bb669d TCTJHAWKINS                                        James Hawkins Jabber for iPhone                           
1b7d9e07-f550-ad1a-43a9-cc95e84fa928 TABJHAWKINS                                        James Hawkins Jabber for iPad                             
459dcb10-4f80-6721-97c3-3e30280f0097 SEP07872776AB23                                    Extension Mobility 1                                      
5ca716fd-50f5-59bd-9c35-29b4222462fb SEPA456C561D234                                    Extension Mobility 2                                      
cbbfb41c-3a97-968d-678b-0aee016543ff SEP0786AC561274                                    Extension Mobility 3                                      
e80c5172-a253-5814-5a07-d71e2f3e274a JHAWKINS-UDP                                       James Hawkins Extension Mobility                          
e64a0518-4342-5b5c-cffa-b987bc6dc1d2 SEP389A267CBA34                                    James Hawkins DX80     
```
Note that they primary key value is a *Globally Unique Identifier* or GUID which uniquely identifies the record.
### Selecting Phone Device Information from the *devices* Table
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
admin:run sql select pkid,name,description from device where tkclass = '1'
pkid                                 name                                           description
==================================== ============================================== ==================================================
1acdd19c-2c63-4191-b21f-983f27c88f18 Sample Device Template with TAG usage examples #FirstName# #LastName# (#Product# #Protocol#)
fdf9378d-7d70-4b28-477b-11d3e5536a28 UCCX_889301                                    Technical-1
a802f3fe-fd3e-94ec-fafb-c451f4bb8b99 UCCX_889302                                    Technical-1
2d7ad949-f76d-5155-8219-db380b027d1b UCCX_889303                                    Technical-1
332be9f4-e7a3-2836-f5a7-f813cf321162 UCCX_889304                                    Technical-1
6eb51642-2723-8ceb-0431-6d1f70437fb8 CSFJHAWKINS                                    James Hawkins Jabber CSF
58cfca43-5de2-969f-cd26-fc306e9dab41 SEPB4E9B0B1CE67                                ALD Test Phone
1b7d9e07-f550-ad1a-43a9-cc95e84fa928 TABJHAWKINS                                    James Hawkins Jabber for iPad
459dcb10-4f80-6721-97c3-3e30280f0097 SEP07872776AB23                                Extension Mobility 1
5ca716fd-50f5-59bd-9c35-29b4222462fb SEPA456C561D234                                Extension Mobility 2
cbbfb41c-3a97-968d-678b-0aee016543ff SEP0786AC561274                                Extension Mobility 3
e64a0518-4342-5b5c-cffa-b987bc6dc1d2 SEP389A267CBA34                                James Hawkins DX80
```
This list of devices is better but notice the devices with a name starting UCCX. These are CTI ports used by Cisco Contact Center Express (applications such as attendant consoles also use CTI ports). Also notice the template device on the first line..

It is possible to filter these using the *tkmodel* field in the device table. *tkmodel* contains numerical values from the *enum* field of the *typemodel* as shown below.
```
admin:run sql select enum,name from typemodel order by enum
enum  name
===== ==================================================
1     Cisco 30 SP+
2     Cisco 12 SP+
3     Cisco 12 SP
4     Cisco 12 S
5     Cisco 30 VIP
6     Cisco 7910
7     Cisco 7960
8     Cisco 7940
9     Cisco 7935
10    Cisco VGC Phone
11    Cisco VGC Virtual Phone
12    Cisco ATA 186
15    EMCC Base Phone
20    SCCP Phone
30    Analog Access
40    Digital Access
42    Digital Access+
43    Digital Access WS-X6608
47    Analog Access WS-X6624
48    VGC Gateway
50    Conference Bridge
51    Conference Bridge WS-X6608
52    Cisco IOS Conference Bridge (HDV2)
53    Cisco Conference Bridge (WS-SVC-CMM)
61    H.323 Phone
62    H.323 Gateway
70    Music On Hold
71    Device Pilot
72    CTI Port
73    CTI Route Point
80    Voice Mail Port
83    Cisco IOS Software Media Termination Point (HDV2)
84    Cisco Media Server (WS-SVC-CMM-MS)
85    Cisco Video Conference Bridge (IPVC-35xx)
86    Cisco IOS Heterogeneous Video Conference Bridge
87    Cisco IOS Guaranteed Audio Video Conference Bridge
88    Cisco IOS Homogeneous Video Conference Bridge
90    Route List
100   Load Simulator
110   Media Termination Point
111   Media Termination Point Hardware
112   Cisco IOS Media Termination Point (HDV2)
113   Cisco Media Termination Point (WS-SVC-CMM)
115   Cisco 7941
119   Cisco 7971
120   MGCP Station
121   MGCP Trunk
122   GateKeeper
124   7914 14-Button Line Expansion Module
125   Trunk
126   Tone Announcement Player
131   SIP Trunk
132   SIP Gateway
133   WSM Trunk
134   Remote Destination Profile
227   7915 12-Button Line Expansion Module
228   7915 24-Button Line Expansion Module
229   7916 12-Button Line Expansion Module
230   7916 24-Button Line Expansion Module
232   CKEM 36-Button Line Expansion Module
253   SPA8800
254   Unknown MGCP Gateway
255   Unknown
302   Cisco 7985
307   Cisco 7911
308   Cisco 7961G-GE
309   Cisco 7941G-GE
335   Motorola CN622
336   Third-party SIP Device (Basic)
348   Cisco 7931
358   Cisco Unified Personal Communicator
365   Cisco 7921
369   Cisco 7906
374   Third-party SIP Device (Advanced)
375   Cisco TelePresence
376   Nokia S60
404   Cisco 7962
412   Cisco 3951
431   Cisco 7937
434   Cisco 7942
435   Cisco 7945
436   Cisco 7965
437   Cisco 7975
446   Cisco 3911
468   Cisco Unified Mobile Communicator
478   Cisco TelePresence 1000
479   Cisco TelePresence 3000
480   Cisco TelePresence 3200
481   Cisco TelePresence 500-37
484   Cisco 7925
493   Cisco 9971
495   Cisco 6921
496   Cisco 6941
497   Cisco 6961
503   Cisco Unified Client Services Framework
505   Cisco TelePresence 1300-65
520   Cisco TelePresence 1100
521   Transnova S3
522   BlackBerry MVS VoWifi
537   Cisco 9951
540   Cisco 8961
547   Cisco 6901
548   Cisco 6911
550   Cisco ATA 187
557   Cisco TelePresence 200
558   Cisco TelePresence 400
562   Cisco Dual Mode for iPhone
564   Cisco 6945
575   Cisco Dual Mode for Android
577   Cisco 7926
580   Cisco E20
582   Generic Single Screen Room System
583   Generic Multiple Screen Room System
584   Cisco TelePresence EX90
585   Cisco 8945
586   Cisco 8941
588   Generic Desktop Video Endpoint
590   Cisco TelePresence 500-32
591   Cisco TelePresence 1300-47
592   Cisco 3905
593   Cisco Cius
594   VKEM 36-Button Line Expansion Module
596   Cisco TelePresence TX1310-65
597   Cisco TelePresence MCU
598   Ascom IP-DECT Device
599   Cisco TelePresence Exchange System
604   Cisco TelePresence EX60
606   Cisco TelePresence Codec C90
607   Cisco TelePresence Codec C60
608   Cisco TelePresence Codec C40
609   Cisco TelePresence Quick Set C20
610   Cisco TelePresence Profile 42 (C20)
611   Cisco TelePresence Profile 42 (C60)
612   Cisco TelePresence Profile 52 (C40)
613   Cisco TelePresence Profile 52 (C60)
614   Cisco TelePresence Profile 52 Dual (C60)
615   Cisco TelePresence Profile 65 (C60)
616   Cisco TelePresence Profile 65 Dual (C90)
617   Cisco TelePresence MX200
619   Cisco TelePresence TX9000
620   Cisco TelePresence TX9200
621   Cisco 7821
622   Cisco 7841
623   Cisco 7861
626   Cisco TelePresence SX20
627   Cisco TelePresence MX300
628   IMS-integrated Mobile (Basic)
631   Third-party AS-SIP Endpoint
632   Cisco Cius SP
633   Cisco TelePresence Profile 42 (C40)
634   Cisco VXC 6215
635   CTI Remote Device
640   Usage Profile
642   Carrier-integrated Mobile
645   Universal Device Template
647   Cisco DX650
648   Cisco Unified Communications for RTX
652   Cisco Jabber for Tablet
659   Cisco 8831
681   Cisco ATA 190
682   Cisco TelePresence SX10
683   Cisco 8841
684   Cisco 8851
685   Cisco 8861
688   Cisco TelePresence SX80
689   Cisco TelePresence MX200 G2
690   Cisco TelePresence MX300 G2
20000 Cisco 7905
30002 Cisco 7920
30006 Cisco 7970
30007 Cisco 7912
30008 Cisco 7902
30016 Cisco IP Communicator
30018 Cisco 7961
30019 Cisco 7936
30027 Analog Phone
30028 ISDN BRI Phone
30032 SCCP gateway virtual phone
30035 IP-STE
36041 Cisco TelePresence Conductor
36042 Cisco DX80
36043 Cisco DX70
36049 BEKEM 36-Button Line Expansion Module
36207 Cisco TelePresence MX700
36208 Cisco TelePresence MX800
36210 Cisco TelePresence IX5000
36213 Cisco 7811
36216 Cisco 8821
36217 Cisco 8811
36219 Interactive Voice Response
36224 Cisco 8845
36225 Cisco 8865
36227 Cisco TelePresence MX800 Dual
36232 Cisco 8851NR
36235 Cisco Spark Remote Device
36239 Cisco TelePresence DX80
36241 Cisco TelePresence DX70
36247 Cisco 7832
36248 Cisco 8865NR
36250 Cisco Meeting Server
36251 Cisco Spark Room Kit
36254 Cisco Spark Room 55
36255 Cisco Spark Room Kit Plus
36256 CP-8800-Video 28-Button Key Expansion Module
36257 CP-8800-Audio 28-Button Key Expansion Module
36258 Cisco 8832
36260 Cisco 8832NR
```
Updating the query to add a filter that returns records where *tkmodel* is not equal to "72" or "645" excludes CTI ports and device templates from the returned results.
```
admin:run sql select pkid,name,description from device where tkclass = '1' and not (tkmodel = '72' or tkmodel = '645')
pkid                                 name            description
==================================== =============== ===============================
6eb51642-2723-8ceb-0431-6d1f70437fb8 CSFJHAWKINS     James Hawkins Jabber CSF
5a5fbb7b-6b34-7a81-332f-e8bf39bb669d TCTJHAWKINS     James Hawkins Jabber for iPhone
1b7d9e07-f550-ad1a-43a9-cc95e84fa928 TABJHAWKINS     James Hawkins Jabber for iPad
459dcb10-4f80-6721-97c3-3e30280f0097 SEP07872776AB23 Extension Mobility 1
5ca716fd-50f5-59bd-9c35-29b4222462fb SEPA456C561D234 Extension Mobility 2
cbbfb41c-3a97-968d-678b-0aee016543ff SEP0786AC561274 Extension Mobility 3
e64a0518-4342-5b5c-cffa-b987bc6dc1d2 SEP389A267CBA34 James Hawkins DX80
```
In the query above we used *tkmodel* to exclude unwanted results. It can be useful to display the device model in our query so that we can see which types of devices we are dealing with. The query below does just this by using an "inner join" to pull the *name* field from the *typemodel* table.
```
admin:run sql select d.pkid,d.name as devicename, d.description,tm.name as devicemodel from device as d inner join typemodel as tm on tm.enum = d.tkmodel where d.tkclass = '1' and not (d.tkmodel = '72' or d.tkmodel = '645')

pkid                                 devicename      description                     devicemodel
==================================== =============== =============================== =======================================
6eb51642-2723-8ceb-0431-6d1f70437fb8 CSFJHAWKINS     James Hawkins Jabber CSF        Cisco Unified Client Services Framework
5a5fbb7b-6b34-7a81-332f-e8bf39bb669d TCTJHAWKINS     James Hawkins Jabber for iPhone Cisco Dual Mode for iPhone
1b7d9e07-f550-ad1a-43a9-cc95e84fa928 TABJHAWKINS     James Hawkins Jabber for iPad   Cisco Jabber for Tablet
459dcb10-4f80-6721-97c3-3e30280f0097 SEP07872776AB23 Extension Mobility 1            Cisco 7841
5ca716fd-50f5-59bd-9c35-29b4222462fb SEPA456C561D234 Extension Mobility 2            Cisco 7841
cbbfb41c-3a97-968d-678b-0aee016543ff SEP0786AC561274 Extension Mobility 3            Cisco 7841
e64a0518-4342-5b5c-cffa-b987bc6dc1d2 SEP389A267CBA34 James Hawkins DX80              Cisco DX80
```
The query above lists the phone devices that consume CUCM licenses but further information is needed to view ownership information. This is stored in the *fkenduser* field in the device table. The value of *fkenduser* in the *device* table is set to the primary key value (*pkid*) of the enduser table for the user that is configured as the owner of the device. If no owner is specified the value of *fkenduser* is "NULL". The updated query below shows the fkenduser values for the phone devices configured on the test system.
```
admin:run sql select d.pkid,d.name as devicename, d.description,tm.name as devicemodel,d.fkenduser from device as d inner join typemodel as tm on tm.enum = d.tkmodel where d.tkclass = '1' and not (d.tkmodel = '72' or d.tkmodel = '645')
pkid                                 devicename      description                     devicemodel                             fkenduser            
==================================== =============== =============================== ======================================= ====================================
6eb51642-2723-8ceb-0431-6d1f70437fb8 CSFJHAWKINS     James Hawkins Jabber CSF        Cisco Unified Client Services Framework 8c758cf4-da8c-bd00-53be-a0902f1707e1
5a5fbb7b-6b34-7a81-332f-e8bf39bb669d TCTJHAWKINS     James Hawkins Jabber for iPhone Cisco Dual Mode for iPhone              NULL                 
1b7d9e07-f550-ad1a-43a9-cc95e84fa928 TABJHAWKINS     James Hawkins Jabber for iPad   Cisco Jabber for Tablet                 NULL                 
459dcb10-4f80-6721-97c3-3e30280f0097 SEP07872776AB23 Extension Mobility 1            Cisco 7841                              NULL                 
5ca716fd-50f5-59bd-9c35-29b4222462fb SEPA456C561D234 Extension Mobility 2            Cisco 7841                              NULL                 
cbbfb41c-3a97-968d-678b-0aee016543ff SEP0786AC561274 Extension Mobility 3            Cisco 7841                              NULL                 
e64a0518-4342-5b5c-cffa-b987bc6dc1d2 SEP389A267CBA34 James Hawkins DX80              Cisco DX80                              NULL    
```
At the time this query was run only the *CSFJHAWKINS* had an owner configured. 

The *fkenduser* value in in the form of a database *Globally Unique Identifier* or GUID. This is a unique sixteen bit value which is used as the primary key for the database table. Obviously the GUID is not meaningful to humans - the query on the *enduser* table below shows that the userid *jhawkins* is associated with the fkenduser GUID value from the previous query.

```
admin:run sql select pkid,userid from enduser where pkid = '8c758cf4-da8c-bd00-53be-a0902f1707e1'
pkid                                 userid
==================================== ========
8c758cf4-da8c-bd00-53be-a0902f1707e1 jhawkins
```
It is possible to show the *userid* value in the device query shown above by adding an inner join to the *enduser* table as shown in the query below.
```
admin:run sql select d.pkid,d.name as devicename, d.description,tm.name as devicemodel,d.fkenduser,eu.userid from device as d inner join typemodel as tm on tm.enum = d.tkmodel inner join enduser as eu on eu.pkid=d.fkenduser where d.tkclass = '1' and not (d.tkmodel = '72' or d.tkmodel = '645')
pkid                                 devicename      description              devicemodel                             fkenduser                            userid
==================================== =============== ======================== ======================================= ==================================== ==========
6eb51642-2723-8ceb-0431-6d1f70437fb8 CSFJHAWKINS     James Hawkins Jabber CSF Cisco Unified Client Services Framework 8c758cf4-da8c-bd00-53be-a0902f1707e1 jhawkins
```
Notice that only a single line is returned. This is because SQL inner joins do not work if one of the values selected for the join is "NULL".

### Updating Device Ownership
