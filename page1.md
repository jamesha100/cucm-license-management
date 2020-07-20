## Article 1 - Introduction and CUCM Licensing Overview

Cisco Unified Communications Manager 9 introduced a user based licensing model based upon user ownership of physical and software based endpoints. This model generally works well but problems can be encountered if the ownership of devices is not configured correctly. This series of articles provides an overview of licensing and strategies to ensure that licenses are used efficiently.
These strategies are based upon using SQL database queries to collect and update device ownership. Where multiple database operations are required Python scripts will be used in conjunction with the CUCM AXL API to perform operations.

- Article 1 - Introduction and CUCM Licensing Overview.
- [Article 2 - Basic SQL Commands to Monitor and Update Licensing.](https://jamesha100.github.io/cucm-license-management/page2)
- [Article 3 - Using the CUCM AXL API.](https://jamesha100.github.io/cucm-license-management/page3)
- [Article 4 - Saving Existing Device Ownership Data](https://jamesha100.github.io/cucm-license-management/page4)

**Note that no liability is accepted for any system damage caused as a result of following instructions in this series of articles.**

### CUCM Licensing Overview

CUCM licensing is user based with different license types supporting different device types and numbers. The available license types are listed below.

- UCL Basic - supports **one** "basic" device of the types listed below:
    - Analogue device - this would typically be one port on an analogue telephony adapter or an FXS port on a Cisco gateway.
    - Cisco 3905 IP phone.
    - Cisco 6901 IP phone.
- UCL Essential - supports **one** "essential" device of the types listed below:
    - Cisco 7811 IP phone.
    - Cisco 7821 IP phone.
- UCL Enhanced - supports **one** user device of any type. This includes physical IP phones, softphone/mobile devices such as Jabber and personal video devices such as the Cisco DX80. This license type does **not** support room based video endpoints such as the Cisco WebEx Room 55.
- UCL Enhanced Plus - supports **two** user devices of any type. These devices must be allocated to the same user. A typical example for this license type would be a user who has a physical IP phone for use in the office and Cisco Jabber for Windows installed on their laptop for use whilst working remotely.
- CUWL Standard - supports up to **ten** user devices of any type. These devices must be allocated to the same user. A typical example for this license type would be a user who has a physical IP phone for use in the office, Cisco Jabber for Windows installed on their laptop for use whilst working remotely and Cisco Jabber for iPhone on their mobile phone. CUWL Standard also provides a license for Cisco Unity Connection messaging.
- CUWL Meetings - supports up to **ten** user devices of any type. These devices must be allocated to the same user. A typical example for this license type would be a user who has a physical IP phone for use in the office, Cisco Jabber for Windows installed on their laptop for use whilst working remotely and Cisco Jabber for iPhone on their mobile phone. CUWL Standard also provides a license for Cisco Unity Connection messaging and a personal license for multi-party meeting technology using Cisco Meeting Server or WebEx.
- TelePresence Room - this license type supports **one** Cisco "room" based video conferencing device. These range from the 55" single screen devices, such as the WebEx Room 55, to the three screen immersive TelePresence systems such as the IX9000. This license type also covers codec devices, such as the WebEx Room Kit Pro, which are used with third party screens.

Licenses are allocated when the devices are configured on CUCM and are consumed even if the device is not registered with CUCM.

CUCM has the concept of borrowing licenses from higher tiers if insufficient licenses of the appropriate type are available. For example, if no UCL Enhanced licenses were available and a new 7841 IP phone were configured, CUCM will attempt to use a license of a higher tier (UCL Enhanced Plus if available, then CUWL Standard, then CUWL Meetings) to license the new device. If no higher tier license is available CUCM will enter a license non-compliance state. The CUCM administrator will be informed of this on the CUCM administration login page. The system will continue to operate for a grace period of 60 days during which the administrator should resolve the issue by either purchasing more licenses to cover the overage or deleting devices until a compliant state is reached. After 60 days if the system is still non-compliant it will enter a read-only state where services will continue to operate but no configuration changes are possible. Cisco TAC must be contacted to remove the read-only state.

For more details on CUCM licensing refer to the Cisco document [CUCM Licensing At A Glance](https://www.cisco.com/c/dam/en/us/products/collateral/unified-communications/unified-communications-licensing/C45_523902_11_9_licensing_aag_v5a_1.pdf).

### License Usage

For CUCM licensing to work as designed when using license types that support multiple devices (UCL Enhanced Plus, CUWL Standard or CUWL Meetings) owners must be assigned to devices.

This is done on the CUCM device configuration page as shown below.

![Device Owner ID](/images/ownerid.png)

The *Owner* field must be set to *User* and the appropriate username selected for the *Owner User ID* field.

-If a user is assigned as the owner of just one device then a UCL Enhanced license will be consumed (assuming that the user device does not fall into the UCL Basic or UCL Essential categories).
-If a user is assigned as the owner of two devices then a UCL Enhanced Plus license will be consumed.
-If a user is assigned as the user of between three and ten devices then a CUWL license will be consumed. From the CUCM device licensing perspective there is no difference between CUWL Standard and CUWL Meetings althougb the former seem to be allocated before the latter.

If device ownership is not configured correctly then the wrong license types can be consumed. For example, if a user has a deskphone, a personal video conference unit, Jabber for Windows, Jabber for iPhone and Jabber for iPad then all their devices could be covered by a single CUWL Standard license. If the ownership of these devices is not configured then five UCL Enhanced licenses would be allocated. If the CUCM system in this example only had CUWL Standard licenses available then four of this license type would be consumed - CUCM borrows from the higher tier - which is obviously inefficient in terms of license usage and cost.

### Typical License Issues

This section will detail some typical license issues encountered on production CUCM systems.

#### Devices Not Allocated To Correct Owners

TBC

#### Devices Not Deleted When Users Leave The Organisation

TBC

#### Bogus License Users

Often when a CUCM system enters a non-compliant licensing state administrators will implement a quick "fix" by creating a dummy license user and allocating multiple devices to it. This is particularly effective for systems with CUWL licenses available as each license can support ten devices. Configuring the system in this way is breaking the conditions of the CUCM license agreement which states all devices covered by a particular license should be used by a single individual rather than multiple users. I am unaware of any cases where Cisco have audited customer systems for compliance with these terms but there is the possibility that it could happen so sticking to the rules and avoiding this bogus fix is recommended. 
