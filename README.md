wanelo.github.io
================

Wanelo Engineering Blog.

To run locally:

```bash
hub clone wanelo/wanelo.github.io
cd wanelo.github.io
bundle
jekyll serve --watch --port 4000
```

To create a draft (drafts do not get published to github):

Create a file in the _drafts folder e.g.

```bash
touch _drafts/2015-02-03-decoupling-distributed-ruby-applications-with-rabbitmq.md
```

You can check this file in and push to github. Only when you move it to the `_posts` directory will it be published!
