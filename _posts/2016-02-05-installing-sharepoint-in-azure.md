---
title: Installing Sharepoint in Azure
layout: post
---

### Requirement

I had some Sharepoint applications to write. I therefore needed a development instance to work against.

### Blocker

My MSDN license gave me access to an evaluation version of Sharepoint 2013, but my development machine was a Macbook, and there wasn't much chance of Sharepoint running on that. I did have [parallels](http://www.parallels.com) installed, but that was running Windows 8.1 - and Sharepoint 2013 requires a Windows Server OS to run under. So I needed an instance of Windows 2012, and somewhere to run it.

### Solution

 I originally intended to run the Windows 2012 instance under VirtualBox. This had two benefits - (1) VirtualBox is free, and (2) I could run it on my Macbook. But for reasons I was never able to resolve, Microsoft refused to let me download the evaluation Windows Server 2012 ISO. Luckily MSDN Enterprise comes with some free Azure time each month. And Azure comes with prebuilt images, which include Windows Server 2012.

### Steps

#### Activating Microsoft Azure

1. Create a Microsoft login, and associate that account with your MSDN license
1. Go to http://msdn.microsoft.com/subscriptions
1. Select "My Account"
1. Select "Activate Microsoft Azure". This will give you $150 per month of credits.

#### Start a Windows Server 2012 Virtual Machine

Azure has two portals (see below for links). The documentation you will find online for creating a Sharepoint instance under Azure references the old management portal. But here are the instructions for the new portal.

1. Go to new [Azure Portal](https://portal.azure.com) and login
1. Select "Virtual Machines"
1. Click "+ (Add)" in the top left hand corner.
1. Select "Windows Server" from the gallery.
1. You have a choice here, I chose "Windows Server 2012 R2 Datacenter"
1. Choose classic as the resource manager
1. Select "create"
1. Enter a username and password that will be the administration user on this machine
1. Wait. (You will do a lot of this)

#### Install Sharepoint 2013

Hint: As it can take a while for the Windows server to provision I downloaded the evaluation image of Sharepoint 2013 locally. I think uploaded this onto my OneDrive so I could access it from the virtual machine. Also, so I would have access to it for future installations.

1. Download the "SharePointServer_x64_en-us.img" evaluation image from the link (see below)
1. Make a note of the key - you will need this later
1. Mount the image by double clicking on the file
1. Run the executable file "prerequisiteinstaller.exe" (right click to run as Administrator)
1. Once this has completed run the "Setup" file. This is where you'll need the key

#### Known issues

It is entirely possible you will get the following error when you run the "prerequisiteinstaller.exe" file.

>  The tool was unable to install Application Server Role, Web Server (IIS) Role.

To resolve this, you will need to complete the following Steps

1. Open a Powershell terminal and run the shell scripts (see below)
1. Go to the following folder Windows System32 folder (C:\Windows\system32)
1. Check that there is an executable called "ServerManagerCmd.exe"
1. If there is not, copy the file ServerManager.exe to ServerManagerCmd.exe
1. Then rerun the "prerequisiteinstaller.exe" file
1. Once that is complete, you can then run the "Setup".

##### Powershell Script (see above known issue)
~~~~~~
Import-Module ServerManager

Add-WindowsFeature NET-WCF-HTTP-Activation45,NET-WCF-TCP-Activation45,NET-WCF-Pipe-Activation45

Add-WindowsFeature Net-Framework-Features,Web-Server,Web-WebServer, `
    Web-Common-Http,Web-Static-Content,Web-Default-Doc,Web-Dir-Browsing, `
    Web-Http-Errors,Web-App-Dev,Web-Asp-Net,Web-Net-Ext,Web-ISAPI-Ext, `
    Web-ISAPI-Filter,Web-Health,Web-Http-Logging,Web-Log-Libraries,Web-Request-Monitor, `
    Web-Http-Tracing,Web-Security,Web-Basic-Auth,Web-Windows-Auth,Web-Filtering, `
    Web-Digest-Auth,Web-Performance,Web-Stat-Compression,Web-Dyn-Compression, `
    Web-Mgmt-Tools,Web-Mgmt-Console,Web-Mgmt-Compat,Web-Metabase,Application-Server, `
    AS-Web-Support,AS-TCP-Port-Sharing,AS-WAS-Support, AS-HTTP-Activation, `
    AS-TCP-Activation,AS-Named-Pipes,AS-Net-Framework,WAS,WAS-Process-Model, `
    WAS-NET-Environment,WAS-Config-APIs,Web-Lgcy-Scripting,Windows-Identity-Foundation, `
    Server-Media-Foundation,Xps-Viewer
~~~~~~

### Useful links

MSDN (-> Downloads -> Trial Software)
: https://msdn.microsoft.com

Evaluation version of Sharepoint 2013
: https://msdn.microsoft.com/sharepoint2013resources

Evaluation version of Windows Server 2012
: https://msdn.microsoft.com/windowsserver2012r2resources

Microsoft Azure management portal
: https://manage.windowsazure.com/

New Portal
: https://portal.azure.com/#
