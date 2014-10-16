---
layout: default
title: "Keeping up with Sprout-Wrap"
author: James Hart
author_username: james
---

Keeping your sprout-wrap recipes and cookbooks up to date is a good thing to do! It keeps your software secure (because bugs like Heartbleed happen), and it allows your developers to enjoy improving their workstation workflow and processes with the most recent software and tools.

Sprout-wrap has been a moving target lately, and has undergone some recent growing pains. Many cookbooks have moved, and as a result your old recipes and cookbooks will grow stale unless you change your `Cheffile` and `soloistrc` file to point to the newest cookbooks and recipes. The good news is, this should be a one time "upgrade." After pointing towards the newest cookbooks a simple `librarian-chef update` will suffice for keeping up to date. This blog post will hold your hand while you perform this one-time upgrade, as I found it to be a bit tricky!

## Upgrade sprout-osx-base to sprout-base first

If you're using the `sprout-osx-base` cookbook, it has been renamed, and this proves to be tricky when there are dependencies from external cookbooks.

Add `sprout-base` to your `Cheffile`, and keep the old `sprout-osx-base` cookbook there for now.

```ruby
cookbook 'sprout-base',
  :git => 'git://github.com/pivotal-sprout/sprout-base.git'

cookbook 'sprout-osx-base',
  :git => 'git://github.com/pivotal-sprout/sprout-base.git'
```

Adding `sprout-base` and `sprout-osx-base` to your `Cheffile` will allow for your cookbooks to be backwards compatible. If old cookbooks reference `sprout-osx-base`, their dependencies will resolve properly. Similarly, when new cookbooks reference `sprout-base` they will resolve to those same recipes. We'll remove the `sprout-osx-base` cookbook at the end.

Try running a `librarian-chef install`. If the cookbooks have been extracted to separate git repositories already you'll see an error message like this:


```ruby
No metadata file found for rubymine from git://github.com/pivotal-sprout/sprout.git#master(sprout-osx-rubymine)! If this should be a cookbook, you might consider contributing a metadata file upstream or forking the cookbook to add your own metadata file.
```

This means that the cookbook has been moved. For sprout-osx-rubymine you can replace this code piece in your `Cheffile`

```ruby
cookbook 'rubymine',
  :git => 'git://github.com/pivotal-sprout/sprout.git',
  :path => 'sprout-osx-rubymine'
  ```
with

```
cookbook 'sprout-rubymine',
  :git => 'git://github.com/pivotal-sprout/sprout-rubymine.git
```
Okay, now that we've managed to do a `libarian-chef install`, try running a `bundle exec soloist`. During cookbook compilation, you'll notice that some cookbooks aren't found. That's because some recipes have moved to their own dedicated cookbook. In `soloistrc` where you once had both `sprout-osx-apps::rubymine` and `sprout-osx-apps::rubymine_preferences` we can now simply say `sprout-rubymine::default`.

## Dedicated Cookbooks

You should take a moment to read through the sprout-rubymine::default recipe, as it might have changed slightly.  Dedicated cookbooks allow our recipes to remain simple, the cookbooks dependencies remain more explicit, and the seperation of concerns more defined. It makes testing these cookbooks much easier (you are testing your cookbooks, right?).  In rubymine's case, we can focus on installation and managing it's preferences. Simple!

Here is a list of the dedicated cookbooks for sprout-wrap that you'll want to be using to be up to date with sprout-wrap development.

* Git
* Homebrew
* Terminal
* Vim
* Mysql / Postgres
* Rbenv
* Rubymine
* OSX Settings

Now that we've moved all of our recipes to their dedicated cookbooks, hopefully you can do a successful run of `bundle exec soloist`. We can start upgrading our cookbooks now. Let's try upgrading one cookbook at a time:

`bundle exec librarian-chef update sprout-pivotal`

Try running `soloist` again on your machine to ensure that all of the recipes you've been using are still working on your current machine. You can look farther down in the troubleshooting section to see how to resolve numerous issues you may commonly hit.

Now that you've gotten past upgrading one cookbook, you can go through and upgrade each one sequentially. If you're feeling ambitious, you can do so with a simple `librarian-chef update`. At this point, I'd like to note that I have found that `librarian-chef update --verbose` to be extremely helpful in understanding _why_ the error messages are happening, and troubleshoot accordingly.

## Remove pivotal-workstation cookbook

The pivotal-workstation cookbook was the first generation of sprout-wrap. It's recipes are continually being deprecated, and support for them is dwindling (if not already dead.) The recipes that are in your `soloistrc` file have been ported over to other individual cookbooks.

You can see how I removed the pivotal-workstation cookbook [here](https://github.com/hjhart/sprout-wrap/compare/ea089d6ff7f28c4dcea173d5116c355ea7e34db8...fea5351ba594a81b7911ccd80660d9c29dd32df8).

After you remove the pivotal-workstation cookbook from your `Cheffile`, you'll notice that all of your dependnecies on the `sprout-osx-base` cookbook have gone away. Go ahead and remove the following lines from your Cheffile.

```ruby
cookbook 'sprout-osx-base',
  :git => 'git://github.com/pivotal-sprout/sprout-base.git'
```

## Sprout-wrapping it up

Alright! Nice work. Looks like you're all up to date with sprout-wrap now. Going forward it **will** be much easier to upgrade, thankfully. A simple `librarian-chef update` will definitely be enough to keep your machine's Chef recipes up to date.

