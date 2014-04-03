---
layout: default
title: Capistrano 3, You've Changed! (Since Version 2)
---
**Capistrano** has been around for almost as long as Rails has been around, perhaps short by just a year or so. Back in the early days it introduced much needed sanity into the world of deployment automation, including documenting in code some of the best practices for application deployment, such as the directory layout that included 'releases' folder with the ability to roll back, 'shared' folder with the ability to maintain continuity from release to release. Capistrano was built upon the concept of having roles for application servers. Finally, being written in Ruby, Capistrano always offered remarkable levels of flexibility and customization. So it should not come as a surprise that it became highly popular, and that subsequent infrastructure automation tools like Chef and Puppet include Capistrano-like deployment automation recipes.

These days it is not uncommon to bump into Python, Java, or Scala applications that are deployed to production using Capistrano (which itself is written in ruby).  It's because a lot of the assumptions that Capistrano makes are not language or framework specific.

It's worth noting that in it's entire history of existence, Capistrano have not had an upgrade so dramatically different from the previous version, that in some way it requires rewiring some of your brain neurons to grasp new concepts, new callbacks, and the new mappings between roles and servers, for example.

This blog post represents a typical tale of "We upgraded from version X to version Y. It was hard! But here's what we learned.". And amazingly, despite having been released more than 4 months ago, there is still a massive shortage of quality Capistrano 3 documentation (or upgrade paths) online.  With this post I am hoping to bridge this gap a tiny bit, and perhaps help a few folks out there upgrading their deployment scripts.


The lessons below are ordered in the order of importance, from top (highest) to the bottom (lowest). And now, with the introduction out of the way, let's dive straight into it!

## Assume your Capistrano 2 Plugins Won't Work

Gems that give Capistrano extended features were pretty common in version 2. Version 3 took the most important features and merged it into either the `capistrano` or `capistrano-rails` codebase. If you're working with a Ruby on Rails project like we are, you'll find most of the functionality you need within those two gems. This includes asset compilation, bundling, deploying via your SCM of choice, symlinking of folders as well as standard configuration. There is fantastic documentation about "capistrano 3 flow" [here](http://capistranorb.com/documentation/getting-started/flow/).

## Flow Has Changed

The callbacks that you knew and loved in capistrano 2 are gone. When upgrading to version 3 and you want to write custom deployment tasks, you'll need to refer to the Capistrano 3 Flow documentation and find the appropriate callback to hook into. Thankfully, they're a lot easier to understand now. Here is the flow as of Capistrano 3.1 (excluding rails callbacks).

```ruby
deploy:starting    - start a deployment, make sure everything is ready
deploy:started     - started hook (for custom tasks)
deploy:updating    - update server(s) with a new release
deploy:updated     - updated hook
deploy:publishing  - publish the new release
deploy:published   - published hook
deploy:finishing   - finish the deployment, clean up everything
deploy:finished    - finished hook
```
from [http://capistranorb.com/documentation/getting-started/flow/](http://capistranorb.com/documentation/getting-started/flow/)

Capistrano 2 had less clear callbacks like `deploy:setup`, `deploy:default`, and `deploy:update_code`. Make sure if you hooked into any of those, you change them to a corresponding Cap 3 callback.

## Command Line Argument Overrides Have Changed

You can no longer override variables in capistrano using -S. If you try to deploy a particular branch in Cap3 with

```bash
cap deploy -S branch=foo
```

you will now receive `invalid option: -S`. The standard practice now is to pass them in as environment variables. For example, in our deploy.rb file we have this:

```ruby
set :branch, ENV["REVISION"] || ENV["BRANCH_NAME"] || "master"
```

So we can deploy a specific branch or revision now by specifying one of the valid environment variables, e.g.

```bash
REVISION=8b42a3e be cap production deploy
BRANCH_NAME=unsafe_production_branch be cap staging deploy
```

## REVISION no longer exists.

Capistrano used to create a `#{app_path}/current/REVISION` file that contained the current 41 character git revision string. If your capistrano tasks or application code depends on this file, make sure you change it to accommodate the new default capistrano process. The data still exists, but it is now stored at `#{app_path}/revisions.log`. It looks like this:

```bash
$ cd /home/wanelo/app
$ tail revisions.log
Branch master (at 8b42a3e) deployed as release 20140305230738 by wanelo;
Branch master (at 65918e9) deployed as release 20140306003234 by wanelo;
Branch master (at 450a273) deployed as release 20140306004830 by wanelo;
Branch master (at db36080) deployed as release 20140307003934 by wanelo;
Branch master (at e324ded) deployed as release 20140307012920 by wanelo;
Branch master (at 7d8eba2) deployed as release 20140307191841 by wanelo;
Branch master (at 4c12f57) deployed as release 20140308001458 by wanelo;
```

So, with a little bit of bash magic we were able to still parse the (now shorter) git revision string to use in our application code.

```bash
$ cd /home/wanelo/app
$ grep deployed revisions.log | tail -n 1 | cut -f1 -d')' | cut -f4 -d' '
=> b3d13f3
```

## Rollbacks, You've Changed

Capistrano 3 no longer rolls back automatically if something fails. Upon the first non-zero exit status from a remote machine the deployment stops executing. In capistrano 2, if a command that executes returns a non-zero exit status it would, by default, delete the currently deploying release directory.  Now, If the symlinking from the `release` directory to the `current` directory has already happened, your new code is successfully on the server. If you restart your application server at this point, you'll happily get your new code.

## Continue On Failures No Longer Supported

There used to be a nice feature in capistrano 2 that would allow failed tasks to not abort the flow of the deployment. For instance:

```ruby
task :notify_campfire_of_successful_deploy, :on_error => :continue do
    # your code could explode here in capistrano 2
end
```

This no longer exists in Capistrano 3! The best solution I could find to this problem was a `begin; rescue; end` to ensure that finicky code like HTTP requests to third-parties don't cause your deploy to halt execution.

# Getting to Know Capistrano 3

## Server Definitions Over Role Definitions

In capistrano 3 you now have two different methods of defining servers and server roles. The role based definition is more terse and better for smaller deployment setups. The server based definition is more versatile and allows you to better understand the responsibilities of each server. You can also assign custom attributes to individual servers and use those in your own capistrano tasks. Take a look at the difference between role based definition and server based definitions.

```ruby
role :sidekiq,                %w{ sidekiq1 sidekiq2 }
role :web,                    %w{ load_balancer1 load_balancer2 }
role :app,                    %w{ application_server1 application_server2 application_server3 application_server4 }
role :db,                     %w{ application_server1 }
role :whenever_scheduler,     %w{ application_server2 }
```

```ruby
server "sidekiq1",                    roles: [:sidekiq],                    cluster: 'c1'
server "sidekiq2",                    roles: [:sidekiq],                    cluster: 'c2'

server "application_server1",         roles: [:app, :db],                   cluster: 'c1'
server "application_server2",         roles: [:app, :whenever_scheduler],   cluster: 'c1'
server "application_server3",         roles: [:app],                        cluster: 'c2'
server "application_server4",         roles: [:app],                        cluster: 'c2'

server "load_balancer1",              roles: [:web],                        cluster: 'c1'
server "load_balancer2",              roles: [:web],                        cluster: 'c2'
```

In the server based deployment definitions, it's much easier to see which roles each individual server is playing.

As an added bonus, you can assign arbitrary attributes to servers in the second definition. These attributes can be used in your capistrano tasks.

```ruby
namespace :cluster do
  task :c1 do
    set :cluster, 'c1'
    set :other_cluster, 'c2'
    set :filter, :hosts => roles(:all).select { |server| server.properties.cluster == fetch(:cluster) }
  end
end
```

In the example above, we use the built-in `:filter` setting which filters down all servers that will receive commands throughout the deployment process. This allows us to perform cluster based deployments without having to write a cluster-specific conditional in every capistrano task!

## SSHKit Interface

SSHKit is a new gem that capistrano uses under the hood to issue common commands to servers. It contains useful utilities for issuing remote commands during deployment, such as setting environment variables, properly changing directories, and changing users. Since their documentation is so good, I'll refer you to the [SSHKit examples page](https://github.com/capistrano/sshkit/blob/master/EXAMPLES.md) instead of duplicating the code here.

## Finding the Capistrano 3 Public Interface

Common variables that are useful to know can be found in the `Capistrano::DSL::Paths` module. It contains variables like `current_path`, `shared_path`, and `asset_timestamp` – things we found valuable when writing our own custom cap tasks.
Common setters and getters that are useful for scripts can be found in `Capistrano::DSL::Env`. `set`ting and `fetch`ing configuration is there. As is setting servers and roles, and getting a list of roles and/or servers.

Whew! If you made it this far, I applaud you. As you can see, the capistrano 3 overhaul is a pretty big undertaking with some drastic changes. Thankfully, after upgrading you'll notice that making changes or improvements to your deployments process becomes much easier.

[&mdash; James](http://wanelo.com/james)

