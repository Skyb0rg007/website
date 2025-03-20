---
title: 'Introduction to WDAC'
date: null
draft: true
---

# Guide for home users

Windows Defender Application Control (WDAC) is a Windows system for
whitelisting executables, drivers, dlls, and scripts.

The preinstalled Windows Defender mostly relies on telemetry to block
known viruses by file hash.
Other antivirus software will inspect what an unknown application is doing,
and quarantine it when it tries to perform suspicious actions.
Both of these methods are reactive: everything is allowed to run,
unless it has a specific hash or performs a specific task.

WDAC is an active form of protection, which is meant to prevent *everything*
from running.
Then you one-by-one enable applications to be able to use your system.

On your phone, this is how things work for the most part.
Every application is reviewed by Apple/Google, and is run in a sandboxed
environment.
An analogue on Linux for WDAC would be SELinux/AppArmor, which let you
specify when users can execute certain binaries.
