# Semantic Versioning steps in Jenkins pipeline
---
This is my Groovy interpretation of a Jenkins pipeline with Semantic Versioning for a git repository. At this point, the pipeline only do very a handful of things:

- Read the latest tag
- Read the commit type
- Bump the version number (or not) in accordance with the commit type
- Tag and push the new version to the git origin

## Background
There are quite a lot of Action plugins for Semantic Versioning on GitHub Actions, which get you the current and next version automatically. Sadly, this wasn't the case for Jenkins. I have only seen [one Jenkins plugin](https://plugins.jenkins.io/semantic-versioning-plugin/) that has something to do with Semantic Versioning. From the description, it seems to have a few limitations:

- It requires a text file to read the version number and does not edit the version string automatically after building.

- It support apps that uses POM/SBT, not general Git repos

- It has not been updated since 2023

There seems to be no other Jenkins plugin/extension that do the SemVer stuff, and there were little to no blogs/forum answers for this matter. Therefore, I've decided to write my own Jenkins pipeline and post it here for later reuses. I also hope that someone may come accross this and either see it as a solution, or see the mistakes and point 'em out. I hope, like some have said, a wrong answer will be a faster path to the right answer than a question.

## My Versioning scheme

The version string format will be `<major>.<minor>.<patch>` for releases and `<major>.<minor>.<patch>-<build time>` or `<major>.<minor>.<patch>-<CustomString>` for pre-released.

The version numbers will be bumped depending on the type of commit:

- Breaking Change: Increments the major number and reset minor and patch to 0

- Feat: Increments the minor number and reset patch to 0

- Fix: Increments the patch number

- Pre-release: Does not increment the major, minor, nor patch, but will add a hyphen and an extra string (either defined by the developer or the build time will be assigned automatically).

## Code explanation

My