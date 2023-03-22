---
layout: post
title: Multi-module Gradle to Debian
subtitle: Debian out of a multi-module gradle application 
---

There is already a [gradle plugin](https://github.com/gesellix/gradle-debian-plugin) which creates a debian out of a gradle project. 

However, as far as I remember, that did had certain problems with the newer version of Gradle (2.2+) and in multi-module projects. (Unfortunately, it is too late to remember what exactly was the problem).

Anyhow, I wrote some custom gradle tasks, that just creates a debian package. Of course, the debian control files, and file heirarchies are project specific and needs to be changed as per requirements.

As mentioned, I added a couple of tasks, starting with the one `allJar` which creates a fat jar with all the complied class files :

~~~
task allJar(type: Jar, dependsOn: [subprojects.tasks['classes'], subprojects.sourcesJar, subprojects.jar, subprojects.copyToLib ] ) {
    baseName = project.name + '-all'
    subprojects.each { subproject ->
        from subproject.configurations.archives.allArtifacts.files.collect {
            zipTree(it)
        }
    }
    manifest {
        attributes 'Main-Class': mainClassName
    }
}
~~~

The task `copyToLib` is used here to include any external jar files located in `lib` directories.

~~~
subprojects {
    task copyToLib( type: Copy) {
        into "$buildDir/libs/lib"
        from configurations.runtime
    }
}
~~~

After this, we can just add all the required tasks in one single file, for e.g., `debian.gradle`.

~~~
task copyJarDeps(type: Copy) {
    from(subprojects.configurations.runtime)
    into project.file('debian/package/usr/share/test-project')
}

task copyControlFiles(type: Copy) {
    from './DEBIAN_CONTROL'
    into project.file('debian/package/DEBIAN')
}

task copyAppConfigFiles(type: Copy) {
    from 'rootProject/config'
    into project.file('debian/package/etc/configName')
}

task copySwaggerFiles(type: Copy) {
    from 'swagger'
    into project.file('debian/package/usr/share/assets/swagger')
}

task copyDebRunScripts(type: Copy){
    from './startup_script'
    into project.file('debian/package/etc/init.d')
}

task copyMainClassJar(type: Copy, dependsOn: allJar){
    from 'build/libs/root-project-all-0.1.0.jar'
    into project.file('debian/package/usr/share/projectName')
}

task copyDebFiles(dependsOn: [copyJarDeps, copyAppConfigFiles, copyDebRunScripts, copyControlFiles, copySwaggerFiles])


task buildDeb(dependsOn: [allJar, copyDebFiles, copyMainClassJar]) {
}
~~~

Of course, the string constants are to be replaced as per requirements of the project. And so is the debian control files in the directory `DEBIAN_CONTROL`.

Now, we just include this file in the main `build.gradle` file :

~~~
apply from: file("gradle/debian.gradle")
~~~

Now, we can build a debian package by using the command :

~~~
./gradlew clean buildDeb
~~~

