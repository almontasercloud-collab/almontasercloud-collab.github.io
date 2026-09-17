---
title: PEAP & EAP-TLS — Machine and User AAA
date: 2026-08-07 12:25:00 +0300
categories: [Labs, Cisco ISE]
tags: [networking, Network Engineering, 802.1x, MAB, security, cisco ISE, MSCHAPv2, EAP, EAP-TLS,  Network Access Control, AAA, Network Access Control]
image:
  path: /assets/img/post_covers/PEAP_&_EAP_TLS.png
pin: false
---

As the previous [**PEAP & MSCHAPv2 — Machine and User AAA**](https://almontaserbabiker.com/posts/PEAP_&_MSCHAPv2/) lab article concluded, username and password authentication is not the most secure way to authenticate users or machines, especially given the common vulnerabilities inherent in credential-based methods. To address these persistent security gaps, this article explores a significantly more robust alternative: **PEAP with EAP-TLS**. 

By combining the encrypted tunneling of Protected Extensible Authentication Protocol (PEAP) with the rigorous, certificate-based mutual authentication of Extensible Authentication Protocol-Transport Layer Security (EAP-TLS), organizations can eliminate password-based vectors entirely. We will dive into how this architecture functions, why it mitigates traditional credential risks, and the practical steps required for its deployment.

## Protocols and Protocols! 

Before jumping into the configuration, review the protocols defined under [**PEAP & MSCHAPv2**](https://almontaserbabiker.com/posts/PEAP_&_MSCHAPv2/#protocols-always) lab article. The same exact set of protocols will be used in this lab while replacing MSCHAPv2 with to EAP-TLS as the inner method for PEAP.

**EAP-TLS** (Extensible Authentication Protocol-Transport Layer Security): is a high-security network access method that uses digital certificates on both the client device and the authentication server to verify identities instead of relying on traditional usernames and passwords.

**EAP-TLS Features:**

* **Mutual Certificate Authentication:** Eliminates passwords entirely by requiring cryptographic digital certificates on both the client device and the server to verify identities.
* **Cryptographic Handshake:** Uses standard TLS protocols to negotiate cipher suites and verify private key ownership.
* **Dynamic Key Generation:** Automatically generates a unique Master Session Key (MSK) for every session, creating individual WPA2/WPA3 encryption keys to protect all subsequent network traffic.
* **Immunity to Credential Theft:** Completely prevents brute-force, dictionary, and phishing attacks because network access relies strictly on non-shareable, machine-bound cryptographic keys rather than human-inputted text.

### Topology Diagram

Before walking through the authentication flow, let's establish the lab environment. The topology is intentionally simple: a single endpoint connecting through a Layer 2 access switch, which uplinks to a core switch and then to the identity and directory services.

![CMD as Administrator](/assets/img/posts_photos/PEAP_EAP_TLS/Diagram.png)

### Key Roles 

* **Location2-EP-1:** (supplicant) As shown in the diagram, Location2-EP-1 plays the supplicant role. The endpoint is joined to the `montaser.local` Active Directory domain, and its supplicant configuration—including the PEAP/MSCHAPv2 settings, trusted CA, and authentication mode—is pushed automatically through a Group Policy Object (GPO).
 
* **Location2_Switch:** (Authenticator)
The access port `Gi0/1` connects to the endpoint, while the uplink `Gi0/0` connects to Core-Switch. The switch runs an authentication order of dot1x mab, meaning dot1x is attempted first. It forwards RADIUS requests to ISE and applies the authorization result, including any VLAN assignment returned by ISE.

* **Core-Switch:** (Transit)
The Core-Switch provides the Layer 3 path between the access switch and ISE. It simply routes the RADIUS traffic between them; no authentication or authorization decisions are made here.

* **ISE:** (RADIUS / Policy)
ISE acts as the RADIUS server and policy engine. It identifies the endpoint, and returns the initial authorization result. Later, when the endpoint authenticates using PEAP, ISE handles the authentication process, validates the MSCHAPv2 credentials against Active Directory, and returns the appropriate authorization result.

> This article assumes that Cisco ISE is already joined to the `montaser.local` Active Directory domain.
{: .prompt-warning }

* **AD:** (Identity Store)
Active Directory stores the user accounts used to authenticate inside the PEAP tunnel. It also hosts the internal Certificate Authority (CA), which issued the EAP certificate used by ISE.

## Configure the Supplicant (Location2-EP-1)

Basically, Location2-EP-1 is a Windows 11 workstation joined to the montaser.local Active Directory domain.

For convenience, you can push the supplicant service and network profile configuration from the domain controller using a Group Policy Object (GPO). This keeps the lab close to how things are handled in real deployments, where you might have tens or hundreds of workstations to configure.

Instead of manually configuring every endpoint, you can let the domain handle it.

First, let's verify that the workstation is properly joined to the Active Directory domain. Navigate to `Control Panel > System and Security > System`

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/Endpoint_loc2.png)

Computer name , Domain, and workgroup settings section confirms that Location2-EP-1 has joined `montaser.local` domain, at this point you can jump directly to the domain controller to create configuration GPOs.

## Overview of Active Directory Configuration

In this lab, Active Directory provides the certificate auto-enrollment and Group Policy services required for 802.1X authentication. The domain-joined machines receive the necessary supplicant configuration and trusted root CA through Group Policy, while certificate templates and auto-enrollment policies handle the provisioning of machine and user certificates for EAP-TLS authentication. The following sections focus on the specific Active Directory configurations used in this lab and how they support the authentication flow.

> **Note:** This guide bypasses the initial deployment steps for certificate auto-enrollment and client supplicant provisioning, assuming these baseline services are already fully operational. Instead, the following key points are highlighted to outline exactly how the Active Directory infrastructure supports and interacts with our authentication flow inside this lab environment.
{: .prompt-tip }

### Users and User Groups:

The lab environment includes the `LOCATION2-EP-1` computer account, which represents the domain-joined endpoint used for 802.1X authentication. The `Location_2_Wired_Workstations` security group is used to manage workstation-specific settings, such as machine certificate auto-enrollment, and to restrict the Cisco ISE authentication policy to the intended endpoints. The `Location2_Net_Admins` and `location2_users` security groups, along with their respective user accounts, are created to test different authorization profiles in Cisco ISE. User certificate auto-enrollment is also configured to support the required authentication methods and demonstrate how Active Directory provisions user identities for network access.


![CMD as Administrator](/assets/img/posts_photos/PEAP_EAP_TLS/users_and_user_groups.png)

### GPOs:

Under Group Policy Management, a dedicated Organizational Unit (OU) named **Location_2_Wired_Workstations** is created within the `montaser.local` domain to manage the workstations used in this lab. The OU has three linked Group Policy Objects (GPOs): `Certificate_Enrollment` for certificate auto-enrollment, `Endpoint-Net-Admins-GPO` for the required endpoint settings, and `WiredAuto-Config-PEAP-GPO` for deploying the wired 802.1X supplicant configuration. This allows the required policies to be applied to workstations placed in the OU, rather than configuring each endpoint individually.

![CMD as Administrator](/assets/img/posts_photos/PEAP_EAP_TLS/GPOs.png)

For Example, Check the **`WiredAuto-Config-PEAP-GPO`** configuration: 

This GPO enables the `Wired-AutoConfig` service, which is the native Windows 802.1X supplicant, and pushes the PEAP and EAP-TLS configuration to all endpoints within the `Location_2_Wired_Workstations` OU.

**Automatically enable 802.1x supplicant on domain-joined Endpints:**

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/Wired_AutoConfig.png)

**Automatically Configure 802.1x supplicant User and machine authentication using PEAP and EAP-TLS as inner method:**

![CMD as Administrator](/assets/img/posts_photos/PEAP_EAP_TLS/Supplicant_Conf.png)


### Certificate Templates: 

Another aspect worth looking at is the certificate templates created for the automatic enrollment of user and machine certificates. Active Directory uses these templates to provision certificates for the users and computers in Location 2.

![CMD as Administrator](/assets/img/posts_photos/PEAP_EAP_TLS/Cert_templates.png)

### Automatic Client Configuration:

Once Active Directory is properly configured and the relevant policies are applied to domain-joined endpoints to provision client certificates and supplicant settings, you can see that users are automatically equipped with their user certificates, while the computer is provisioned with its machine certificate and supplicant configuration.
 
![CMD as Administrator](/assets/img/posts_photos/PEAP_EAP_TLS/policy_applied.png)

### Check the issued User certificate details:

This section highlights an important detail that will affect the ISE configuration later. If you open the Location 2 NetAdmin certificate, you can see that the account username is embedded in the certificate's `Subject Alternative Name (SAN)` field.

This is important because, later in the configuration, you will need to tell ISE which field within the certificate it should use to identify the user and perform the directory lookup. This is configured in the Certificate Authentication Profile (CAP), which will be explained in detail later in this article.

![CMD as Administrator](/assets/img/posts_photos/PEAP_EAP_TLS/SAN.png)

At this stage, you are ready to proceed with the configuration of the remaining entities: the authenticator, which in this lab is a Cisco virtual switch, and the authentication server, Cisco ISE.

## Configure the Authenticator (Location2_Switch)

The configuration in this section is intentionally minimal, focusing only on the essential settings required to enable 802.1X authentication

### AAA Global Configuration:

```bash 
aaa new-model
aaa local authentication ISE-Auth authorization ISE-Authz


aaa authentication dot1x default group ISE-SG
aaa authorization network default group ISE-SG
aaa accounting dot1x default start-stop group ISE-SG

aaa group server radius ISE-SG
 server name ise.montaser.local
 deadtime 5

radius server ise.montaser.local
 address ipv4 172.16.2.101 auth-port 1812 acct-port 1813
 key *****

aaa server radius dynamic-author
 client 172.16.2.101 server-key *****

dot1x system-auth-control
aaa session-id common

interface Vlan100
 description RADIUS-L3
 ip address 10.1.100.3 255.255.255.0

ip route 172.16.0.0 255.255.0.0 10.1.100.1

```

### Access Interface Configuration:

```bash
interface GigabitEthernet0/1
Description ### Location2-EP-1 ###
 switchport mode access
 negotiation auto
 authentication event fail action next-method
 authentication order dot1x mab
 authentication priority dot1x mab
 authentication port-control auto
 authentication periodic
 mab
 dot1x pae authenticator
 spanning-tree portfast edge
```

## Configure The Authentication Server (ISE)

## Adding `Location2_Switch` as a Network Device

Before ISE can process authentication requests from the access switch, the switch must be registered as a Network Device.

Navigate to:

`Administration > Network Resources > Network Devices`

Click **Add** and enter the device details.

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/ISE_NAD.png)

### Configure the device parameters:

Configure the following device parameters:

* **Name:** Location2_Switch
* **IP Address:** The IP address used by the switch to communicate with ISE.
* **Device Type:** Select the appropriate network device type for your environment.
* **Network Device Group:** Assign the device to the relevant group if required.

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/ISE_NAD2.png)

### Configure RADIUS parameters:

Enable RADIUS authentication and configure the shared secret.

The shared secret must match the key configured on the switch under the RADIUS server definition. In this lab, the switch uses ISE as its RADIUS server for authentication, authorization, and accounting.

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/ISE_NAD3.png)

### Integrate with AD and Enable MAR

Cisco ISE uses Active Directory as the identity source for validating the credentials submitted by the Windows supplicant. In this lab, ISE is joined to the `montaser.local` Active Directory domain. This allows ISE to authenticate domain users and computer accounts through the configured AD integration.

Navigate to:`Administration > Identity Management > External Identity Sources > Active Directory`

**Join ISE to the domain:**

If ISE is not already joined to the domain, configure the Active Directory join point using the domain name and an account with sufficient permissions to join the ISE node to the domain. After joining the domain, verify that the connection is successful, then retrieve the required AD groups. In this lab, the `Network Administrators` and `Corporate Users` groups are imported into ISE.

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/AD_Ext_Src.png)

**Enable MAR:**

**Machine Access Restriction (MAR)** is used to support the traditional PEAP machine-before-user authentication workflow.

Enable the machine authentication and MAR options under the Active Directory join-point Advanced configuration.

The purpose of MAR is to allow ISE to determine whether a machine has previously authenticated successfully before authorizing a subsequent user authentication. This is important when the requirement is that a user must not receive normal network access unless the endpoint has already authenticated using its machine account.

The machine authentication and user authentication are separate PEAP authentication sessions. MAR allows ISE to correlate the machine authentication with the later user authentication for authorization purposes.

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/enable_MAR.png)

> Note: MAR is a traditional mechanism used to support machine authentication with PEAP. Newer outer methods, such as TEAP, were developed to reduce dependency on MAR. TEAP uses a different authentication model and will be evaluated separately in a dedicated lab.
{: .prompt-warning }

### Create the certificate authentication profile:(CAP) 

As mentioned earlier, ISE needs to know which field in the client certificate it should use to extract the user's identity and perform the corresponding directory lookup. In this lab, the user's account name is embedded in the `Subject Alternative Name` (SAN) field of the certificate.

To configure this, navigate to `Administration > Identity Management > External Identity Sources > Certificate Authentication Profile` and click Add to create a new Certificate Authentication Profile.

![CMD as Administrator](/assets/img/posts_photos/PEAP_EAP_TLS/Create_CAP.png)

Give the profile a descriptive name, such as `EAP-TLS`. Under Principal Name X509 Attribute, select `Subject Alternative Name` and specify the appropriate SAN attribute that contains the user's account name.

Finally, enable Resolve Identity Ambiguity so that ISE can use the extracted identity to resolve the corresponding user account.

![CMD as Administrator](/assets/img/posts_photos/PEAP_EAP_TLS/CAP_Conf.png)

