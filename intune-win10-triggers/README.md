
# Optimising Azure VPN P2S costs using Intune and Windows 10 VPN autotrigger

#  Introduction

## Scenario

- Contoso are using the **Azure VPN Client** plugin across their Windows 10 client estate 
- They are using certificate authentication 
- They are using **Azure Virtual WAN (VWAN)** as the remote service for P2S VPN termination in Azure, and onwards access to applications hosted in Azure. ([Azure Virtual WAN also allows transit routing via Hybrid connections, such as ExpressRoute or S2S VPN, back to On-Premises if required]( https://docs.microsoft.com/en-us/azure/virtual-wan/virtual-wan-global-transit-network-architecture))
- They are currently using an **Always-On** configuration for all users
- Some of their staff only leverage the VPN for infrequent use of a single application
- They would like to explore if it’s possible to change the VPN connection behaviour to optimise cost (Azure P2S termination solutions, VPN Gateway and Azure VWAN, incur P2S connection charges on a per connection, per hour basis I.e., the amount of time a client is connected, has direct impact on the overall solution cost).

## Big Picture

Caveat empor. The solution explored in this article is narrowly focused on allowing a VPN to trigger based on specific conditions. There are other technical approaches to solving these customer scenarios, that fall outside the scope of this document. Examples include exposing the required application directly to the Internet and using a Modern Authentication method, or utilising Azure AD Application Proxy to publish the application via the Internet, thereby potentially removing the requirement for a Client VPN at all.

## Win 10 VPN 

A (very) quick summary of the Windows 10 VPN platform:

- Built-in VPN Client software. Pre-packaged out of the box, supports IKEv2 and SSTP protocols
- Supports universal VPN plugins, often used to enhance the base feature set E.g. to support SSL based VPNs
  - Azure VPN Client is just one example of a plugin, many other vendors such as Palo Alto exist in this space
   - These plugins require installation beyond what is supplied in the base Win10 O/S
- Configuration via the VPNv2 configuration service provider (CSP) standardised interface https://docs.microsoft.com/en-us/windows/client-management/mdm/vpnv2-csp
  -  This CSP can be configured locally via PowerShell, or remotely via an MDM (E.g., Intune)
  -  Support for ProfileXML files that contain a list of profile settings, in a defined structure that aligns with the parameters set out in the VPNv2 schema

More technical detail here https://docs.microsoft.com/en-us/windows/security/identity-protection/vpn/vpn-guide

Using the above structure, it is therefore possible to leverage the native VPNv2 schema for general parameter definition (E.g. When should this VPN connect? what server should I connect to? what is my P2S DNS configuration?), whilst at the same time leverage the Universal Plugin for authentication types and transport mechanisms not provided by the native O/S. Azure VPN Client is an example of this, providing supports for the OpenVPN transport protocol, and Azure AD authentication, both of which are not provided by Windows10 Client VPN capability out of the box. This article builds upon this functionality to define general trigger settings that affect the Azure VPN Client behaviour.

# VPN Trigger options

The popularity of the term Always-On VPN (AOVPN) can cause confusion when approaching VPN designs on Windows 10. Whilst its true the Always-On experience is the most common configuration; it is possible to configure the client to activate in several ways.

- Trigger option 1. **Always-O**n. To achieve this, simply add <AlwaysOn>true</AlwaysOn> to your XML file. The VPN will connect after the user logs in and remain connected.
- Trigger option 2. **Manual**, user driven. Set <AlwaysOn>false</AlwaysOn> in your XML file. The VPN only connects when a user hits “connect”, and only disconnects when they choose “disconnect”
- Trigger option 3. **App-based trigger**, the VPN connects after a specified application is launched and disconnects ~5 minutes after the application is closed.
- Trigger option 4. **Name-based trigger**, the VPN connects after the O/S processes a DNS lookup for a specified domain. See section below for disconnect behaviour related to this trigger type.

More detail:  https://docs.microsoft.com/en-us/windows/security/identity-protection/vpn/vpn-auto-trigger-profile

_Important!_
The following tick-box =/= Always-On. Instead, this effectively tells the VPN platform to connect automatically **only** if one of the above triggers = YES.

![auto_connect](https://github.com/adstuart/azure-vpn-p2s/blob/main/intune-win10-triggers/images/connect_auto.png)

# Lab Setup

### Prerequisites

- Client is AAD joined, and enrolled in Intune
- Client machine has required client certificates installed
- Azure VPN Client is installed on Windows 10 (either manually, or via Intune)

See more:

https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-profile-intune

https://docs.microsoft.com/en-us/azure/virtual-wan/vpn-profile-intune

### Configuration

-	Authentication: Certificate
-	Azure head-end: Azure VWAN P2S
-	Protocol: OpenVPN
-	Client O/S: Win10 (1909)
-	Client O/S VPN: User Tunnel only
-	Client VPN software: Win10 + Azure VPN Client plugin
-	Device Management/MDF: Intune

# App-based auto trigger 

## Configuration
Inside of your XML, specify the following configuration parameters:
   
```
<AppTrigger>  
    <App>  
      <Id>C:\windows\system32\notepad.exe</Id>  
    </App>  
</AppTrigger>
```

Verify App-Trigger settings have been pushed to client, by launching PowerShell and running `Get-VpnConnectionTrigger`

```
PS C:\Users\AdamStuart> Get-VpnConnectionTrigger

cmdlet Get-VpnConnectionTrigger at command pipeline position 1
Supply values for the following parameters:
ConnectionName: Global-WAN

ConnectionName         : Global-WAN
ApplicationID          : { C:\windows\system32\notepad.exe }
```
### :point_right: Note

_App-trigger, and name-trigger do not activate if you are leveraging Trusted Network Detection and your DNS suffix on the Ethernet/Wi-Fi interface matches the variable specified in this parameter._

## User Experience

VPN does not connect automatically

![image](https://github.com/adstuart/azure-vpn-p2s/blob/main/intune-win10-triggers/images/a.PNG)

VPN connects when application is launched
  
![image](https://github.com/adstuart/azure-vpn-p2s/blob/main/intune-win10-triggers/images/c.PNG)

VPN disconnects after ~5 minutes of closing the app

# Name-based autotrigger 

## Configuration
Inside of your XML, specify the following configuration parameters:
   
```
<DomainNameInformation>  
    <DomainName>hrapp.contoso.com</DomainName>  
    <DnsServers>8.8.8.8</DnsServers>  
    <AutoTrigger>true</AutoTrigger>  
</DomainNameInformation>
```
  
Verify Name-Trigger settings have been pushed to client, by launching PowerShell and running `Get-VpnConnectionTrigger`

```
PS C:\Users\AdamStuart> get-vpnconnectiontrigger

cmdlet Get-VpnConnectionTrigger at command pipeline position 1
Supply values for the following parameters:
ConnectionName: GlobalWAN

ConnectionName         : GlobalWAN
TrustedNetwork         : {corp.contoso.com}

Dns Suffix                                                  Dns Servers
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _           _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
hrapp.contoso.com                                           {8.8.8.8}
```
  
### :point_right: Note

_App-trigger, and name-trigger do not activate if you are leveraging Trusted Network Detection and your DNS suffix on the Ethernet/Wi-Fi interface matches the variable specified in this parameter._
  
## User Experience

VPN does not connect automatically
  
![image](https://github.com/adstuart/azure-vpn-p2s/blob/main/intune-win10-triggers/images/a.PNG)

VPN connects when DNS lookup is performed is running
  
![image](https://github.com/adstuart/azure-vpn-p2s/blob/main/intune-win10-triggers/images/b.PNG)

### :point_right: Note
  
_The VPN disconnect experience for name-trigger is different to that of app-trigger. This is presumably because the O/S has a clear way to acknowledge when an application is closed but applying the same approach to DNS lookups only would result in an unusable intermittent connection. Therefore, by default, with name-trigger. The VPN will trigger, and then remain connected until the user logs off._

It is possible to change this behaviour of Win10 VPN, by modifying a setting called `IdleDisconnectSettings` E.g.

`PS C:\Users\AdamStuart> Set-VpnConnection -Name GlobalWAN -IdleDisconnectSeconds 10 -thirdpartyvpn`

This will then allow the VPN Connection to timeout if an active trigger is not detected. In my testing with name-trigger, it took between 5-15mins to disconnect, but I am unaware of the underlying variables in play. If you run `Get-VPNconnection`, you will notice that the default IdleTimeoutSeconds value = 0, I.e., Idle timeout is disabled.

### :point_right: Note
  
_Unfortunately the VPNv2 schema does not appear to include the IdleTimeoutSeconds variable, therefore you cannot use the Profile XML definition approach. In my testing I used local PowerShell as per above, however if working at scale, you can package the script via Intune. https://docs.microsoft.com/en-us/mem/intune/apps/intune-management-extension_

### :point_right: Note
  
_The easiest way to review historical VPN connect/disconnects is via Event Viewer as per below:_
  
![event](https://github.com/adstuart/azure-vpn-p2s/blob/main/intune-win10-triggers/images/event.PNG)
  
<image>


# Important!
  
I was initially testing using Windows 10 clients hosted on Azure Virtual Machines, however app/name triggering does not seem to function in this scenario. (Always-On triggering works fine). I therefore switched to using a local Windows 10 client. One to watch out for if you are proving this out in a virtual lab before production.



# Example pricing impact
  
Virtual WAN pricing: https://azure.microsoft.com/en-gb/pricing/details/virtual-wan/
  
VPN Gateway pricing: https://azure.microsoft.com/en-gb/pricing/details/vpn-gateway/

Example based on Virtual WAN, the subject of this article:

-	Contoso has 1000 users
-	500 require “Always-On” VPN, as they work throughout the day using systems that require  an active Private network connection to applications hosted in Azure. 
-	500 primarily work “over the Internet” using SaaS services such as M365, spending most of their working day using Office apps such as Outlook and SharePoint.
  - These users only require the VPN connection on Friday afternoons when performing a specific task, that requires an application that users a private network connection to Azure
  
Before tuning: Variable P2S Connection Units cost only
  
-	1000 users * 40 hours per week * 48 working weeks per year * $0.013 per hour charge = $24960 p/a

After tuning: Variable P2S Connection Units cost only
  
-	500 users * 40 hours per week * 48 working weeks per year * $0.013 per hour charge = $12480 p/a
-	500 users  * 4 hours VPN required per week * 48 working weeks per year * $0.013 per hour charge = $1248p/a
-	Total cost $13728 p/a (=$11232 saving p/a)

You achieve the above at scale by using multiple Intune Device Configuration profiles, assigned to different groups of users. For example, in the screenshot below, you can see I have 4 3 different VPN profiles that all use the same Azure VWAN service in the Cloud, however they contain different triggering options to reflect the different user requirements.

![intune](https://github.com/adstuart/azure-vpn-p2s/blob/main/intune-win10-triggers/images/intune.PNG)

# Closing

Thanks to my fellow Microsoft colleagues for supporting this testing.
  
- Antonio Traetto
- Jack Tracey
- Claire Brydon
- Jason Jones 
