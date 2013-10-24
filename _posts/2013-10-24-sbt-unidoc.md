---
layout: post
title: "consolidating scattered scala api docs"
description: ""
category: ""
tags: [scala]
---
{% include JB/setup %}

__TL;DR: This post shows you how to build Scala API docs across multiple libraries, so that cross links work fine.__

There is days when you hate sbt, the sometimes not so simple build tool for Scala, and others where you are delighted to find out that others have invested energy to solve your problems. This is about the latter case. 

## preliminaries

One of my open source projects is [ScalaCollider](https://github.com/Sciss/ScalaCollider), a sound synthesis library talking to the SuperCollider server. For those who have no clue, SuperCollider server is one of the best sound synthesis systems out there. It is designed to be talked to from a client, the default being the SuperCollider language (or SC-Lang), a powerful computer music language. ScalaCollider replaces SC-Lang and brings computer music programming to my favourite general purpose language, Scala. I am not going into the details of ScalaCollider or why it exists, let me just say that (*hints at Paul Philips*) the main motivation for me was to be able to enjoy the benefits of a _general purpose language_, which in my experience is something that _exists_.

I am a person that keeps API documents open in the browser most of the time, despite enjoying the support of such terrific IDEs as IntelliJ IDEA. Scala follows Java in providing a standard documentation tool, scaladoc, which parses a body of source code, generates trees for the classes and methods, and finds documentation fragments from specially formatted comment blocks in the source code. To manage my Scala projects, I use [sbt](http://www.scala-sbt.org/), which conveniently generates the API docs with the `doc` task.

Now this project depends on a few libraries I wrote, for example [ScalaOSC](https://github.com/Sciss/ScalaOSC) for Open Sound Control support, the protocol used to talk to SuperCollider, or [ScalaAudioFile](https://github.com/Sciss/ScalaAudioFile) for accessing audio files on the client side. Furthermore, ScalaCollider itself got split, "outsourcing" a separate project [ScalaCollider-UGens](https://github.com/Sciss/ScalaColliderUGens) whose purpose it is to automatically generate a long list of class files for the basic DSP building blocks in SuperCollider, the Unit Generators (UGens). It does so by parsing an XML description and synthesising `.scala` source files, which are then a dependency for ScalaCollider.

## problem

The problem with this modularity is that the API docs get scattered across different related sub projects. They are even in different git repositories. Generating the docs for each project on its own is fine, but scaladoc doesn't know there are other documents out there, so there aren't any cross links. If you open the main ScalaCollider docs, you may find information about the `Server` class, the `Buffer` and `Bus` resources, but the UGens are missing&mdash;they are really the main thing you will look up most of the time. And vice versa, you open the UGen docs, and cross links to `Synth` or `Buffer` or the type class `GEOps`, which provides all sorts of combinators for UGens, are broken.

## solution

One of the most active sbt plugin developers is Eugene Yokota, I recommend checking out his projects, there is plenty of useful things to discover. So I just found out about the [sbt-unidoc](https://github.com/sbt/sbt-unidoc) plugin which promises to solve the problem of scattered API docs. The readme shows how this works with a multi project sbt setup. Fortunately, it is possible to adapt this to the situation where the sources are spread across multiple repositories. Sbt has a cool feature called (external) project references, Alvin Alexander has a [brief blog post about this](http://alvinalexander.com/scala/using-github-projects-scala-library-dependencies-sbt-sbteclipse).

You define a new multi project build file that declares all the libraries whose API docs you want to combine as external projects, then you run `sbt unidoc` on the root aggregate. With a few extra steps these docs can then be published to the main project's [GitHub pages](http://pages.github.com/).

The basic layout I have:

    project/build.properties
    project/plugins.sbt
    build.sbt

The file `build.properties` just contains the sbt version used:

{% highlight properties %}
sbt.version=0.13.0
{% endhighlight %}
    
The plugins file contains the unidoc plugin. In order to publish to GitHub pages, also the [sbt-ghpages](https://github.com/sbt/sbt-ghpages) plugin by Josh Suereth is added:

{% highlight scala %}
resolvers += "jgit-repo" at "http://download.eclipse.org/jgit/maven"

addSbtPlugin("com.typesafe.sbt" % "sbt-ghpages" % "0.5.2")

addSbtPlugin("com.eed3si9n" % "sbt-unidoc" % "0.3.0")
{% endhighlight %}

And here is how my final `build.sbt` file looks:

{% highlight scala linenos %}
scalaVersion in ThisBuild := "2.10.3"

val lOSC       = RootProject(uri("git://github.com/Sciss/ScalaOSC.git#v1.1.2"))

val lAudioFile = RootProject(uri("git://github.com/Sciss/ScalaAudioFile.git#v1.4.1"))

val lUGens     = RootProject(uri("git://github.com/Sciss/ScalaColliderUGens.git#v1.7.2"))

val lMain      = RootProject(uri("git://github.com/Sciss/ScalaCollider.git#v1.10.0"))

git.gitCurrentBranch in ThisBuild := "master"

val root = (project in file("."))
  .settings(unidocSettings: _*)
  .settings(site.settings ++ ghpages.settings: _*)
  .settings(
    site.addMappingsToSiteDir(mappings in (ScalaUnidoc, packageDoc), "latest/api"),
    git.remoteRepo := s"git@github.com:Sciss/ScalaCollider.git",
    scalacOptions in (Compile, doc) ++= Seq("-skip-packages", "de.sciss.osc.impl")
  )
  .aggregate(lOSC, lAudioFile, lUGens, lMain)
{% endhighlight %}

Some explanation is needed. Lines 3, 5, 7, 9 contain the references to the libraries for which the docs should be generated. They are specified as URIs to their respective GitHub repositories. What happens when sbt spins is it will clone these repos somewhere into `~/.sbt/0.13/staging/` and _build them from source_. If you somehow change a thing in 'origin', this is the directory you should wipe to make sbt re-download the repository.

I guess it is also advised to specify _which version_ of each library should be used. This can be done by appending a hash character `#` and a git tag corresponding with the version. If you haven't used git tags, imagine you are about to push a new version 1.2.3 of your library. After committing the last changes, you tag that last commit like follows:

    $ git tag -a 'v1.2.3' -m 'version 1.2.3'
    
(The `-m` bit is a comment for the tag.) Then don't forget to push the new tag to GitHub:

    $ git push --tags
    
Ok. Now you can reference that tag as shown in the `build.sbt`.

Second, you might need to add line 11 which seems to be a bug of `sbt-git` when you are not in the root directory of a git repository. The next important thing is to define the root project as an aggregate over all the libraries. If you only want to build your local docs, the settings in line 14 are suffcient, whereas lines 15, 17, 18 are needed for the GitHub pages. Line 17 tells the [sbt-site plugin](https://github.com/sbt/sbt-site) (which is automatically downloaded with sbt-ghpages) to map the output of unidoc to directory `latest/api` on the GitHub page. The `remoteRepo` specifies the repository to which the docs should be uploaded. It assumes that you have performed the initial steps of setting up a `ghpages` branch, as described in the sbt-ghpages [readme](https://github.com/sbt/sbt-ghpages).

Finally, line 19 shows you how to exclude a particular package from the docs. I have used this here to eliminate some implementation specific package.

## ready

After this setup, generating the docs is as straight forward as running `sbt unidoc`. If all goes well, this will clone the libraries and run a large scaladoc task including all the sources together. The result is found in `target/scala-2.10/unidoc/index.html`. Previewing the GitHub pages variant is as easy as running `sbt preview-site`. You want to open the URL `http://localhost:4000/latest/api` then. If you are happy with the result, `sbt ghpages-push-site` will push the docs to the `ghpages` branch [of your repository](http://sciss.github.io/ScalaCollider/latest/api/).

## caveats

I found two caveats. First, it may turn out that the compilation fails although all your libraries compile fine in isolation. You may have introduced some new symbol shadowing. In my case, I used `io.Source` in the UGens project, but when throwing in the ScalaAudioFile library, I suddenly had a package `de.sciss.io` which was visible since UGens also use the `de.sciss` prefix. Fortunately, this was easy to fix by explicitly importing `scala.io.Source` in the UGens project (which I had to push again as a result with a new version tag).

The second problem I encountered is that because sbt is now building all the libraries from source, you may be in trouble when they where configured for an older sbt version. In my case, one of the libraries used sbt 0.12.3 and not 0.13.0. Now sbt tries to build the project with 0.13.0 and chokes when it encounters a plugin (from the build file of that library) which cannot be resolved. I used a version of `sbt-buildinfo` which was not published for sbt 0.13.0. Luckily that is also easy to fix, I just bumped the sbt and plugins version in my library project along with its minor version and pushed it with a new version tag.
