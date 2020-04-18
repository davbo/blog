---
title: Building a tool to improve our Github security
date: 2018-07-13T00:06:40Z
draft: false
author: 
 - Dave King while working at GDS
summary: While at GDS I built a simple tool to capture an audit of activity in Github and record it into Amazon S3 for querying with Amazon Athena.
---

This post was originally published on the [GDS technology blog](https://gdstechnology.blog.gov.uk/2018/07/13/building-a-tool-to-improve-our-github-security/).

GitHub plays a major role in the software supply chain at GDS. All our source code is stored in GitHub - mainly in [Alphagov](https://github.com/alphagov) - and we work hard to make sure our repositories are secure.

At GDS, we strengthen username and password authentication by requiring users to set up [2-factor authentication](https://help.github.com/articles/requiring-two-factor-authentication-in-your-organization/). Here’s why the Security Engineering team created an audit tool to log all GitHub events in GDS code repositories.

## Why we started our audit log

In May 2017, we were contacted by a security researcher who had discovered GitHub personal access tokens in Travis-CI log output. Travis-CI is a hosted [Continuous Integration](https://en.wikipedia.org/wiki/Continuous_integration) service which many teams at GDS use to automate the build and testing of their applications. Travis-CI [disclosed this data breach publicly on their blog](https://blog.travis-ci.com/2017-05-08-security-advisory).

Some of the tokens belonged to a member of Alphagov and this was a serious concern to us as these tokens behave like GitHub credentials without the 2-factor requirement.

We wanted to check if the leaked tokens had been used maliciously so we contacted GitHub support who supplied us with use logs of recent tokens. Unfortunately, the use logs GitHub provided were limited to a few weeks.

To search for events further back in time we identified the [GitHub Archive](https://www.githubarchive.org/) project as a way to do this for all public GitHub events with years of history available. Using [Google BigQuery](https://cloud.google.com/bigquery/), it was easy for us to write SQL-like queries against the GitHub Archive data. This gave us the confidence that any exposed tokens hadn’t been used maliciously.

## Creating an audit log of GitHub activity

Handling this incident showed us the importance of having a strong audit log of activity in GitHub, similar to the one we have in Amazon Web Services with the CloudTrail service. We created our own audit log after being inspired by the public [Github Archive](https://www.githubarchive.org/)’s use of [Google BigQuery](https://cloud.google.com/bigquery/).

We now capture all events that take place in Alphagov for both public and private repositories using GitHub’s [Webhooks](https://developer.github.com/webhooks/) feature. The Webhook delivers GitHub events to a web service, which writes the events into the Amazon Web Services (AWS) S3 object store. We have done this in the open and have published our [Terraform configuration](https://github.com/alphagov/archive-github-events) which sets up the web service and S3 bucket.

## Using our audit log

Recently a colleague in GDS noticed a token used to [configure a dashboard](https://github.com/alphagov/fourth-wall) in our office (which only needed read-only access) had more privileges than necessary. This issue was exactly the type of situation we we created the audit log for.

We used [Amazon Athena](https://aws.amazon.com/athena/) to check the usage of the token. This AWS service uses the open source [Presto](https://prestodb.io/) database engine to query S3 objects. We were able to write SQL queries against our audit S3 bucket to check how the token was used. We were able to quickly eliminate any possibility that anyone had used the token maliciously.

Following this incident a colleague made [some improvements](https://github.com/alphagov/fourth-wall/commit/aade6fc524d3889a24de3f38cdba63d710295bfe) to the GitHub dashboard to make these kinds of issues less likely. Now the dashboard has better documentation about the read-only scopes required. This means users are warned when personal access tokens with write access are used.

By using our audit log for GitHub events in Alphagov, we can see if any malicious action has taken place and view any details. In the future, we may develop an automated audit tool so we can be alerted to any possible malicious activity.
