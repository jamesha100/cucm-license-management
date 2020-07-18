## Introduction

Cisco Unified Communications Manager 9 introduced a user based licensing model based upon user ownership of physical and software based endpoints. This model generally works well but problems can be encountered if the ownership of devices is not configured correctly. This series of articles provides an overview of licensing and strategies to ensure that licenses are used efficiently.
These strategies are based upon using SQL database queries to collect and updated device ownership.

**Note that no liability is accepted for any system damage caused as a result of following instructions in this series of articles.**

## CUCM Licensing Overview

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

For more details on CUCM licensing refer to the Cisco document [CUCM Licensing At A Glance](https://www.cisco.com/c/dam/en/us/products/collateral/unified-communications/unified-communications-licensing/C45_523902_11_9_licensing_aag_v5a_1.pdf).

## Typical License Issues

For CUCM licensing to work as designed devices must be assigned owners.

![Device Owner ID](/images/ownerid.png)
