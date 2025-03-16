+++
authors = ["Marcel Nguyen"]
title = "IGA Insights"
date = "2025-03-16"
description = "IGA Insights"
tags = [
    "IAM","IGA","PowerShell","KQL","Entra","Azure",
]
categories = [
    "IGA","Powershell"
]
series = [""]
+++

# 🔎 Enhance IGA in SSO Applications with IGA insights 

## Introduction

In one of my last projects, a colleague in my company asked if we could automate a time-consuming task that takes him up to 40 hours of manual work a year: identify stale user accounts in a third-party SSO application. The current process was neither efficient nor accurate, relying on a large log file of the "last sign-in dates" of the Azure accounts rather than the actual SSO application login. 

In this article, I will show you how I tackled and solved this problem using PowerShell and KQL.


## 🤨 Why do accounts become inactive in SSO applications? 
There are several reasons:
**Employee Turnover**: When employees leave the organization, their accounts may not be immediately deactivated, leading to inactivity.
**Role Changes**: Employees moving to different roles within the organization might no longer need access to certain applications. If there is no (automated) process in place to revoke the access the account is getting inactive.
**Inactive Users**: Users who take extended leave (such as sabbatical or maternity leave) might have inactive accounts during their absence.

## 🤔 What is IGA?  
But first, let's clarify: What exactly is IGA? Identity Governance and Administration (IGA) is an integral component of Identity and Access Management (IAM), designed specifically to address the scenarios I've mentioned above. Professional IGA solutions manage digital identities and control access to systems and data, ensuring that only authorized individuals can access the necessary resources (Zero Trust Model).


## 🤷 Why not use "Access Reviews" in Entra-ID? 
In my opinion, the out-of-the-box Access Review feature is a great feature but lacks a lot of flexibility. My problems were following:
**Data**:  Initially, I was uncertain about the specific data used to determine whether a user is active or inactive. It now appears that the last SSO sign-in date is also considered.
**Flexibility**: The decision-helper provides recommendations based solely on the past 30 days, which did not meet our needs due to different requirements from external auditors. 
**Efficiency**: Managing access control list groups requires setting up one access review per group, which proved to be inconvenient for us.

## 👨‍💻 My Approach with PowerShell 
**How the script works in a nutshell**:
Creates a hash table of users who signed into the specified app in the past x days (based on the threshold you set).
Creates an array with all users that are assigned to the application (via security groups)
Compares the array with the hash table to identify users who haven't logged in within the threshold.
Categorizes users into active and inactive arrays.
Exports the inactive users in a CSV-file.

## What the script does

In a nutshell, the script performs the following steps:

1. Connects to Azure and Microsoft Graph services
2. Queries Log Analytics Workspace for sign-in data
3. Retrieves group memberships for the specified application
4. Combines the data to identify active and inactive users
5. Generates detailed reports in both CSV and interactive HTML formats

## Prerequisites

Before running the script, you'll need:

- Az PowerShell module
- Microsoft.Graph PowerShell module
- Azure account with Log Analytics access
- Appropriate permissions (Log Analytics Reader, Azure AD Reader)

## Key Components of the Script

### 1. Authentication Methods

The script supports two authentication methods:

```powershell
# User-based authentication
if (-not (Initialize-UserConnection)) {
    exit 1
}

# App-based authentication
if (-not (Initialize-AppRegistrationConnection -TenantId $TenantId -ClientId $ClientId -CertThumbprint $CertThumbprint)) {
    exit 1
}
```

This flexibility allows you to run the script interactively or as part of an automated process using service principals.

### 2. Data Collection from Log Analytics

The script uses KQL (Kusto Query Language) to query the Log Analytics workspace:

```powershell
# KQL query for active users
$activeUsersQuery = @"
SigninLogs
| where TimeGenerated > ago(${ThresholdDays}d)
| where AppDisplayName =~ '$($AppName)'
| summarize LatestSignIn = arg_max(TimeGenerated, *) by UserPrincipalName
| project UserPrincipalName, LatestSignIn = format_datetime(LatestSignIn, 'yyyy-MM-dd HH:mm:ss')
"@
```

This query finds all users who have signed in to the specified application within the threshold period.

### 3. Group Membership Retrieval

The script identifies which groups are assigned to the application and gets all members from those groups:

```powershell
function Get-EnterpriseAppGroups {
    param (
        [string]$ApplicationName
    )
    
    # Get the service principal for the application
    $servicePrincipal = Get-MgServicePrincipal -Filter "DisplayName eq '$ApplicationName'"
    
    # Get app role assignments (group assignments)
    $assignments = Get-MgServicePrincipalAppRoleAssignedTo -ServicePrincipalId $servicePrincipal.Id -All
    
    # Filter for group assignments
    $groupAssignments = $assignments | Where-Object { $_.PrincipalType -eq "Group" }
    
    # Get group details for each assignment
    $groups = @()
    foreach ($assignment in $groupAssignments) {
        $group = Get-MgGroup -GroupId $assignment.PrincipalId
        $groups += $group
    }

    return $groups
}
```

### 4. User Activity Analysis

The script combines the sign-in data with group membership to categorize users:

```powershell
foreach ($user in $unique_app_users) {
    $userGroups = $userGroupMemberships[$user.UserPrincipalName] -join "; "
    $jobTitle = $userJobTitles[$user.UserPrincipalName]
    $Department = $userDepartments[$user.UserPrincipalName]
    
    if ($activeUsers.ContainsKey($user.UserPrincipalName)) {
        $lastSignIn = $activeUsers[$user.UserPrincipalName]
        # User is active
        $activeAppUsers += [PSCustomObject]@{
            UserPrincipalName = $user.UserPrincipalName
            LatestSignIn = $lastSignIn
            AppDisplayName = $AppDisplayNames[$user.UserPrincipalName]
            Groups = $userGroups
            JobTitle = $jobTitle
            Department = $Department
        }
    }
    else {
        # User is inactive
        $inactiveAppUsers += [PSCustomObject]@{
            UserPrincipalName = $user.UserPrincipalName
            LatestSignIn = $lastSignIn
            AppDisplayName = $AppDisplayName
            Groups = $userGroups
            JobTitle = $jobTitle
            Department = $Department
        }
    }
}
```

### 5. Interactive HTML Reporting

It generates a comprehensive HTML report including:

- Summary statistics
- Interactive charts (using Chart.js)
- Filterable, paginated user table
- Export to CSV functionality

The report provides a modern UI with filtering capabilities:

```javascript
// Table filtering functionality
const filterConfig = [
    { dropdownId: 'filter-upn', columnIndex: 0 },
    { dropdownId: 'filter-job-title', columnIndex: 1 },
    { dropdownId: 'filter-department', columnIndex: 2 },
    { dropdownId: 'filter-last-signin', columnIndex: 3 },
    { dropdownId: 'filter-status', columnIndex: 4 },
    { dropdownId: 'filter-groups', columnIndex: 5 }
];

function filterTable() {
    const table = document.getElementById('users-table');
    const rows = table.getElementsByTagName('tbody')[0].getElementsByTagName('tr');

    for (let row of rows) {
        let showRow = true;

        filterConfig.forEach(config => {
            const dropdown = document.getElementById(config.dropdownId);
            const dropdownValue = dropdown.value;
            const cellValue = row.cells[config.columnIndex].textContent.trim();

            if (dropdownValue && cellValue !== dropdownValue) {
                showRow = false;
            }
        });

        row.style.display = showRow ? '' : 'none';
    }
}
```

## Running the Script

Using the script is straightforward:

```powershell
.\iga-insights.ps1 -AppName "Figma" -ThresholdDays 90
```

This will analyze the Figma application and identify users who haven't signed in within the last 90 days.


## Conclusion

Identity Governance is an essential component of any modern security strategy. By implementing tools like this PowerShell script, you can gain valuable insights into your SaaS application usage, optimize costs, and improve your security posture.

The script demonstrates the power of combining Azure Log Analytics, Microsoft Graph, and PowerShell to create comprehensive governance solutions. The interactive HTML report makes it easy for both technical and business stakeholders to understand application usage patterns.

I encourage you to try this script in your environment and see what insights you can uncover about your own SSO application usage. The full code is available on [GitHub](https://github.com/marcel-ngn/iga-insights) if you want to contribute or customize it further.


⚠️ Disclaimer: This content reflects my personal experience. 
Please refrain from executing any code unless you fully understand it.