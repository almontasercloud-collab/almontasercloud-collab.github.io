---
title: Dot1x-PEAP Machine and User AAA
date: 2026-07-28 16:25:00 +0300
categories: [Labs, Cisco ISE]
tags: [networking, Network Engineering, 802.1x, MAB, security, cisco ISE, MSCHAPv2, EAP,  Network Access Control, AAA, Network Access Control]
image:
  path: /assets/img/post_covers/MAB_to_PEAP.png
pin: false
---

# Introduction

This post defines a set of network authentication protocols and demonstrates a scenario where an endpoint initially connects using MAC Authentication Bypass (MAB) and is later authenticated using PEAP with MSCHAPv2.

The setup uses Cisco ISE as the RADIUS server, Active Directory for user authentication, and dynamic VLAN assignment based on the authorization result. The goal is to understand how these authentication methods work together and how the network handles an endpoint as it moves from MAB to user-based authentication.


## Protocols... always! 

**802.1X:** The framework that controls network access by requiring an endpoint to authenticate before gaining access to the network. It defines the roles of the supplicant (endpoint), authenticator (switch), and authentication server (ISE), and uses EAP to carry authentication messages between them.
Think of it as the access control mechanism that decides when an endpoint is allowed onto the network. The switch communicates with ISE through RADIUS, while the endpoint and switch exchange EAP messages over EAPoL.

Simply put, 802.1X is the gatekeeper that makes sure an endpoint authenticates before getting network access.

**PEAP (Protected Extensible Authentication Protocol):** PEAP is an EAP authentication method that establishes a protected TLS tunnel between the supplicant and the authentication server. The tunnel protects the inner authentication exchange from being exposed on the network.

**MAB:** The mechanism that lets a switch authenticate an endpoint using its MAC address instead of a username or certificate. The switch takes the MAC address, sends it to RADIUS, and asks ISE whether the endpoint is allowed on the network.


**RADIUS:** The protocol that carries authentication and authorization requests between the switch and ISE. It’s the conversation channel: the switch asks ISE, “Can this endpoint get in?” and ISE answers with a yes/no, plus any additional attributes such as VLAN assignment. RADIUS provides the IP-based communication channel between the switch and ISE, allowing the network to exchange authentication, authorization, and accounting information beyond the original EAP conversation.

Why do we need RADIUS? Why not just exchange EAP messages directly? A valid question to ask yourself. If EAP already provides a way to exchange authentication messages between the endpoint and the switch, why do we need another protocol like RADIUS?

The answer is that authentication is only part of the story. We also need a way to carry additional context, make authorization decisions, and communicate the final result back to the network device.

And this is where RADIUS comes in.



## Lab Topology and Components

Before walking through the authentication flow, let's establish the lab environment. The topology is intentionally simple: a single endpoint connecting through a Layer 2 access switch, which uplinks to a core switch and then to the identity and directory services.

### Topology Diagram

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/Diagram.png)

### Key Roles 

#### Location2-EP-1 (supplicant)
As shown in the diagram, Location2-EP-1 plays the supplicant role. The endpoint is joined to the `montaser.local` Active Directory domain, and its supplicant configuration — including the PEAP/MSCHAPv2 settings, trusted CA, and authentication mode — is pushed automatically through a Group Policy Object (GPO).
 
#### Location2_Switch (Authenticator)
The access port `Gi0/1` connects to the endpoint, while the uplink `Gi0/0` connects to Core-Switch.

The switch runs an authentication order of mab dot1x, meaning MAB is attempted first. It forwards RADIUS requests to ISE and applies the authorization result, including any VLAN assignment returned by ISE.

#### Core-Switch (Transit)
The Core-Switch provides the Layer 3 path between the access switch and ISE. It simply routes the RADIUS traffic between them; no authentication or authorization decisions are made here.

#### ISE (RADIUS / Policy)
ISE acts as the RADIUS server and policy engine. It receives MAB requests, identifies the endpoint, and returns the initial authorization result.

Later, when the endpoint authenticates using PEAP, ISE handles the authentication process, validates the MSCHAPv2 credentials against Active Directory, and returns the appropriate authorization result — in this case, `VLAN 20`.

ISE is joined to the `montaser.local` domain.

#### AD (Identity Store)
Active Directory stores the user accounts used to authenticate inside the PEAP tunnel. It also hosts the internal Certificate Authority (CA), which issued the EAP certificate used by ISE.

## Configure the Supplicant (Location2-EP-1)

So basically, Location2-EP-1 is a Windows 11 workstation joined to the montaser.local Active Directory domain.

For convenience, you can push the supplicant service and network profile configuration from the domain controller using a Group Policy Object (GPO). This keeps the lab close to how things are handled in real deployments, where you might have tens or hundreds of workstations to configure.

Instead of manually configuring every endpoint, you can let the domain handle it.

First, let's verify that the workstation is properly joined to the Active Directory domain. Navigate to `Control Panel > System and Security > System`

> **Note:** You may need to change the default system-generated hostname and join the PC to your existing Active Directory domain.
{: .prompt-tip }

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/Endpoint_loc2.png)

Computer name , Domain, and workgroup settings section confirms that Location2-EP-1 has joined `montaser.local` domain, at this point you can jump directly to the domain controller to create configuration GPOs.

## Configuring Active Directory Users, Computers and GPOs

The main purpose of this section is to automatically enable 802.1x and configure supplicant for domain-joined Endpoints plus trusting your root CA for authentication:

1- Run `dsa.msc` to open Active Directory Users and Computers.

2- Create a new Organizational Unit (Optional) name it `Location_2_Wired_Workstations`

3- Add Location2-EP-1 to the newly created Organizational Unit

4- Create A new security group and add the workstation to that group.

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/users&computers.png)

5- Run `gpmc.msc` to open Group Policy Management console.
6- Under `Location_2_Wired_Workstations` OU create a new policy object and name it `WiredAuto-Config-PEAP-GPO`

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/new_gpo.png)

7- Right-click on `WiredAuto-Config-PEAP-GPO` and select Edit to start editing the GPO.

8- Under `Computer Configuration > Windows Settings > Security Settings > System Services` locate the Wired AutoConfig service and set it to `Automatic`

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/Wired_AutoConfig.png)

9- `Computer Configuration > Windows Settings > Security Settings > Wired Network (IEEE 802.3) policies` right-click and Create new ... name it `Wired-Endpoints-802.1x-Policy` 

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/supplicant_policy.png)

10- Configure the native supplicant Policy as follows:

> **Note:** Change trusted authentication server name and CA paramaters according to your setup.
{: .prompt-tip }

Enable use of IEEE 802.1X authentication for network access: `Enabled`

Select a network authentication method: Microsoft: `Protected EAP (PEAP)`

Connect to these servers: `ise.montaser.local`

Trusted Root Certification Authorities: `montaser-LAB-DC01-CA`

Select Authentication Method: Secured password `EAP-MSCHAP v2`

Automatically use my Windows logon name and password (and domain if any): `Enabled`

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/supplicant_conf.png) 

11- Save all and Update the policy in Location2-EP-1 using the following command: 

```bash 
gpupdate /force
```
![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/update_success.png)

12- Verify Location2-EP-1 802.1x is enabled and configured according to The GPO defined earlier:

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/verify_supplicant.png)

At this point you are ready to start configuring the Network Access Device (Location2_Switch)

## Configure the Authenticator (Location2_Switch)

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
 key Flora@123

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
 authentication order mab dot1x
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

### Device parameters

Configure the following:

* **Name:** Location2_Switch
* **IP Address:** The IP address used by the switch to communicate with ISE.
* **Device Type:** Select the appropriate network device type for your environment.
* **Network Device Group:** Assign the device to the relevant group if required.

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/ISE_NAD2.png)

### Configure RADIUS parameters:

### RADIUS parameters

Enable RADIUS authentication and configure the shared secret.

The shared secret must match the key configured on the switch under the RADIUS server definition.

In this lab, the switch uses ISE as its RADIUS server for authentication, authorization, and accounting.

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/ISE_NAD3.png)

### Integrate with AD and Enable MAR

Cisco ISE uses Active Directory as the identity source for validating the credentials submitted by the Windows supplicant.

In this lab, ISE is joined to the `montaser.local` Active Directory domain. This allows ISE to authenticate domain users and computer accounts through the configured AD integration.

Navigate to:`Administration > Identity Management > External Identity Sources > Active Directory`

**Join ISE to the domain:**

If ISE is not already joined to the domain, configure the Active Directory join point using the domain name and an account with sufficient permissions to join the ISE node to the domain.

After joining the domain, verify that the connection is successful.

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/AD_Ext_Src.png)

**Enable MAR:**

**Machine Access Restriction (MAR)** is used to support the traditional PEAP machine-before-user authentication workflow.

Enable the machine authentication and MAR options under the Active Directory join-point Advanced configuration.

The purpose of MAR is to allow ISE to determine whether a machine has previously authenticated successfully before authorizing a subsequent user authentication.

This is important when the requirement is that a user must not receive normal network access unless the endpoint has already authenticated using its machine account.

The machine authentication and user authentication are separate PEAP authentication sessions. MAR allows ISE to correlate the machine authentication with the later user authentication for authorization purposes.

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/enable_MAR.png)

> Note: MAR is a traditional PEAP machine-authentication mechanism. It is not EAP chaining. TEAP uses a different authentication model and should be evaluated separately.
pro
{: .prompt-warning }

### Create the Policy Set:

The policy set defines how ISE handles authentication requests received from the wired access switch.

Navigate to:

`Policy > Policy Sets`

Create a new policy set for the Location2 wired authentication scenario.

Policy Set configuration

Name:`Location2_Wired`

Condition:
For example, the policy set can match the network device group containing device type `Wired Devices` and Device location `Locaction2`.

The policy set should be placed above the default policy set so that the requests from this lab are processed by the intended authentication and authorization rules.

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/policy_set1.png)

After creating the policy set, open it to configure the Authentication Policy and Authorization Policy.

### Configure Authentication policy:

The Authentication Policy determines how ISE validates the credentials submitted by the endpoint.

For this lab, the authentication method is PEAP with EAP-MSCHAPv2, and Active Directory is used as the identity source.

Create or select an Allowed Protocols profile that permits:

- PEAP
- EAP-MSCHAPv2 as the inner authentication method

Authentication rule

Configure the wired 802.1X authentication rule with the following settings:

|Setting	| Value
Condition |	Wired 802.1X authentication
Allowed Protocols |	Montaser_AD_Server (Wich is an ISS that performs lookup against Active directory)

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/policy_set2.png)

The authentication result is then passed to the Authorization Policy, where ISE determines the appropriate network access level.

### Location2_Vlan_Assign Authorization Profile:

The Location2_Vlan_Assign authorization profile is used to assign the endpoint to the VLAN associated with the successful authentication result.

Navigate to: `Policy > Policy Elements > Results > Authorization > Authorization Profiles`

Create or select the authorization profile used by the Location2 wired authentication policy.

VLAN assignment

Configure the VLAN assignment attributes required by the switch.

For this lab, the successful PEAP authentication result assigns the endpoint to `VLAN 20`.

The switch receives the authorization attributes from ISE through RADIUS and applies the VLAN assignment to the authenticated session.

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/Location2_vlan_assign.png)

### PreUserAuth_Access Authorization Profile:

The `PreUserAuth_Access` authorization profile is intended for the initial access state before the endpoint completes user authentication.

In this lab, the endpoint first connects using PEAP. ISE identifies the endpoint using its `host/endpointname$` username format and returns the initial authorization result.

The purpose of this profile is to provide the endpoint with the access required during the pre-user-authentication stage, while preventing it from receiving the same access level as a fully authenticated user.

Depending on the intended lab design, this profile can be used to assign a restricted VLAN or apply a downloadable ACL (dACL) that limits the traffic allowed before user authentication.

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/PreUserAuth_Access.png)

**Permit_AD_service dACL:**

The dACL should permit only the required traffic to the appropriate infrastructure servers, rather than allowing unrestricted network access.

For this lab, the required services may include:

DNS resolution.
DHCP, if the endpoint requires an IP address.

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/Permit_AD_Service_dACL.png)

### Permit_Internal_Access Authorization Profile:

After succesful user and machine authentication you can permit a specific users group to accesss a specific subnet to test authoirization functionallity, in this lab users in `Corporate Users` AD Group will be permitied to only communicate with hosts in `172.16.2.0/24` subnet.

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/Permit_Internal_Access.png)

**Internal Only dACL:**

The Internal Only downloadable ACL is retured to restrict the communication to `172.16.2.0/24` network resources.

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/Internal_Only_dACL.png)

### Configure Authorization policy:

The authorization policy processes requests sequentially from top to bottom. It enforces a dual-factor validation logic, verifying the user’s Active Directory group membership while strictly checking that the underlying machine successfully passed its own computer-phase authentication using the Network Access:WasMachineAuthenticated attribute.

![CMD as Administrator](/assets/img/posts_photos/MAB_PEAP/policy_set3.png)

**PEAP_Machine_Auth:**

Conditions: `Wired_802.1X` AND `Montaser_AD_Server:ExternalGroups EQUALS montaser.local/.../Location_2_Wired_Workstations` 

Results: `Location2_Vlan_Assign`, `PreUserAuth_Access`

Explanation: Matches a domain-joined computer at boot-up or the Windows lock screen. It validates the machine account against Active Directory and populates Cisco ISE's internal cache for that MAC address.

**802.1x_User_Authr_1:**

Conditions: `Wired_802.1X` AND `ExternalGroups EQUALS .../Network Administrators AND WasMachineAuthenticated EQUALS True`

Results: `Location2_Vlan_Assign`, `PermitAccess`

Explanation: Grants elevated network access to network administrators (Full Access), provided they log in from a corporate-managed machine that has successfully passed the machine phase.

**802.1x_User_Authr_2:**

Conditions: `Wired_802.1X` AND `ExternalGroups EQUALS .../ISE-CORP-USERS AND WasMachineAuthenticated EQUALS True`

Results: `Location2_Vlan_Assign`, `Permit_Internal_Access`

Explanation: Grants general internal network access to standard corporate users (`172.16.2.0/24` subnet only). It requires the underlying endpoint to be corporate-owned and registered in the active session cache.

At this stage AAA configuration for Location2 wired Devices is completed, tests and verifications can be made to confirm the intended behaviour.
## Testing AAA
## Conclusion
Implementing sequential machine-and-user authorization rules provides a highly secure approach to endpoint control without the administrative overhead of Machine Access Restrictions (MAR). By leveraging the native Network Access:WasMachineAuthenticated attribute, Cisco ISE effectively creates a dual-factor validation requirement: a user must not only possess valid corporate credentials but must also operate from a managed, Active Directory-joined asset.

This simple policy modification acts as a strong defense against rogue or unmanaged personal devices attempting to access your internal networks. To ensure this configuration functions seamlessly, verify that your client endpoints are pushed via Group Policy to use "User or computer authentication," and maintain a single Policy Service Node (PSN) or strict session persistence on your network load balancers to preserve ISE's internal authentication cache.

However, this PEAP-based caching approach has inherent limitations. Because ISE tracks the machine status using temporary memory caches per Policy Service Node (PSN), it is highly vulnerable to session drops during Windows user logons, relies on strict load-balancer persistence, and can fail if the machine cache expires before the user authenticates. To completely eliminate these caching and timing dependencies, organizations should transition to TEAP (Tunnel Extensible Authentication Protocol), which builds a single secure tunnel to authenticate both the machine and user simultaneously, natively providing reliable EAP-chaining without infrastructure workarounds.