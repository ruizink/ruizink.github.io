---
layout: post
title: "Chef, Test Kitchen, and Berkshelf. What could go wrong?"
author: Mário Rui Santos
modified:
excerpt: "An awesome set of tools to assist cookbook development."
date: 2015-10-01
comments: true
---
An awesome set of tools to assist cookbook development.

<br>
When I found out about this awesome combination of tools, I immediately thought that this was the perfect set for developers to test their cookbooks. (I still do!)

This set of tools, when combined, will vastly improve your cookbook development workflow. It will allow you to fully test your cookbooks locally, without messing around with actual boxes from your estate.

<div style="text-align:center" markdown="1">
![chef_kitchenci_berkshelf](/images/chef_kitchenci_berkshelf.png)
</div>

<br>

> Discovering

To start with, I had my cookbooks stored on a [Chef Server](https://www.chef.io/chef/), had a [Berkshelf API](http://berkshelf.com/) to connect to and also [Test Kitchen](http://kitchen.ci/) installed.

I wrote my first .kitchen.yml, edited my Berksfile and _voilá_...

* **Pro tip:** Install [ChefDK](https://downloads.chef.io/chef-dk/) instead of each separate tool. It packs really nicely a lot of useful tools.

A reasonable choice for the Test Kitchen provisioner was chef_solo, so it was my first option. I ran `kitchen converge` and I felt proud because I was able to test my cookbooks running against several Test Kitchen suites, with a single freakin' command.

<br>

> Quick win

Just brilliant, until I realized that I store a set of cookbook version constraints in some of my Chef environments.

Chef running in solo mode doesn't support that and smashed me in the face with a:

{% highlight bash %}
Unexpected Error:
-----------------
Chef::Exceptions::IllegalVersionConstraint: Environment cookbook version constraints not allowed in chef-solo
{% endhighlight %}

Hold on a minute, that's an easy fix: just STAHP using the chef_solo provisioner and start running a Chef Server inside your box instead, using the chef_zero provisioner.

Seems that I'm right on track again. 

With a single command, everything happens like magic: Test Kitchen, with the chef_zero provisioner, runs Berkshelf that resolves and downloads all the cookbook dependencies, spins up a box and runs Chef Zero with those cookbooks.

<br>

> Problems ahead

Using chef_zero didn't totally solve my problem...

If you're familiar with Berkshelf, you know that is awesome for resolving cookbook dependencies, but it doesn't care about Chef environments at the time of dependency resolution.

How does Berkshelf actually works?

Berkshelf API looks into the metadata.rb for each cookbook version that lives in a given cookbook repo. From there, it exposes a JSON (often called _universe_) via HTTP which contains all the dependencies for each cookbook version. The _universe_ also contains info about how/where a cookbook can be downloaded.

The Berkshelf client is responsible for generating a dependency graph in runtime, based on the _universe_ fetched from the Berkshelf API.

What happens if you don't specify a dependency version in your cookbook's metadata file?

Berkshelf will assume, logically, that your cookbook should be ok with any version of the dependency and so, it vendors the latest version available from the repository.

<br>
Let's take a look at the example below:

{% highlight bash %}
# wrapper_cookbook/metadata.rb

name "wrapper_cookbook"
...
depends "application_cookbook"
{% endhighlight %}

{% highlight bash %}
# application_cookbook/metadata.rb

name "application_cookbook"
...
{% endhighlight %}

{% highlight bash %}
# dev_environment.json

...
"cookbook_versions": {
  "wrapper_cookbook": "= 1.2.1",
  "application_cookbook": "= 1.0.0"
}
...
{% endhighlight %}

{% highlight bash %}
# cookbook list on your chef repo

wrapper_cookbook       0.0.1   1.0.0   1.2.1
application_cookbook   0.0.1   1.0.0   2.0.0
{% endhighlight %}

<br>
Given those metadata files, Berkshelf will vendor the latest cookbooks available: `wrapper_cookbook-1.2.1` and `application_cookbook-2.0.0`.

It's easy to predict that by the time Chef Zero loads the environment, it tries to converge using the `wrapper_cookbook-1.2.1` and `application_cookbook-1.0.0` (the versions specified in the Chef environment) and it fails, because Berkshelf didn't vendor the version 2.0.0 of the application_cookbook:

{% highlight bash %}
Missing Cookbooks:
------------------
Could not satisfy version constraints for: application_cookbook
{% endhighlight %}

<br>

> Denial phase

We tend to believe that given so many people are using the same tools that we do, then someone must have been facing the same problems that you do.

That may not be the case. Even when facing the same problem, people eventually came up with a solution that doesn't suit your needs. Maybe they just dealt with the problem or changed the way they use the tool.

Quick search is acceptable: There's always a good possibility of quickly finding the solution for your problem. Perhaps it's a very common one.

Deep search: You need to decide if it's worth it. Find the balance between the size/complexity of your problem and the time you'll need to fix it yourself. Make a choice and stick with it.

Desperate search: Ok, you're in denial. You probably chose the wrong tool for the job. Move on.

<br>

> Reality

I believe that many people that use Berkshelf, take advantage of the generated Berksfile.lock to lock those versions on the Chef environment using `berks apply`, and not the other way around. But if your Chef environment is not being managed with Berkshelf, you probably want a way to feed Berkshelf with the version constraints specified in your Chef environment.

One may argue that the second option isn't the best workflow ever, and I'll probably agree with that. I find that the [Environment Cookbook Pattern](http://blog.vialstudios.com/the-environment-cookbook-pattern/) makes much more sense. But that's a big question for debate, out of the scope of this story.

In the end, sometimes it's harder to change the way of how teams work and their workflows than it is to customize a tool to solve your problems.

<br>

> Get sh*t done

Waiting for someone to come up with a solution for your problems? Yeah, right... Take charge and fix it, 'cause probably no one else will.

<br>
So I took action and started digging through Test Kitchen's source code. I really enjoyed the way you can just plug in new provisioners, brilliant.

I ended up extending a few classes of the chef provisioners combined with the Berkshelf for dependency solving.

I've built a custom provisioner called chef_zero_berks_env. (Yeah, you can laugh... I also laughed in the next morning, when I realized what I've done!)

The provisioner simply loads a chef environment, fetches the cookbook version constraints and adds dependencies to Berkshelf in runtime before it saves the Berksfile.lock.

After that, it vendors the cookbooks using Berkshelf, spins up a box and provisions the box with Chef Zero.

This way, the vendored cookbook versions are always gonna match what Chef Zero expects to converge.

Pretty simple, right?

I've also packaged this provisioner in a gem and I've pushed it to [RubyGems](https://rubygems.org/gems/kitchen-chef_zero_berks_env/). This enables any person to just `gem install kitchen-chef_zero_berks_env` and start using this provisioner out of the box.

The source code is hosted on [GitHub](https://github.com/ruizink/kitchen-chef_zero_berks_env), so feel free to PR. I'm expecting you to poke me if you see something wrong!

<br>

> End of story

Tools are built by people that face specific problems. If you're lucky to share the same problems, cool, it should be plug-n-play. If not, well... that's the beauty of open-source projects and the community around them.

Don't waste too many time looking for someone facing the same problem as you do. Sometimes it's more productive to just dig into the code and try to make it so that the tool solves your problem. And then share that back with the community, 'cause maybe someone will eventually end up needing it and finding it on google.

It's all about this awesome knowledge sharing loop that anyone should feel proud to make part of. You should too!

<br>
**Later chaps!**
