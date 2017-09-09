---
title: Verifying WinRM Connectivity
description: Verifying WinRM Connectivity
header: Verifying WinRM Connectivity
tags: [chef, dev]
---
Originally found as a troubleshooting tip on the [Chef Compliance](https://www.chef.io/solutions/compliance/) server, I've found myself doing the following so often that I'm capturing this here for posterity.

Hope this helps with your WinRM debugging journey!

> Here are a few steps to enable and verify the WinRM configuration of a node:

> From CMD, start the WinRM service and load the default WinRM configuration.

>`c:\> winrm quickconfig`

>Verify whether a listener is running, and which ports are used. The default ports are 5985 for HTTP, and 5986 for HTTPS.

>`c:\> winrm enumerate winrm/config/listener`

>Enable basic authentication on the WinRM service. Run the following command to check whether basic authentication is allowed.

>`c:\> winrm get winrm/config/service`

>Run the following command to enable basic authentication on the WinRM service. If you use `POWERSHELL`, you need quotes here `'@{...}'`

>`c:\> winrm set winrm/config/service/auth @{Basic="true"}`

>Run the following command to allow transfer of unencrypted data on the WinRM service.

>`c:\> winrm set winrm/config/service @{AllowUnencrypted="true"}`

>Enable Unencrypted client connections for the test winrm identity command to work.

>`c:\> winrm set winrm/config/client @{AllowUnencrypted="true"}`

>Run the following command to test the connection to the WinRM service.

>`c:\> winrm identify -r:http://NODE:5985 -auth:basic -u:USERNAME -p:PASSWORD -encoding:utf-8`
