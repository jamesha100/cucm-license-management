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

### Basic SQL Commands

