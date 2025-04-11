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

My pipelines includes 5 stages, with the first being optional.

### Import the class and define the global variables

Since the tag will include the date and time string in certain cases, we need a date string format class.

```groovy
import java.text.SimpleDateFormat
```

Define a few variables

```groovy
// version string v<major>.<minor>.<patch>-<CustomString/BuildTime>
def majorVer = 0
def minorVer = 0
def patchVer = 0
def devver = null

boolean isPreRelease = false // define if the version is a release or a pre-release
boolean buildNewVerson = false // whether to build a new version or not

def versionMessage="No additional Message" // the tag message, which is the value of the "-m" option of the "git tag" command
```

### (Optional) Checkout:

```groovy
stage('Checkout Code') {
    steps {
        git branch: 'main', credentialsId: 'kira17-git', url: 'http://192.168.69.11:8081/user/repo'
    }
}
```
This particular stage might not be necessary if you get the Jenkins pipeline from the Jenkinsfile in the SCM, as it will do the checkout step before executing the pipeline.

### Set git identity:

```groovy
stage('Configure Git Identity') {
    steps {
        sh 'git config user.name "kira sev entine"'
        sh 'git config user.email "kirazero17@anywhere.com"'
    }
}
```
You can also set the name and e-mail in the local settings. However, if you want a different identity for this pipeline, you may still want to keep this stage.

### Get the current version number

```groovy
stage("Git: Get current version"){
    steps{
        script{
            def tag = sh(returnStdout: true, script: "git tag --sort version:refname | tail -1").trim()
            if (tag != null)
            {
                println "Last version: $tag"
                (majorVer, minorVer, patchVer) = tag.tokenize('v|\\.|-')
                majorVer = majorVer as Integer
                minorVer = minorVer as Integer
                patchVer = patchVer as Integer
                println "Major Version: $majorVer"
                println "Minor Version: $minorVer"
                println "Patch Version: $patchVer"
            }
        }
    }
}
```

The command `git tag --sort version:refname | tail -1` will return the latest tag, which is the last on the full list (hence it's piped with the `tail` command)

As the version string is `v<major>.<minor>.<patch>-<CustomString/BuildTime>` we use `v`, `.`, and `-` as tokens to split the string, using the first 3 substrings as the major, minor, and patch number respectively.

### Get next verstion number

```groovy
stage("Git: Get next Verstion"){
            steps{
                script{
                    def commit = sh(returnStdout: true, script: "git log -1 --format=%s").trim()
                    def commitType = commit.tokenize(':')[0]
```

A conventional commit message should have the following format:

**\<Commit type\>: \<Commit message\>**

Therefore, everything before the colon can be split out and stored into a variable, which will later be compared with a set of pre-defined commit type to bump the version correctly.

```groovy
switch(commitType.toLowerCase()){
    case "fix":
        patchVer++
        buildNewVerson = true
        break
    case "feat":
        minorVer++
        patchVer = 0
        buildNewVerson = true
        break
    case "breaking change":
        majorVer++
        minorVer = 0
        patchVer = 0
        buildNewVerson = true
        break
    case "pre-release":
        buildNewVerson = true
        isPreRelease = true
        break
    default:
        println "No version bump."
}
```

Here, the `string.toLowerCase()` method is used to convert the commit type to a fully lowercased one. This make sure the commit type is case-insensitively compared, removing the concern for case when writing commit messages.

```groovy
if (buildNewVerson == true) {
    if (isPreRelease == true) {
        if (devver == null) {
            def date = new Date()
            def sdf = new SimpleDateFormat("yyyyMMddHHmmss")
            devver = sdf.format(date)
        }
        devver = "-${devver}"
    }
    else {
        devver = null
    }
}
```

The dev version (`devver`) is used for pre-release commits. It will append the date and time in the **yyyyMMddHHmmss** format if nothing is defined. Otherwise, the dev version should be empty.

### Tag and push the repo to its origin

```groovy
stage("Git: Tag and push the next Verstion"){
    steps{
        withCredentials([gitUsernamePassword(credentialsId: 'kira17-git', gitToolName: 'Default')]){
            script {
                println "Next version: v${majorVer}.${minorVer}.${patchVer}"
                    sh """git tag -a v${majorVer}.${minorVer}.${patchVer}${devver} -m \"Release version ${majorVer}.${minorVer}.${patchVer}${devver} - ${versionMessage}\"
                    git push origin v${majorVer}.${minorVer}.${patchVer}${devver}"""
            }
        }
    }
}
```

It requires a git credential, saved as a username-password pair of secret. You can replace the `credentialsId` value to the ID of your own credential.

## End note

That's all you need to know about the pipeline. I may improve it a bit soon, and there will be a hands-on instruction on how to set this up.

Stay tuned !