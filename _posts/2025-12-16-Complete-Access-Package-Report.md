---
title: "The Ultimate Entra ID Access Package Report & Authorization Matrix"
date: 2025-12-16
classes: wide
categories:
  - Access Packages
---

If you’ve spent any time working with Access Packages in Entra ID Entitlement Management, you’ve probably come to the same conclusion I have: the built-in reporting options leave a lot to be desired.

Picture this: You need to remove a specific security group, but that group is buried as an approver or reviewer in about 40 different access packages. How do you find them? Or maybe you need to hunt down every package that contains a specific PIM group, or see which ones trigger a Logic App upon assignment.

Right now, your only option is to click through every single access package, check the settings, rinse, and repeat. If you have 10 packages? Fine. If you have 50, 100, or 1,000? It’s an absolute nightmare.

I got tired of the manual clicking, so I wrote a PowerShell script to save my sanity. It captures almost everything. Basically, if it’s in your access package, this script probably exports it.

## Script
If you don't want to read through the prerequisites, feel free to download the [script](https://github.com/TiboPowershell/PowershellScripts/blob/main/FullAccessPackageReport/FullAccessPackageReport.ps1) directly.

## Result
When you run this script, it will create an Excel file that looks like this. It has a lot of worksheets so I will only show some of them.
![alt]({{ site.url }}{{ site.baseurl }}assets/images/fullaccesspackagereport/Screenshot_1.png)
