---
title: "Installing Applications Remotely using Powershell"
layout: post
excerpt: " "
tags:
  - powershell
---

## Objective

Installing [sshfs-win](https://github.com/billziss-gh/sshfs-win) on a remote Windows 10 host.

## Tasks

- Download the binaries on local system
- Open up a session with the remote host using `New-PSSession`
- Copy files to the remote system
- Execute the msi files on remote system using `Invoke-Command`

Before continuing, ensure WinRM service is running on the remote host.

## Challenges faced

- On running `New-PSSession -ComputerName $computerName -credential $cred`, I got an error
  ```
  New-PSSession : [X.X.X.X] Connecting to remote server X.X.X.X failed with the following error message : The WinRM client cannot process the request. Default authentication may be used with an IP address under the following conditions: the transport is HTTPS or the destination is in the TrustedHosts list, and explicit credentials are provided. Use winrm.cmd to configure TrustedHosts. Note that computers in the TrustedHosts list might not be authenticated. For more information on how to set TrustedHosts run the following command: winrm help config. For more information, see the about_Remote_Troubleshooting Help topic
  ```
- To fix this I ran `Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force`. But this gave another error
  ```
  Set-Item : The client cannot connect to the destination specified in the request. Verify that the service on the destination is running and is accepting requests. Consult the logs and documentation for the WS-Management service running on the destination, most commonly IIS or WinRM. If the destination is the WinRM service, run the following command on the destination to analyze and configure the WinRM service: "winrm quickconfig".
  ```
  Turns out winrm service was not running on my laptop and I did not have administrative privileges to do so.
- The easier solution was to connect to the remote host using WinRM over HTTPS (over port 5986).
  - First configure the remote host:
    ```
    winrm create winrm/config/Listener?Address=*+Transport=HTTPS @{Hostname="<your_server_dns_name_or_whatever_you_like>"; CertificateThumbprint="<certificate_thumbprint_from powershell>"}
    ```
    Another option is to run this script on the remote host: <https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1>
- Now we can create session like this:
  ```
  $so = New-PsSessionOption -SkipCACheck -SkipCNCheck
  $session = New-PSSession -ComputerName $computerName -credential $cred -UseSSL -SessionOption $so
  ```

Complete script is available here: <https://gist.github.com/amard33p/d98b1c1f9579a7ee9e23327b9d5a6d40>

_References:_  
- <https://powershellexplained.com/2017-04-22-Powershell-installing-remote-software/>  
- <https://cloudblogs.microsoft.com/industry-blog/en-gb/cross-industry/2016/02/11/configuring-winrm-over-https-to-enable-powershell-remoting/>
