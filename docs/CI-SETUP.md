# Configuring your repo for Jenkins CI

This document contains information on how to set up Jenkins CI for your repo.

## Overview

The CI systems located at https://ci.dot.net and https://ci2.dot.net serve a large number of projects in the .NET Foundation and Microsoft.  It is a fairly standard Jenkins instance running in Azure with a variety of execution nodes including:
  * Windows Server 2012 2
  * Ubunutu 14.04
  * Centos 7.1
  * OpenSUSE 13.2
  * Mac OSX
  * FreeBSD

### What can Jenkins do?

In the most basic sense, Jenkins is a task scheduling and management system.  Given a set of jobs that need to be run based on certain triggers (e.g. GitHub pushes or pull requests), run those jobs on a specific pool of machines.  While tons of those sort of systems have been created over the years, the great thing about Jenkins is its flexibility and plugin model.  There are thousands of plugins for Jenkins which automate tasks that we would normally need to deal with ourselves.

### How does the .NET CI model work?

While Jenkins does have a rich UI for configuring jobs, maintaining a large number of jobs this way is inefficient, cumbersome and prone to error.  Furthermore, it's difficult to version and backup the system when it is configured like this.  Luckily, Jenkins has a plugin that solves just this problem.  The Job DSL plugin enables the creation of jobs (as well as other types of Jenkins assets) using a Groovy-based DSL (domain specific language).  See https://github.com/jenkinsci/job-dsl-plugin for more information.

Here is the lifecycle of .NET CI.

  1. A project is created in Jenkins that watches the dotnet-ci repo for changes.
  2. When it changes, it runs MetaGenerator.groovy.
  3. MetaGenerator reads the list of repos under CI control from (jobs/data/repolist.txt).
  4. For each repo, MetaGenerator generates a new folder for that project and a generator.  The generator watches the repo's netci.groovy file in the root of the master branch for changes, as well as changes to dotnet-ci (which contains Utility functionality)
  5. MetaGenerator also generates a GenPRTest folder underneath the project folder and a generator in that folder.  Functionally, that generator is equivalent to the commit one, except it watches for PR tests in the project repo.  New jobs are generated relative to the GenPRTest folder and are immediately disabled afterward.
  6. PRs to the project can be commented with "test ci please" to do a test generation.  While this doesn't execute any of the generated projects, this verifies that the file is syntactically correct, and new jobs can be examined for correctness.
  7. Upon commit, the primary generator runs and modifies/adds jobs.  Existing jobs that are removed (were generated by the previous run but not by this one) are disabled.  Jobs that are currently being run are updated in place, so occasionally there may be some unexpected side effects.

## Onboarding steps for your project

Below contains information on how to onboard your project onto Jenkins.

  1. Determine a set of build/test commands that can be executed to validate the state of your project or a PR.  Most projects encapsulate this 'inner loop' logic in a script in the root of the repository.  This makes execution on a developer machine easy.
  2. Determine your prerequisite software.  This should be simple where possible.  Most .NET projects have hard install dependencies on one or more of the products listed below.  On Unix family operating systems, additional packages for building native components can be installed if necessary.
    * Visual Studio 2013/2015 Community edition
    * CMake (for building cross platform native code)
    * Python
    * Powershell
    * Bash/Shell scripting
  Let the .NET CI administrators know what will be needed (@mmitche, netciadmin alias)
  3. Send a PR to dotnet-ci adding your repo to jobs\data\repolist.txt.  The server (dotnet-ci or dotnet-ci2) is specified in the line.  Typically dotnet-ci is used, though newer repos may use dotnet-ci2.
  4. Ensure your repo is accessible by @dotnet-bot and @mmitche.
  5. Configure web hooks for the CI.  You need two entries:
    * A GitHub webhook for push events - Go into the repo settings, click "Webhooks", then click "Add webhook".
        - Payload URL: https://ci.dot.net/github-webhook/ (For projects on dotnet-ci2, use https://ci2.dot.net/github-webhook/)
        - Content type: application/x-www-form-urlencoded
        - "Just send me the push event"
    * A GitHub webhook for pull request events - Go into the repo settings, click "Webhooks", then click "Add webhook".
      - Payload URL: https://ci.dot.net/ghprbhook/ (For projects on dotnet-ci2, use https://ci2.dot.net/ghprbhook/)
      - Content type: application/x-www-form-urlencoded
      - "Let me select individual events"
        - Pull request
        - Issue comment
  6. Create a file called netci.groovy in root of your repo in the target branch (this could also be named something different based on the line in the repolist.txt file).
  7. [Write your CI definition](WRITING-NETCI.md)
  8. PR the netci.groovy file, /cc @mmitche for review and comment "test ci please" to the PR thread.
  9. Once the test generation completes, you may examine the jobs for correctness by clicking on the Details link of the job.
