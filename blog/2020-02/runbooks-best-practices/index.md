---
title: Runbooks best practices
description: This post provides a step by step template you can use to generate high quality runbooks in Octopus
author: matthew.casperson@octopus.com
visibility: private
published: 2999-01-01
metaImage:
bannerImage:
tags:
 - Octopus
---

Agile, Extreme Programming, CEBTENZZVAT, ZBGURESHPXRE (Google that last one for a laugh, although it is NSFW)… Whether you implement them or even agree with them, there is no denying that developers are always looking to optimize their workflows. And while the specific methodologies differ, they all boil down to maintaining a high velocity and high quality. To achieve that, you need automation.
The operations space hasn’t had the same love that developers have enjoyed though, leaving the operations team to hack CI servers to automate their tasks. CI servers were built to automate software development, and the ops team will always be a secondary consideration.

With Runbooks, Octopus elevates operations tasks to a first-class concept, giving ops teams a workflow designed for their needs.

In this blog post we’ll look at the best practises for designing runbooks, providing a template for ops teams to move from manual to automated workflows. As an example, we’ll create a runbook for restarting an Azure web application.

## Describe

It’s 1 AM and your pager goes off to let you know that your web site is down. The last thing you want to do is wade through pages of projects and scripts to find the one that will resolve the issue. This makes discoverability critical to runbooks.

Each Octopus project has a description field. Runbooks should take advantage of this field to document what services the runbook targets and which problems the runbook solves. The description field is searchable from the main Octopus dashboard allowing operations and support staff to find suitable projects based on keyword searches rather than a rote knowledge of all available scripts.
In the screenshot below you can see the sample project includes several keywords like “500” and “Azure Web App” that match the expected scenarios the runbook would be used in. Searching for these keywords in the Octopus dashboard returns the runbook project.

## Inspect

The first step in the runbook should attempt to inspect the current state of the system it interacts with to do two things:

1.	Capture any relevant information that may be useful in debugging the root cause. This may include things like application log files and metrics.
2.	Determine if the system is degraded. For example, checking the HTTP response codes from a web application.

For our example runbook we’ll use the `Run an Azure Powershell script` step to download the application logs, capture the resulting zip file as an artifact, and then make a HTTP request to the website to inspect the response code. The result of the HTTP call is then saved in the Octopus variable `TestResult`.

The script below captures the log file and tests the HTTP response code:

```
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.SecurityProtocolType]::Tls12;

# Redirect to a file to removeoutput printed to stderr
az webapp log download `
  --name MySalesWebApp `
  --resource-group SalesResourceGroup `
  --log-file logs.zip

New-OctopusArtifact "logs.zip"

$status = [int]([System.Net.WebRequest]::Create("https://$Hostname").GetResponse().StatusCode)
Set-OctopusVariable `
  -name "TestResult" `
  -value ($status -eq 200)

Write-Host "Web application returned HTTP status code $status"
```

## Confirm

There are many potential reasons why a web application would be down. For example, memory leaks and resource spikes can render a site unusable while still occasionally returning a valid HTTP response code.

If the inspect step failed to identify an issue, we display a prompt asking whether the runbook should continue regardless, giving support staff the opportunity to run manual tests and make their decision to proceed.

In practical terms this step is implemented as a manual intervention step that runs on the condition that the `TestResult` variable created in the last step is true (or in other words if the last step successfully contacted the web application, thus not finding a problem).

![](confirm.png "width=500")

It is worth paying attention to how often the confirmation step is presented. If it is the norm to manually proceed, it means the inspect step doesn’t accurately identify the error state of the system.

## Rectify

The rectify step is the meat of the runbook, and in our example is where the Azure web app is restarted. This is implemented with a Run an Azure Powershell script step calling the following script to restart the web app:
```
az webapp restart
```

Turning it off and on again may be a running joke in our industry, but only because it is so effective.

## Verify

The verify step is similar to the inspect step, with the exception that the checks implemented here are expected to pass thanks to the rectify step.

In our example the verify step enters a loop to check the HTTP status code over the course of 5 minutes. Once a 200 response code is returned, we consider the app to be running. The code for this step is shown below:

```
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.SecurityProtocolType]::Tls12;

for ($x = 0; $x -lt 30; ++$x)
{
  $status = [int]([System.Net.WebRequest]::Create("https://$Hostname").GetResponse().StatusCode)
  if ($status -eq 200) {
    exit 0
  }
  Start-Sleep 10
}

# We didn't get a good response in 5 mins, so we failed
exit 1
```

## Notify

It is often useful to notify other members of the team that a restart was performed. Octopus has steps for sending messages through several communications platforms, and here we have used the Slack - Send Simple Notification community step to report the status of each proceeding step.

The text below loops over the runbook steps and prints their status.
```
#{each step in Octopus.Step}
StepName: #{step}
Status: #{step.Status.Code}
#{/each}
```

![](notify.png "width=500")

## Test

If you have a broken system and an untested runbook designed to fix it, you have 2 problems.

The idea of environment progression for deployments has been a central tenant of Octopus from the beginning, and runbooks have access to all the same environments. Just as you deploy to a test environment before going to production, executing a runbook in a test environment allows the process to be validated before it is used in anger with a production outage.

## Automate

Implementing the previous steps means that your runbook accurately identifies when a system is not working as expected, rectifies the problem, and verifies that the system is back in the desired state. You will also have tested this runbook outside of the production environment. This methodology is specifically designed to instil a high degree of confidence in the runbook process.

At this point you have the ability to trigger the runbook automatically on a regular schedule. To do this, the confirm step is disabled or removed, use the run condition `#{unless Octopus.Action[Inspect].Output.TestResult}true#{/unless}` for the remaining steps, and a scheduled trigger is created to execute the runbook as needed.

![](triggers.png "width=500")

## Summary

Whatever methodology you ascribe to, achieving high velocity and high quality demands automation. The steps outlined in this post are designed to produce a runbook that can be confidently run automatically.

Reliably identifying when a system is not in a desired state, rectifying the problem, verifying the fix, and validating the entire process in a test environment ensures your runbooks have the same quality as your software deployments.