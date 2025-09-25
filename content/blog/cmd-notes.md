---
title: Nano server cmd.exe guide
draft: true
---

# Executables

## AggregatorHost
?

## ARP
https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/arp

Controls the Address Resolution Protocol cache

## attrib
Change file attributes

## auditpol
Manipulate audit policies

## autochk
Used by chkdsk, can't be run directly

## bcdedit
Edit the Boot Configuration Data stores

## CertEnrollCtrl
?

## chkdsk
Check the file system for errors

## cmd
Command interpreter

## conhost
Manages console windows

## csrss
Client/Server Runtime Subsystem

## curl
The regular Curl for Windows, version 8.14.1

## DeviceCensus
For device information

## diskusage
Summarize disk usage recursively

## Dism
Deployment Image Servicing and Management

Nanoserver comes with:

## dllhost
Used for running executables

## dtdump
Used for Windows error reporting

## expand
File expansion utility.

Extracts files from different archive formats.

## findstr
Search for a pattern in files.
Windows `grep`.

## finger
Implements the finger protocol.

## fltMC
Manage MiniFilter drivers.

## HOSTNAME
Prints the computer hostname.

## icacls
Display and modify Discretionary Access Control Lists on files and directories.

## ipconfig
Display IP configuration.
Can also be used to refresh DHCP and/or DNS.

## MLEngineStub
?
Potentially a LOLbin?

## MoNotificationUxStub
MoNotificationUx.exe handles system notifications.

## mountvol
Create, delete, or list volume mount points.

## MRINFO


## MuiUnattend
## net
A whole bunch of stuff.
Mostly responsible for managing user accounts.

## net1
Same as `net`, was introduced to fix Y2K.

## netsh
Manages network components.

## NETSTAT
Displays active TCP connections and other stateful internet info.

## ntoskrnl
The Windows Kernel.

## PATHPING
Computes network latency information.

## PING
Sends an ICMP echo request message.

## PkgMgr
Deprecated command: use dism

## poqexec
?
May be remnant of a different age, since it doesn't even run.

## rdrleakdiag
?
May not be implemented in NanoServer, and just be a stub.

## reg
Performs registry modifications.

## regsvr32
Registers dll files as command components.

## ROUTE
Manipulates network routing tables.

## runexehelper
Launches a given exe

May not be implemented in NanoServer, and just be a stub.

## sc
Communicates with the Service Control Manager

## schtasks
Manages scheduled tasks

## services
?

## setx
Modifies environment variables

## smss
Cannot be run in Win32 mode.

## sort
Sorts input.

## tar
Same as the Unix tar command.

## taskhostw
?

## TCPSVCS
?

## tlist
List running programs and their PIDs.

## TRACERT
Trace an IP request. Windows's traceroute.

## UsoClient
Update Session Orchestrator client. Manages Windows updates.

Nanoserver has a stub.

## WerFault
## WerFaultSecure
## wermgr
## wevtutil
Retrieves information about event logs.

## wimserv
Windows Disc Imaging Utility component

## wpr
Windows Performance Recorder.

## wuauclt
Stub.

## WUDFCompanionHost
?

## WUDFHost
?

## xcopy
Copies files and directories.

# Services

Name
Main Binary
Credentials

Description

## cexecsvc
Container Execution Agent
cexecsvc.exe
LocalSystem

This is used to implement the Windows container

## CryptSvc
Cryptographic Services
svchost.exe
NetworkService

Contains:
- Catalog Database Service
  * Confirms signatures for files + new programs to be installed
- Protected Root Service
  * Adds and removes Trusted Root Certificate Authority certificates
- Automatic Root Certificate Update Service
  * Retrieves SSL root certificates from Windows Update

## DcomLaunch
DCOM Server Process Launcher
svchost.exe
LocalSystem

Launches COM and DCOM servers in response to object activation requests.

## Dhcp
Dynamic Host Configuration Protocol Client
svchost.exe
LocalService

DHCP and DHCPv6 Client, used to obtain an IP address.

## DiagTrack
Connected User Experiences and Telemetry
svchost.exe
LocalSystem

Windows telemetry.
TODO: How do I change the usage privacy option in NanoServer?

## Dnscache
DNS Client
svchost.exe
NetworkService

Sends DNS queries, caches results, registers computer's name.

## EventLog
Windows Event Log
svchost.exe
LocalService

Supports logging events, querying events, subscribing to events,
archiving event logs, managing event metadata, etc.

## nsi
Network Store Interface Service
svchost.exe
LocalService

Delivers network notifications to user mode clients.
Required for network connection.

## ProfSvc
User Profile Service
svchost.exe
LocalSystem

Loads and unloads user profiles.

## RpcEptMapper
RPC Endpoint Mapper
svchost.exe
NetworkService

Resolves RPC interface identifiers. Required for Remote Procedure Calls.

## RpcSs
Remote Procedure Call (RPC)
svchost.exe
NetworkService

Service Control Manager for COM and DCOM servers.

## SamSs
Security Accounts Manager
lsass.exe
LocalSystem

Enforces security policy, handles password changes, etc.

## Schedule
Task Scheduler
svchost.exe
LocalSystem

Allows users to schedule automated tasks.

## SystemEventsBroker
System Events Broker
svchost.exe
LocalSystem

Executes background tasks for WinRT apps.

## TimeBrokerSvc
Time Broker
svchost.exe
LocalService

Executes background tasks for WinRT apps.

## UserManager
User Manager
svchost.exe
LocalSystem

Provides runtime components for multi-user interaction.

## WinHttpAutoProxySvc
WinHTTP Web Proxy Auto-Discovery Service
svchost.exe and pacjsworker.exe
LocalService

Implements Web Proxy Auto-Discovery: looks for a wpad.domain.tld/wpad.dat file,
then uses pacjsworker.exe to execute the JavaScript file whenever a URL is
accessed to determine how to proxy the connection.

# DISM Capabilities

## Microsoft.NanoServer.ADSI
Active Directory Service Interfaces

## Microsoft.NanoServer.BITS.Admin
Background Intelligent Transfer Service

## Microsoft.NanoServer.CommandLine.Utilities
Adds:
- comp
- fc
- find
- fsutil
- ftp
- makecab
- replace
- Robocopy
- takeown
- where

## Microsoft.NanoServer.Datacenter.WOWSupport
WOW - 32-bit support.
This may only implement 32-bit for its containers (?)

## Microsoft.NanoServer.Globalization
## Microsoft.NanoServer.IIS
## Microsoft.NanoServer.OpenSSH.Client
## Microsoft.NanoServer.OpenSSH.Server
## Microsoft.NanoServer.Performance.Monitoring
## Microsoft.NanoServer.PowerShell.Cmdlets
Didn't install for me

## Microsoft.NanoServer.RemoteFS.Client
## Microsoft.NanoServer.StateRepository
?
May implement C:\ProgramData\Microsoft\Windows\AppRepository

## Microsoft.NanoServer.WinMgmt
## Microsoft.NanoServer.WinMgmt.RuntimeDependencies
