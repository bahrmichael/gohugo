---
title: "AWS Cost Tagging"
date: 2018-10-07T11:39:29+02:00
publishdate: 2018-10-07T11:39:29+02:00
image: "/images/blog/7.jpg"
tags: ["interesting", "aws", "serverless", "testing"]
comments: false
draft: true
---
# Motivation
AWS has a fine grained pricing model, charging you only for what you need. Sometimes down to a per-request basis. This turns out to become rather complex though. The AWS Cost Explorer gives you some insight on which products cost you how much.

_picture of cost explorer grouped by services_

That's all good when you run one or two projects. However when you or your organisation is running 10s or 100s of projects you need more granularity to understand where the expensive stuff is.

# Tags

With tags you assign key value pairs to most of AWS' products. You can for example add `Project:project_a` and `Table:project_a_table_1` to a DynamoDB table as well as `Project:project_a` and `Lambda:project_a_function_1` to a Lambda function.

_picture of how it will look like_

## Manual Approach
- show where to manually insert then with the AWS Console

## Infrastructure as Code Approach
- show where to insert them with serverless

- mention serverless tagging hierarchy (e.g. tag a whole stack)

# Cost Explorer Does Not Collect Tags By Default

 - add warning market, DO NOT SKIP THIS STEP
In good AWS fashion it is not activated by default. You have to manually go into the cost explorer and activate tag tracking, wait a day or two and then you will be able to drill down by tags.

_picture slide of how to activate it_

# Tagging Strategy

My experience with granularity
* department level
* project level
* resource within project level
optional
* business critical/non-critical
* use case