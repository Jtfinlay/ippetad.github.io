---
layout: post
title: "How to Set up a Multiple Author Blog"
author: Andrew Fontaine
author-img: andrew_fontaine.png
date: 2014-04-09 19:36:15 -0600
comments: true
categories: setup
---

Hello, fellow bloggers. I've spent some time working on this now. Let's get all
this down before I forget what I'm doing.

<!-- more -->

Setup
-----

Basic setup is the same. Clone the source, start filling out `_config.yml`, and
get it to a working point. In the `Rakefile`, make sure your `new_post` looks
like the following:

{% coderay lang:ruby %}
# usage rake new_post[my-new-post] or rake new_post['my new post'] or rake new_post (defaults to "new-post")
desc "Begin a new post in #{source_dir}/#{posts_dir}"
task :new_post, :title do |t, args|
  if args.title
    title = args.title
  else
    title = get_stdin("Enter a title for your post: ")
  end
  raise "### You haven't set anything up yet. First run `rake install` to set up an Octopress theme." unless File.directory?(source_dir)
  mkdir_p "#{source_dir}/#{posts_dir}"
  filename = "#{source_dir}/#{posts_dir}/#{Time.now.strftime('%Y-%m-%d')}-#{title.to_url}.#{new_post_ext}"
  if File.exist?(filename)
    abort("rake aborted!") if ask("#{filename} already exists. Do you want to overwrite?", ['y', 'n']) == 'n'
  end
  puts "Creating new post: #{filename}"
  open(filename, 'w') do |post|
    post.puts "---"
    post.puts "layout: post"
    post.puts "title: \"#{title.gsub(/&/,'&amp;')}\""
	# This is the important bit. This'll make sure your authors fill their name in.
	post.puts "author: "
    post.puts "date: #{Time.now.strftime('%Y-%m-%d %H:%M:%S %z')}"
    post.puts "comments: true"
    post.puts "categories: "
    post.puts "---"
  end
end
{% endcoderay %}

Basically, it adds an `author` field to your front-matter. Important for keeping
track of who wrote what.

Deployment
----------
We're using [GitHub Pages](https://pages.github.com/) to do the deployment,
which means we can take advantage of forks and pull requests as a sort of
peer-review.

**NOTE:** Make sure only one person is in charge of deploying the blog. You may
find you'll run into asset issues, because git doesn't like binary files very
much.

Your writers will fork the main blog repository, run `rake new_post[...]`, fill
out their post, commit and push. In GitHub, they'll open a pull request, and you
can check it out. Once you're ready, merge it in, and deploy it.

For the Authors
---------------
I'm assuming you're git-savvy, so I pardon if your not.

Basically, you'll need ruby ([installer](http://rubyinstaller.org/) for
Windows), the ruby DevKit (same location, follow
[these instructions](https://github.com/oneclick/rubyinstaller/wiki/Development-Kit)),
and [RubyGems](http://rubygems.org/pages/download).

Get into the command line, find the spot you want to hold all the files for the
blog, and run:

{% coderay lang:bash %}
git clone THE_REPO_URL DIR source
{% endcoderay %}

This'll clone the repository down and checkout the `source` branch. Get into the
new directory, and run `bundle install`. This installs all the gems you'll need
to use and preview the blog. To make a post, do the following:

{% coderay lang:bash %}
rake new_post[$TITLE]
$EDITOR source/_posts/$DATE-$TITLE
{% endcoderay %}

To check it out and make sure it looks all right:

{% coderay lang:bash %}
rake generate
rake preview
{% endcoderay %}

Then, head over to `localhost:4000`. When everything looks good, commit and push
to your fork. Open a pull request against the main repository's `source` branch,
and let the review happen.

I hope that wasn't too complicated. This is pretty long for a new blog.

Cheers,

Andrew
