# Misc commands, links and snippets of knowledge useful when working with Azure VPN P2S solutions

## How to force Intune resync from PowerShell?

Useful for packaging in to scripts etc. Taken from https://oofhours.com/2019/09/28/forcing-an-mdm-sync-from-a-windows-10-client/

`Get-ScheduledTask | ? {$_.TaskName -eq ‘PushLaunch’} | Start-ScheduledTask'`

## How to force disconnect a Windows 10 VPN from command line?

`rasdial <connection name> /DISCONNECT`

## P2S Gateway Powershell commands

- Health including P2S user count, `Get-AzP2sVpnGatewayConnectionHealth -ResourceGroupName <rg> -Name <gw-name>`

- P2S user connected details, `Get-AzP2sVpnGatewayDetailedConnectionHealth h -ResourceGroupName <rg> -Name <gw-name> -OutputBlobSasUrl <sas url>`, note SaS URI must point to text file in blob, not to the container itself. Example output:

> `[{"P2SConnectionConfigurationResourceId":"/<resource-ID>","UserNameVpnConnectionHealths":[{"UserName":"P2SChildCert","VpnConnectionHealths":[{"VpnConnectionId":"OVPN_A2915F26-1E9E-E893-2BA3-AFD87666B44E","VpnConnectionDuration":165,"VpnConnectionTime":"2021-07-26T21:03:03","PublicIpAddress":"77.98.70.226:54192","PrivateIpAddress":"172.20.202.130","UserName":"P2SChildCert","MaxBandwidth":39000000,"EgressPacketsTransferred":9,"EgressBytesTransferred":351,"IngressPacketsTransferred":29,"IngressBytesTransferred":6660,"MaxPacketsPerSecond":1}]}]}]`

Note how this includes connected duration, external PiP and internal allocated P2S IP address

- Disconnect a specified P2S user, (get the user connection ID from the previous command) `Disconnect-AzP2sVpnGatewayVpnConnection  -ResourceGroupName <rg> -VpnConnectionId "OVPN_635294B2-475F-074B-6E31-ADAB452F5247"`

## Force tick the "connect automatically" tickbox in Windows 10, via PowerShell

https://powers-hell.com/2020/11/28/set-your-azure-vpn-connections-to-connect-automatically-with-powershell/


