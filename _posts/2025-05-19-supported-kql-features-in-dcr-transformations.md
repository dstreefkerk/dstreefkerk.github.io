---
layout: post
title: "Supported KQL Features in Azure Monitor Data Collection Rule (DCR) Transformations"
date: 2025-05-19
categories: [cloud, azure, sentinel, monitoring]
tags: [kql, azure-monitor, dcr, transformations, log-analytics, reference]
author: "Daniel Streefkerk"
excerpt: "A comprehensive reference guide to permitted and blocked KQL functions and operators in Azure Monitor Data Collection Rule transformations."
---

# Supported KQL Features in Azure Monitor Data Collection Rule (DCR) Transformations

While working on a [Codeless Connector](https://learn.microsoft.com/en-us/azure/sentinel/create-codeless-connector) implementation recently, as well as a CSV-based log ingestion project for a different client, I found myself repeatedly searching for the list of KQL functions permitted in Data Collection Rule transformations. If you've ever tried to use a function only to have the deployment fail with a cryptic error about "unsupported operators", you'll understand the frustration. DCRs only support a very limited subset of the full KQL language features.

I've compiled this reference list of all explicitly permitted (and blocked) KQL features for DCR transformations based on [Microsoft's official documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/data-collection-transformations-kql). This should save you some time when crafting those transformation queries. 

> You can also feed this info to a LLM to help you generate accurate KQL, grab the raw post from my site's repo [here](https://raw.githubusercontent.com/dstreefkerk/dstreefkerk.github.io/refs/heads/main/_posts/2025-05-19-supported-kql-features-in-dcr-transformations.md).

## Key Limitations to Remember

Before diving into the details, there are a few critical constraints to keep in mind:

- The `parse` operator may output **no more than 10 columns** per statement
- Only functions and operators explicitly listed by Microsoft are allowed
- Multi-table semantics operators (`join`, `summarize`, etc.) are disallowed for DCRs
- Microsoft's [official documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/data-collection-transformations-kql) is the source of truth (last updated 9 December 2024)

I've bumped into all of these limitations at one point or another, with the 10-column limit for `parse` being particularly frustrating on CEF log parsing projects.

Let's get into what's permitted and what's not:

## Operators & Statements

|Item|Permitted|Notes|
|---|---|---|
|`source`|**Yes**||
|`print`|**Yes**||
|`let`|**Yes**||
|`where`|**Yes**||
|`extend`|**Yes**||
|`project`, `project-away`, `project-rename`|**Yes**||
|`parse`|**Yes**|≤ 10 output columns|
|`datatable`|**Yes**||
|`columnifexists`|**Yes**||
|**Blocked operators** (`summarize`, `join`, `union`, `mv-expand`, `mv-apply`, `top`, `sort`, `invoke`, `scan`, `partition`, `distinct`, `project-reorder`, `parse_csv`)|**No**|Not in supported list|

## Scalar & Conditional Functions

|Function Group|Functions|Permitted|
|---|---|---|
|**Type conversion**|`tostring`, `toint`, `tolong`, `todouble`/`toreal`, `todatetime`, `totimespan`, `tobool`, `toguid`|**Yes**|
|**Maths & Rounding**|`abs`, `round`, `floor`/`bin`, `ceiling`, `log`, `log10`, `log2`, `exp`, `exp10`, `exp2`, `pow`, `sign`|**Yes**|
|**Bitwise**|`binary_and`, `binary_or`, `binary_not`, `binary_xor`, `binary_shift_left`, `binary_shift_right`|**Yes**|
|**Conditional**|`iif`, `case`, `max_of`, `min_of`|**Yes**|
|**Diagnostics**|`isnull`, `isnotnull`, `isfinite`, `isinf`, `isnan`, `gettype`|**Yes**|
|**Blocked**|`coalesce`|**No**|

## Datetime / Timespan Functions

|Function Category|Functions|Permitted|
|---|---|---|
|**Current / relative**|`now`, `ago`|**Yes**|
|**Start / End helpers**|`startofday`, `endofday`, `startofweek`, `endofweek`, `startofmonth`, `endofmonth`, `startofyear`, `endofyear`|**Yes**|
|**Parts & Maths**|`datetime_add`, `datetime_diff`, `datetime_part`, `hourofday`, `dayofweek`, `dayofmonth`, `dayofyear`, `weekofyear`, `getmonth`, `getyear`, `make_datetime`, `make_timespan`|**Yes**|

## String Functions

|Function Category|Functions|Permitted|
|---|---|---|
|**Building & Slicing**|`strcat`, `strcat_delim`, `substring`, `strlen`, `split`|**Yes**|
|**Casing & Search**|`tolower`, `toupper`, `indexof`, `extract`, `extract_all`, `countof`, `hash_sha256`|**Yes**|
|**Base64**|`base64_encodestring`, `base64_decodestring`|**Yes**|
|**Blocked**|`replace_string`, `replace_regex`|**No**|

## Dynamic / Array Functions

|Function|Permitted|Notes|
|---|---|---|
|`parse_json`, `parse_xml`|**Yes**||
|`pack`, `pack_array`, `array_length`, `array_concat`, `zip`|**Yes**||
|`bag_keys`, `bag_values`, `bag_set`, `bag_remove_keys`|**No**||

## Special Functions

|Function|Permitted|Notes|
|---|---|---|
|`geo_location`|**Yes**|Can increase latency – keep usage minimal|
|`parse_cef_dictionary`|**Yes**||

## A Word of Caution

When building complex DCR transformations, I've found it pays to test each component incrementally. It's quite frustrating to discover after deployment that your carefully crafted KQL is using an unsupported function. What's worse is that sometimes the error messages aren't particularly helpful.

The simplest way to test is by using the DCR editing GUI in the Azure Portal. Otherwise, you can run test deployments of ARM templates that contain the KQL, however that requires a bunch of escaping of characters, etc.

If you're working with CEF logs or other complex formats, remember that the 10-column limit for `parse` can be a serious constraint. You may need to chain multiple parse operations or use alternative methods to extract all the fields you need.

I hope this reference helps you avoid some of the headaches I've encountered.