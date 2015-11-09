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

Impressive, right? Our goal then was to take our exceptionally long RSpec build and split it into four evenly split concurrent jobs, thereby
using all five executors for one build, allowing us to deploy recently pushed code much quicker.



## Let's make it faster!

At first we initialized a [`Rake::FileList`][rake-file-lists] on the entirely of the `spec/` directory,
which returned an enumerable that would be easy to use my new favorite method,
[`#in_groups`][in-groups], to split an array into N even groups.

![Lopsided RSpec Builds](/assets/travis_pro_resources/lopsided_RSpec_builds.png)

As you can see, it's pretty lopsided. The entire directory for `lib/` and `finder/` specs, which are considerably faster than the rest got grouped
together and ended up running in eight minutes. Another build that contained the entirety of the `features/` directory took over
20 minutes to complete.

So, how do we try to split this more evenly?

## Split Smart

We can assume that each file within a subdirectory of `spec/` will run in about the same amount of time
(e.g. `spec/models/a_spec.rb` and `spec/models/b_spec.rb` will be similar in run times relative to one another, as will `spec/features/a_spec.rb` and `spec/features/b_spec.rb`).
We iterated over each subdirectory on `spec/` and split those evenly into N groups. So, if the `models/` and `features/` directories contains
four files each, and we want to split the build into four jobs, each job will run one spec file in each directory. You can
[see the implementation here][file-list-implementation]. Let's see how this method fairs.

![Evenly Divided RSpec Builds](/assets/travis_pro_resources/evenly_divided_RSpec_builds.png)

As you can see, the job times are more evenly distributed than the last build, and we've reduced the time we need to wait for a single
build to finish from over an hour and changed to just under 20 minutes.

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

function error_exit {
  echo "${PROGNAME}: ${1:-"Unknown Error"}" 1>&2
  exit 1
}

xvfb-run -e /dev/stdout bundle exec rake spec:part[$RSPEC_PART,$RSPEC_GROUPS] || error_exit 'RSpec asplode!'
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
[file-list-implementation]: https://github.com/hjhart/rspec-parts/blob/93344d785c6b3e2dcf9d1c0c8c393abef3473ee6/lib/rspec/parts/file_list.rb#L33-L50
