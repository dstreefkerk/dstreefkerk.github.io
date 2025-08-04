---
layout: post
title: "Parsing JSON Data with PowerShell: From Raw API Responses to Structured Reports"
date: 2025-08-04
categories: [powershell, automation, data-analysis]
tags: [powershell, json, data-analysis, api, automation, pscustomobject]
author: "Daniel Streefkerk"
excerpt: "How to use PowerShell to parse and analyse JSON data from APIs and exports, transforming complex nested structures into structured reports ready for analysis."
---

# Parsing JSON Data with PowerShell: From Raw API Responses to Structured Reports

As a long-time PowerShell fan, it's usually the first tool that I reach for when I need to work with JSON data. PowerShell's native JSON handling capabilities make it an excellent tool for quickly parsing, analysing, and transforming whatever data you find yourself working with on a given day.

## The Scenario: Multiple JSON Export Files

On this occasion, I needed to analyse a batch of JSON exports from a Logic App that was triggering via a Microsoft Sentinel automation. 

Each file contained detailed incident information including related entities, alerts, and metadata, and I needed to extract a summarised list of that information to see what I was working with.

This pattern is common across many systems: getting multiple JSON files from API calls, automated exports, or logging systems. Rather than manually reviewing each file or attempting complex Excel imports, PowerShell provided a quick and efficient solution, and I thought I'd better blog about it to share the knowledge.

## Loading and Parsing JSON Files

The first step is getting all those JSON files into PowerShell. Here's the approach:

```powershell
# Define the folder containing JSON exports
$folder = "C:\DataExports"

# Create an array to hold all JSON data
$allJSON = @()

# Load and parse all JSON files in one go
$allJSON = Get-ChildItem $folder -Filter "*.json" | 
    ForEach-Object { Get-Content $_ | ConvertFrom-Json }
```

PowerShell's `ConvertFrom-Json` cmdlet automatically transforms the JSON text into PowerShell objects, making the data immediately accessible for analysis. Each JSON file becomes a fully navigable object structure in memory.

The ```$_``` notation in ```ForEach-Object``` simply represents the current object that's being processed in each iteration. In the example above, we're listing all of the files, then getting the content of each file and converting it to PowerShell objects from JSON.

## Exploring the Data Structure

Once loaded, you can explore the JSON structure interactively. This is where PowerShell shines - you can navigate complex nested objects with simple dot notation:

```powershell
# View the first object
$allJSON[0]
# or
$allJSON | Select-Object -First 1

# Drill into properties
$allJSON[0].properties

# Access nested data
$allJSON[0].properties.title
# Output: "ACMAU-UsersLogonFromOutsideAustralia involving one user"

# Find related entities
$allJSON[0].properties.relatedEntities
```

The beauty of this approach is that PowerShell automatically converts JSON objects into a PowerShell object that you can explore interactively. Tab completion even works on the property names, making discovery intuitive. For more information, you can use Get-Member and Format-List

```powershell
# View the object metadata - properties, methods
$allJSON | Get-Member

# View all of the properties of the object
$allJSON | Format-List
```

## Filtering and Extracting Specific Data

One of the most powerful aspects of PowerShell is its ability to filter and extract data from complex structures. For instance, when working with our Sentinel data, we can extract all user accounts involved across all incidents:

```powershell
# Get all Account entities across all incidents
$allAccounts = $allJSON.properties.relatedEntities | Where-Object { $_.kind -eq "Account" }

# Extract just the account names
$accountNames = $allAccounts | ForEach-Object { $_.properties.accountName }
```

This pattern works universally - whether you're extracting error messages from log files, user IDs from API responses, or configuration values from system exports.

## Transforming Data into Structured Reports with PSCustomObject

The real value comes when you transform the raw JSON into a structured format that's easy to analyse and share. This is where PowerShell's `[PSCustomObject]` type accelerator becomes invaluable.

### What's this [PSCustomObject] thing?

`[PSCustomObject]` is a type accelerator in PowerShell that creates custom objects with properties you define. When you prefix a hashtable with `[PSCustomObject]`, PowerShell converts it into a proper object with named properties. This is much cleaner and more efficient than the older `New-Object` approach in previous versions of PowerShell.

Here's a simple example:

```powershell
# Creating a custom object from a hashtable
$person = [PSCustomObject]@{
    Name = "John Doe"
    Age = 30
    Department = "IT"
}

# Now you can access properties with dot notation
$person.Name  # Returns "John Doe"

# Or view the new object
PS C:\> $person

Name     Age Department
----     --- ----------
John Doe  30 IT
```

One of the best things about PowerShell is that everything's surfaced as an object. Even the output of ```Get-Process``` or ```Resolve-DnsName```, or even ```Get-History```.

If you're curious to read more about why PowerShell works the way it does, have a read of the [Monad Manifesto](https://github.com/MicrosoftDocs/PowerShell-Docs/blob/main/assets/MonadManifesto.pdf) by Jeffrey Snover, the legendary inventor of PowerShell.

### Building Structured Reports

If we apply this PSCustomObject technique to our JSON data, we can create a clean summary report:

```powershell
$results = @()

foreach ($item in $allJSON) {
    $incident = $item.properties

    # Extract the primary user entity
    $userEntity = $incident.relatedEntities | 
        Where-Object { $_.kind -eq "Account" } | 
        Select-Object -First 1

    # Build a custom object with the data we need
    # The [PSCustomObject] decorator transforms our hashtable
    # into a proper object with strongly-typed properties
    $results += [PSCustomObject]@{
        IncidentId = $item.name
        IncidentNumber = $incident.incidentNumber
        Title = $incident.title
        Severity = $incident.severity
        Status = $incident.status
        CreatedTime = $incident.createdTimeUtc
        UserName = $userEntity.properties.accountName
        UPN = $userEntity.properties.additionalData.UserPrincipalName
        DomainName = $userEntity.properties.additionalData.DomainName
        AADUserId = $userEntity.properties.aadUserId
    }
}
```

The magic here is that `[PSCustomObject]` creates objects that PowerShell understands natively. These objects work seamlessly with cmdlets like `Export-Csv`, `Format-Table`, and `Where-Object`. Without the `[PSCustomObject]` decorator, you'd just have a hashtable, which doesn't format as nicely and lacks some of the object-oriented features.

## Viewing and Exporting Results

PowerShell provides many ways to view and export your processed data. Here are a few of them:

```powershell
# Display as a formatted table
$results | Format-Table -AutoSize

# Export to CSV for further analysis
$results | Export-Csv "C:\temp\data-summary.csv" -NoTypeInformation

# Group and summarise
$results | Group-Object Severity | Select-Object Name, Count

# Filter for specific conditions
$results | Where-Object { $_.Severity -eq "Medium" }
```

Because we used `[PSCustomObject]`, our data exports cleanly to CSV with proper column headers matching our property names. Each property becomes a column, and each object becomes a row.

## Tips for Working with JSON in PowerShell

### Handling Large Files

For large JSON files, consider loading the entire file as a single string first using the -Raw switch:

```powershell
$json = Get-Content -Path "large-file.json" -Raw | ConvertFrom-Json
```

### Dealing with Arrays

When you're not sure if a property will be an array or single object:

```powershell
$items = @($object.property)  # Force array even if single item
```

### Null-Safe Navigation

Protect against missing properties (PowerShell 7+):

```powershell
$userName = $userEntity.properties.accountName ?? "Unknown"
```

### Pretty-Printing JSON

When you need to output JSON again:

```powershell
$results | ConvertTo-Json -Depth 10 | Out-File "output.json"
```

## Common Patterns for Complex Data

When working with nested JSON data, certain patterns reoccur regardless of the actual data set:

```powershell
# Extract unique values across all objects
$uniqueValues = $allJSON.properties.someArray | 
    Select-Object -ExpandProperty propertyName -Unique

# Find objects matching multiple criteria
$filtered = $allJSON | 
    Where-Object { 
        $_.properties.status -eq "Active" -and 
        $_.properties.priority -in @("High", "Critical") 
    }

# Flatten nested structures
$flattened = $allJSON | ForEach-Object {
    $parent = $_
    $_.properties.children | ForEach-Object {
        [PSCustomObject]@{
            ParentId = $parent.id
            ParentName = $parent.name
            ChildId = $_.id
            ChildValue = $_.value
        }
    }
}
```

## Conclusion

Because PowerShell is built on top of the .NET Framework, PowerShell's native JSON handling capabilities make it an useful tool for data processing and automation. 

Once you learn to use these patterns, dealing with API responses, log exports, or configuration files will be a cinch due to PowerShell's ability to quickly parse, filter, and transform JSON data. 

The other major benefit is that PowerShell is built into every modern Windows system, so you're not having to request admin access to install something like Python.

## Further Reading

Much ink has been spilled, or many pixels darkened rather, about PowerShell and the topics I've only just briefly touched on above. Here's a collection of links in case you'd like to learn more:

### Official Microsoft Documentation

- **[ConvertFrom-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json)** - Microsoft's official documentation on the ConvertFrom-Json cmdlet, covering parameters like `-AsHashtable`, `-Depth`, and `-NoEnumerate` introduced in PowerShell 6+
- **[ConvertTo-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json)** - Detailed reference for converting PowerShell objects to JSON, including handling of depth levels and special characters
- **[Everything You Wanted to Know About PSCustomObject](https://learn.microsoft.com/en-us/powershell/scripting/learn/deep-dives/everything-about-pscustomobject)** - Comprehensive guide to PSCustomObject, covering creation methods, property management, and advanced techniques
- **[about_PSCustomObject](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pscustomobject)** - Explains the differences between `[psobject]` and `[pscustomobject]` type accelerators
- **[Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)** - Complete reference for working with REST APIs in PowerShell

### Practical Tutorials and Guides

- **[PowerShell and JSON: A Comprehensive Guide](https://adamtheautomator.com/powershell-json/)** - Excellent tutorial covering JSON parsing, API interactions, and practical examples using Visual Studio Code
- **[How to Query REST APIs with PowerShell: A Practical Guide](https://adamtheautomator.com/powershell-rest-api-tutorial/)** - Step-by-step guide to building reusable API query tools with error handling
- **[Calling a REST API from PowerShell](https://4bes.nl/2020/08/23/calling-a-rest-api-from-powershell/)** - Practical examples of authentication, headers, and working with different HTTP methods
- **[PowerShell: Working with JSON & APIs Using Invoke-RestMethod](https://myitforum.substack.com/p/powershell-working-with-json-and)** - Covers pagination, authentication mechanisms, and performance optimization

### PSCustomObject Deep Dives

- **[How to Use Custom Objects (PSCustomObject) in PowerShell](https://www.sharepointdiary.com/2021/08/custom-objects-in-powershell.html)** - Comprehensive guide with practical examples and common use cases
- **[PowerShell: Getting Started - Creating Custom Objects](https://www.gngrninja.com/script-ninja/2016/6/18/powershell-getting-started-part-12-creating-custom-objects)** - Beginner-friendly tutorial showing multiple methods for creating custom objects
- **[Fun With PowerShell Objects – PSCustomObject](https://arcanecode.com/2022/01/10/fun-with-powershell-objects-pscustomobject/)** - Part of a series exploring object-oriented programming in PowerShell

### Quick References

- **[Convert JSON with PowerShell ConvertFrom-Json and ConvertTo-Json](https://4sysops.com/archives/convert-json-with-the-powershell-cmdlets-convertfrom-json-and-convertto-json/)** - Clear examples of JSON conversion workflows
- **[PowerShell ConvertFrom-Json Tutorial](https://zetcode.com/powershell/convertfrom-json/)** and **[PowerShell ConvertTo-Json Tutorial](https://zetcode.com/powershell/convertto-json/)** - Concise tutorials with practical code examples

### Community Resources

- **[Parsing JSON with PowerShell](https://techcommunity.microsoft.com/blog/coreinfrastructureandsecurityblog/parsing-json-with-powershell/2768721)** - Microsoft Tech Community post covering version differences between PowerShell 5.1 and 7+
- **[PowerShell subreddit](https://www.reddit.com/r/PowerShell/)** - Active community for PowerShell questions and discussions
- **[PowerShell.org Forums](https://forums.powershell.org/)** - Community forums for in-depth PowerShell discussions