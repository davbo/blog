---
title: "Disclosing a security issue upstream"
date: 2017-04-03T23:52:30Z
draft: false
author: 
 - Dave King while working at GDS
summary: While working at GDS I went through the process of responsible disclosure with an upstream open source project.
---

This post was originally published on the [GDS technology blog](https://gdstechnology.blog.gov.uk/2017/04/03/disclosing-a-security-issue-upstream/).

Building services using open source software makes you part of the open source community. As a community member, you are responsible for handling any security issues you identify. This post provides an example of a security issue we encountered and resolved by working with the project security team.

## Discovering an issue

On the [GOV.UK PaaS (Platform as a Service) team](https://governmentasaplatform.blog.gov.uk/category/platform-as-a-service/) we manage and deploy the open source technology [Cloud Foundry](https://www.cloudfoundry.org/). In the course of this work we identified and reported a possible security issue [upstream](https://en.wikipedia.org/wiki/Upstream_(software_development)) to the CloudFoundry maintainers.

One of our engineers identified that [private key](https://www.techopedia.com/definition/16135/private-key) material was being logged by one of the CloudFoundry components. The key was used only if CloudFoundry was configured to federate identity using the [SAML (Security Assertion Markup Language) protocol](https://en.wikipedia.org/wiki/Security_Assertion_Markup_Language). We don’t actually use this feature but we recognise logging the private key is definitely an issue.

CloudFoundry provides a [single point of contact](https://www.cloudfoundry.org/security/) email address for reporting possible vulnerabilities within their projects. After informing the security team at GDS, we notified CloudFoundry using this address.

Our email provided a detailed description of the issue and how we found it. In much the same way as any other bug report, our report noted what steps produced the issue, the behaviour we observed as a result (so the logging of the private key) and the behaviour we would have expected instead.

## Working together to resolve the issue

The response from CloudFoundry informed us they were treating the problem as a security incident. After we investigated further we provided the CloudFoundry security team with details on the cause of the issue (an overly verbose logging function) in the User Account and Authentication service.

## Public disclosure

As this issue affected other users of the CloudFoundry projects a CVE (Common Vulnerabilities and Exposures) was assigned: [CVE-2016-6659](https://www.cloudfoundry.org/cve-2016-6659/). Ahead of the public disclosure we were asked if GDS could be attributed with discovery of the issue, which we were happy to allow. CloudFoundry then publicly disclosed the issue and fixed it with the [release of CloudFoundry version 248](https://github.com/cloudfoundry/cf-release/releases/tag/v248).

The process itself was straightforward and the responses from CloudFoundry was friendly and clear.

## Summary

It was great working together with our engineers at GDS and the security team at CloudFoundry through this process to resolve the issue. I hope this post shows how working with open source projects to resolve security issues does not have to be difficult, and improves open source software for all users.

Where government systems are concerned we’re pleased to see the National Cyber Security Centre (NCSC) launching a [vulnerability co-ordination pilot](https://www.ncsc.gov.uk/blog-post/vulnerability-co-ordination-pilot) for handling vulnerability disclosure for open and closed source government systems.

If you’d like to know more about handling vulnerability disclosure we recommend following the [ISO 29147](http://standards.iso.org/ittf/PubliclyAvailableStandards/index.html) standard.
