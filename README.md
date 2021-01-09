# Personal Website

This repo contains the stuff making up my personal site.

### Current Setup
- hosted with GitHub Pages
- uses [Jekyll](http://jekyllrb.com) with the Hyde theme based on [Poole](http://getpoole.com)

### Development Workflow
Jekyll is a static site generator. With that in mind, you need to build the static files anytime you want to make change.

##### Making a Post
Write the post content in a markdown file in the [\_posts](_posts) directory. The publish date and title should be stored in the markdown filename; for example
[2016-12-14-file-permissions-for-django-media-uploads.md](_posts\2016-12-14-file-permissions-for-django-media-uploads.md).

When your satisfied with that beautiful prose, we can build the static files with Jekyll. Because we have a `Gemfile`, we need to build like this:

```bash
bundle exec jekyll serve
```   

This should start a local server on which we can
view the changes:
```
Configuration file: /home/adam/projects/aerispaha.github.io/_config.yml
            Source: /home/adam/projects/aerispaha.github.io
       Destination: /home/adam/projects/aerispaha.github.io/_site
 Incremental build: disabled. Enable with --incremental
      Generating...
       Jekyll Feed: Generating feed for posts
                    done in 0.316 seconds.
 Auto-regeneration: enabled for '/home/adam/projects/aerispaha.github.io'
    Server address: http://127.0.0.1:4000/
  Server running... press ctrl-c to stop.

```

When all looks good, we can publish the changes by merging the static file changes into the master branch (the branch configured in Github to host the GH pages content)
