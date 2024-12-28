---
title: "Export Access Package Resource Groups"
date: 2024-12-28
categories:
  - Access Packages
---

Access Packages are very useful but currently lack robust reporting options. Imagine working in a large environment with over 300 Access Packages, and needing to identify which ones are responsible for adding people to a PIM group (PIM_Security_Sentinel_Reader).
Unfortunately, there isn't an out-of-the-box solution for this query. In the first part of this series, we will write a script that exports all Access Packages across all Catalogs and shows their corresponding groups. We will then export this data to Excel, allowing us to use the filter option for easier analysis.

## Result
We want to have something like this allowing us to filter on GroupDisplayname: 

![alt]({{ site.url }}{{ site.baseurl }}/assets/images/excel.png)

Filtering on GroupDisplayname we can see that PIM_Security_Sentinel_Reader is in the following Access Packages:

![alt]({{ site.url }}{{ site.baseurl }}/assets/images/excel2.png)



