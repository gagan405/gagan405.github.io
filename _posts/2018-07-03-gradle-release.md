---
layout: post
title: Gradle release versioning
subtitle: Versioning per branch
---

There is this awesome [release plugin](https://github.com/researchgate/gradle-release) which takes care of the usual stuff and more, like checking uncommitted files, checking if commit is pushed or not, and adding release tag etc.

You can also customize the release from a specific branch.

But then, it still doesn't support versioning per branch. In cases, where you want to have some snapshot (or any other classifier) release from feature branches, and want to keep that separate from main RELEASE versions.

So, here is a quick and dirty knock around I did to make it work :

`gradle.properties` :

~~~
masterVersion=1.0
snapshotVersion=1.10
version=1.10.SNAPSHOT
~~~

Here, `masterVersion` is used to track the release versions from master branch and `snapshotVersion` for any other branch.

We need to know which branch we are in :

~~~groovy
allProjects {
  ext.branch = gitBranch()
  project.version = getCustomVersion(false)
}

def gitBranch() {
    def branch = ""
    def proc = "git rev-parse --abbrev-ref HEAD".execute()
    proc.in.eachLine { line -> branch = line }
    proc.err.eachLine { line -> println line }
    proc.waitFor()
    branch
}
~~~

We need to set this `project.version` as thats being read by the release plugin. I couldnt make it work with any random project property. So I had to explicitily write it out.

~~~groovy
def getCustomVersion(boolean base) {
    if(ext.branch == "master") {
        return base ?  masterVersion
                    : masterVersion + '.' + 'RELEASE'
    } else {
        return base ?  snapshotVersion
                    : snapshotVersion + '.' + 'RELEASE'
    }
}
~~~

The above will be called by 

~~~groovy
task updateCustomVersions {
    doLast {
        def ver = getCustomVersion(true)
        def newVer = updateCustomVersionsHelper(ver)
        if(gitBranch() == "master") {
            println("WRITING RELEASE VERSION TO " + newVer)
            writeVersion("masterVersion", newVer)
        } else {
            println("WRITING SNAPSHOT VERSION TO " + newVer)
            writeVersion("snapshotVersion", newVer)
        }
    }
}
~~~

And here are the additional methods needed ...

~~~groovy
// increment version by 1
String updateCustomVersionsHelper(String ver) {
    Map<String, Closure> patterns = [
            /(\d+)([^\d]*$)/: { Matcher m, Project p -> m.replaceAll("${(m[0][1] as int) + 1}${m[0][2]}") }
    ]

    for (entry in patterns) {
        String pattern = entry.key
        Closure handler = entry.value
        Matcher matcher = ver =~ pattern

        if (matcher.find()) {
            return handler(matcher, project)
        }
    }
}

// writes version to gradle.properties file
protected void writeVersion(String key, version) {
    try {
        def file = project.file("gradle.properties")
        if (!file.file) {
            project.ant.echo(file: file, message: "$key=$version")
        } else {
            def fileText = file.text
            fileText = fileText.replaceAll(key + ".*", String.format("%s=%s", key, version))
            file.write(fileText)
        }
    } catch (Exception be) {
        be.printStackTrace()
        throw new GradleException('Unable to write version property.', be)
    }
}
~~~

Finally, call the `updateVersion` task when release is executed :

~~~groovy
commitNewVersion.dependsOn updateCustomVersions
~~~

Use automatic versioning and see it running :

~~~
./gradlew release -Prelease.useAutomaticVersion=true
~~~
