---
layout: default
title: "Fully Utilize your Travis Pro Resources"
author: James Hart
author_username: james
---

### Split your rspec build into N equal parts.

We've been subscribed to the Travis Pro service for a while for our CI process for a while. We pay for the Startup plan,
 which gives us 5 concurrent executors at any given time. Over the last two years, we've slowly been pecking away at converting
 our minitest suite to rspec. Within the last month, we finally bit the bullet and devoted some developer time to finishing
 that task. Since we were moving from three test suites (minitest, rspec, and jasmine) to just two, we had some
 concurrency available that was not being fully utilized in any given build.

 Our goal then was to take our exceptionally long rspec build and split it into four evenly split concurrent jobs.

![One Rspec Build](/assets/travis_pro_resources/one_rspec_build.png)

At first we tried splitting it based on a Rake::FileList, which returns an enumerable that is easy to use ActiveSupport's
#groups_of function to split an array into N even groups.

![Lopsided RSpec Builds](/assets/travis_pro_resources/lopsided_rspec_builds.png)

As you can see, it's pretty lopsided. Our `lib/` and `finder/` specs, which are considerably faster than the rest got grouped
together and ended up running in eight minutes alongside builds which contained the `feature/` directories that took over
20 minutes.

So, how do we try to split this more evenly?

Under the assumption that each directory's tests run at similar speeds relative to one another, we iterated over each one
and split those evenly into N groups. So, if the `models/` and `features/` directories contains four files each, and we want to
 split the build into four jobs, each job will run one spec file in each directory.

![Evenly Divided RSpec Builds](/assets/travis_pro_resources/evenly_divided_rspec_builds.png)

As you can see, we're more evenly distributed than the last build, and we've reduced the time we need to wait for a single
build to finish from over an hour and changed to just under 20 minutes.

Great!

So, we've packaged this gem up into something called `rspec-parts`. It's [available on github][rspec-parts] and pretty well documented
as to how to use it in the `README.md`.

If you've got an questions on how to use it, feel free to poke us in the comments or [post an issue][github-issue] on github.

[rspec-parts]: https://github.com/hjhart/rspec-parts
[github-issue]: https://github.com/hjhart/rspec-parts/issues
