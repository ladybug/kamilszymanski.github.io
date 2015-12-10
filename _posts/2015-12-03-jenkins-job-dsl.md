---
layout: post
title: Jenkins job DSL
date: 2015-12-03T13:29:50+01:00
excerpt: "Configuring Jenkins jobs via code using Groovy-based DSL"
tags: [jenkins, DSL, job-dsl]
comments: true
share: true
---

No that long ago one of our engineers made a mistake while running Jenkins CLI script and removed all jobs apart from the ones he wanted to remove.
Looking at how many jobs we have at the moment it might have been over 300 jobs combined into dozens of pipelines.
Pretty scary you might think when you consider that we were not running scheduled backups.
Well, in my opinion it was the best test for the continuous delivery platform we were working on for a couple of weeks.
In less than a 15 minutes we had all the jobs back and it took so long because we didn't want to skip tests during the restore nor bring back unneeded jobs[^1].
How is that possible?
Let me introduce Jenkins [Job DSL / Plugin](https://github.com/jenkinsci/job-dsl-plugin), a project made up of two parts: the Domain Specific Language that allows users to describe jobs using Groovy-based language, and a Jenkins plugin which manages the scripts and the updating of Jenkins jobs which are created and maintained as a result.

But before we jump into job-dsl lets take a step back and take a brief look at how Jenkins jobs are configured.
If you create a new job and go to `http://<JENKINS_HOST>/job/<JOB_NAME>/config.xml` you will get the following XML:

{% highlight xml %}
<project>
  <actions/>
  <description/>
  <keepDependencies>false</keepDependencies>
  <properties/>
  <scm class="hudson.scm.NullSCM"/>
  <canRoam>true</canRoam>
  <disabled>false</disabled>
  <blockBuildWhenDownstreamBuilding>false</blockBuildWhenDownstreamBuilding>
  <blockBuildWhenUpstreamBuilding>false</blockBuildWhenUpstreamBuilding>
  <triggers/>
  <concurrentBuild>false</concurrentBuild>
  <builders/>
  <publishers/>
  <buildWrappers/>
</project>
{% endhighlight %}

Not that interesting so let's add `Execute shell` build step that prints `Hello world!`.
Immediately you will see changes to `builders` node in `config.xml`:

{% highlight xml %}
<builders>
  <hudson.tasks.Shell>
    <command>echo 'Hello world!'</command>
  </hudson.tasks.Shell>
</builders>
{% endhighlight %}

*If you want to view changes to XML on each change made install [JobConfigHistory plugin](https://wiki.jenkins-ci.org/display/JENKINS/JobConfigHistory+Plugin).*

Still nothing spectacular, so let's parameterize the build and look at the changes to `config.xml` where `properties` node should now contain parameter definition:

{% highlight xml %}
<properties>
  <hudson.model.ParametersDefinitionProperty>
    <parameterDefinitions>
      <hudson.model.StringParameterDefinition>
        <name>MESSAGE</name>
        <description/>
        <defaultValue>Hello world!</defaultValue>
      </hudson.model.StringParameterDefinition>
    </parameterDefinitions>
  </hudson.model.ParametersDefinitionProperty>
</properties>
{% endhighlight %}

and `Execute shell` build step should reflect that we're using this parameter in `echo` command:

{% highlight xml %}
<builders>
  <hudson.tasks.Shell>
    <command>echo $MESSAGE</command>
  </hudson.tasks.Shell>
</builders>
{% endhighlight %}

Even though I haven't told what steps I had taken you shouldn't have trouble replaying them - those XMLs are descriptive enough (as for XML).

Ok, it's getting boring so let's do something more sophisticated and install [Rebuild plugin](https://wiki.jenkins-ci.org/display/JENKINS/Rebuild+Plugin) to help us re-run parameterized jobs.
Installing this plugin should yield no changes to existing jobs (in the end they run on plugin defaults) but when you create a new job you should find Rebuild plugin configuration under `properties` node:

{% highlight xml %}
<properties>
  <com.sonyericsson.rebuild.RebuildSettings plugin="rebuild@1.25">
    <autoRebuild>false</autoRebuild>
    <rebuildDisabled>false</rebuildDisabled>
  </com.sonyericsson.rebuild.RebuildSettings>
</properties>
{% endhighlight %}

Now after we enable `Rebuild Without Asking For Parameters` for our existing job it can be re-run with previous parameters without having to go through a view listing all the parameters as reflected in `com.sonyericsson.rebuild.RebuildSettings` node:

{% highlight xml %}
<com.sonyericsson.rebuild.RebuildSettings plugin="rebuild@1.25">
  <autoRebuild>true</autoRebuild>
  <rebuildDisabled>false</rebuildDisabled>
</com.sonyericsson.rebuild.RebuildSettings>
{% endhighlight %}

Still boring as XML (pun intended), fortunately Groovy makes working with XML much more pleasant and has fantastic capabilities for writing DSLs.
This, along with Jenkins built-in Groovy support (it features a Groovy script console that I already mentioned in this post), makes a perfect match for [job-dsl plugin]((https://github.com/jenkinsci/job-dsl-plugin)) that we're going to explore now.

Let's start with the DSL and try to get the equivalent of `config.xml` with Groovy-based DSL.
Armed with [API docs](https://jenkinsci.github.io/job-dsl-plugin/) (*should you feel lost start from `buildFlowJob` node*) take a moment and experiment on [Job DSL Playground](http://job-dsl.herokuapp.com/).
You should end up with a script similar to this one:

{% highlight groovy %}
job('parameterized-hello-world') {
   parameters {
     stringParam('MESSAGE', 'Hello world!') 
   }
   properties {
     rebuild {
       autoRebuild()
     }
   }
  steps {
    shell('echo $MESSAGE')
  }
}
{% endhighlight %}

Way more friendly than XML and it's code that you can put under version control.
Moreover you don't have to click through GUI to get this configuration.
But how do you run it in Jenkins?
Well, it's as simple as creating (let's do it manually for now) and running a seed job which is a regular job with `Process Job DSLs` build step containing your script (or even better pointing to the file containing the script).
In no time you should see a successful build informing you that it has generated a new job
![alt text](../images/posts/jenkins-job-dsl/seed_generated_items.png "seed generated items")
and the generated job will hold information that it has been generated by the seed job
![alt text](../images/posts/jenkins-job-dsl/job_reference_to_seed.png "job reference do seed")

And if you ever want to change anything about the generated job you just apply changes to the DSL script (that you keep under version control) and re-run the seed job (that points to the script(s) under version control).
Like an old record I will once again mention version control, please keep your scripts under version control!

Now you can revoke all permissions for manual job configuration and make all changes via code!

Should job-dsl lack support for your favourite plugin just submit a pull request.
It should be easy after getting the basic concepts and if you need something to start with you can take a look how we've [added support for Rebuild plugin](https://github.com/jenkinsci/job-dsl-plugin/pull/606/files).

After you get familiar with the DSL you can build your own around it to further simplify jobs' (or even whole pipelines) configuration.
You can also consider describing pipelines as metadata kept inside projects' repositories that are scanned by a periodically run seed job that parses that metadata and (re-)generates jobs on any change.
Since it's code you craft anything your CI/CD solution needs.

[^1]: we had a few seed jobs each pointing to multiple (using glob patterns) scripts but to speed up platform development we didn't version control seed jobs
