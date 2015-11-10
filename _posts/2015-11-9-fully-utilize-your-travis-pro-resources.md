---
layout: default
title: "Fully Utilize your Travis Pro Resources by Partitioning your RSpec Build"
author: James Hart
author_username: james
---

## Split your RSpec build into N equal parts with [rspec-parts][rspec-parts].

We've been subscribed to the [Travis Pro][travis-pro] service for our CI process for a while. We pay for the Startup plan,
 which gives us 5 concurrent executors at any given time. Over the last two years, we've slowly been pecking away at converting
 our [minitest][minitest] suite to [RSpec][rspec]. Within the last month, we finally bit the bullet and devoted some developer time to finishing
 that task. Since we moved from three test suites (minitest, RSpec, and Jasmine) to just two, we had some
 concurrency available that was not being fully utilized for a single build.

You can see our _impressively_ long 1 hr and 3 minute build below.

![One RSpec Build](/assets/travis_pro_resources/one_RSpec_build.png)

Amazing, right? Our goal then was to take our exceptionally long RSpec build and split it into four evenly split concurrent jobs, thereby
using all five executors for one build, allowing us to deploy much quicker.



## Let's make it faster!

At first we initialized a `Rake::FileList` on the entirety of the `spec/` directory. The `Rake::FileList` class is
particularly useful, allowing us to easily manipulate lists of files.
 Avdi Grimm has a [good blog post][rake-file-lists] that dives into the details of the class, and covers it better than I will here.
The `Rake::FileList` returns an enumerable of all files within the `spec/` directory, which we can then split into N even
groups with [`#in_groups`][in-groups]. Let's run that on Travis now.

![Lopsided RSpec Builds](/assets/travis_pro_resources/lopsided_RSpec_builds.png)

As you can see, it's pretty lopsided. The entire directory for `lib/` and `finder/` specs, which are considerably faster than the rest got grouped
together and ended up running in eight minutes. Another build that contained the entirety of the `features/` directory took over
20 minutes to complete.

So, how do we try to split this more evenly?

## Split Smart

We can assume that each file within a subdirectory of `spec/` will run in about the same amount of time
(e.g. `spec/models/a_spec.rb` and `spec/models/b_spec.rb` will be similar in run times relative to one another, as will `spec/features/a_spec.rb` and `spec/features/b_spec.rb`).
So, we'll iterate over each subdirectory and break them into even groups. Since we want to split into four jobs, we'll take the `models/` directory, split it into four even groups.
Then we'll take the `views/` directory, and split that into four groups. Similarly with `controllers/`, and every other subdirectory within `spec/`.
Those groups will then be added back to a master `Rake::FileList` which will then be run by rspec.

Simple enough premise, but let's see how this method fairs:

![Evenly Divided RSpec Builds](/assets/travis_pro_resources/evenly_divided_RSpec_builds.png)

As you can see, the job times are more evenly distributed than the last build, and we've reduced the time we need to wait for a single
build to finish from over an hour and change to just under 20 minutes.

So, how did we do it?

### Travis Configuration

The important part of our `.travis.yml` file looks like this now:


```yaml
env:
  - TEST_SUITE=rspec RSPEC_PART=1 RSPEC_GROUPS=4
  - TEST_SUITE=rspec RSPEC_PART=2 RSPEC_GROUPS=4
  - TEST_SUITE=rspec RSPEC_PART=3 RSPEC_GROUPS=4
  - TEST_SUITE=rspec RSPEC_PART=4 RSPEC_GROUPS=4
  - TEST_SUITE=jasmine
script: "script/ci-travis-$TEST_SUITE"
```

Each element in the `env` array represents one build, so you can see how we split the RSpec builds into four parts.
The `script/ci-travis-rspec` file is an executable script which looks like this:

```bash
#!/bin/bash

xvfb-run -e /dev/stdout bundle exec rake spec:part[$RSPEC_PART,$RSPEC_GROUPS]
```

And that's it!

The essence of that script lies in the `rake spec:part` command. It's using the [`rspec-parts` gem][RSpec-parts]
which we developed here at Wanelo. It takes two arguments, the first being the part to run, the second being the number of
groups to split the suite into. To run build one of two you'd run `rake spec:part[1,2]`, for build two `rake spec:part[2,2]`.
If we wanted to upgrade our Travis CI plan to give us 10 concurrent executors, we could simply modify the command to
`rake spec:part[1,9]`, still accouting for our one Jasmine build.

The rspec-parts gem is [available on github][RSpec-parts] and is pretty well documented as to how to use it in the README.

If you've got any questions on how to use it, feel free to poke us in the comments or [post an issue][github-issue] on github.

&mdash;[James](http://wanelo.com/james)


[travis-pro]: https://travis-ci.com/plans
[minitest]: https://github.com/seattlerb/minitest
[rspec]: https://github.com/rspec/rspec
[rspec-parts]: https://github.com/hjhart/rspec-parts
[github-issue]: https://github.com/hjhart/rspec-parts/issues
[rake-file-lists]: http://devblog.avdi.org/2014/04/22/rake-part-2-file-lists/
[in-groups]: http://apidock.com/rails/ActiveSupport/CoreExtensions/Array/Grouping/in_groups
