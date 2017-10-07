---
title: "Blog Setup - Deployment"
date: 2017-09-22T12:37:43+02:00
draft: true
---

Since my last blog post I did some modifications to my setup on the Pi3, so I
decided to turn the content of the previous post into a small series. In this
particular post I want to talk about the work I did to ease the deployment of
new material on my website using a Git hook. For this, I followed and adapted a
little bit [this][digital-ocean] guide provided by Digital Ocean.

[digital-ocean]: https://www.digitalocean.com/community/tutorials/how-to-deploy-a-hugo-site-to-production-with-git-hooks-on-ubuntu-14-04

## Hugo Installation
Since I want to push the code for the website through Git on my Pi3 and then
automatically compile and deploy it in the appropriate folder, the first step
is to get a working Hugo installation on my Raspberry Pi.

This operation deserves a separate section since it turned out to be a little
bit more complex than I first expected. Indeed, after installing the `hugo`
package provided by Raspbian, I discovered that it was almost one year old and
that the theme I chose did not support it. As a consequence, I went looking for
a newer version on [GitHub][hugo-releases], but unfortunately they did not have
the appropriate precompiled package for my Pi3; as a consequence, I had to
compile the whole thing from scratch.

The first step for compilation was to get the [Go][go] compiler and then, after
that, use it to download and compile Hugo and its dependecies. The sequence of
commands I used to do that was:

	$ sudo apt install golang-go
	$ GOPATH=~/hugo go get -v -u github.com/gohugoio/hugo

After waiting a considerable amount of time for the compilation to finish, I
was able to install the `hugo` executable on my system using the following
command:

	$ cp ~/hugo/bin/hugo /usr/local/bin/

[hugo-releases]: https://github.com/gohugoio/hugo/releases
[go]: https://golang.org/

## Git Hook
After verifying that my Hugo installation was working, I cloned a bare version
of my local repositories, one for the theme and the other for the website, and
put them on my server. This can be done as follows:

	$ git clone --bare <name> /tmp/<name>.git
	$ scp -r /tmp/<name>.git <user>@<server-url>:website/<name>.git

After that, I added the repository on the server as a new remote, called
`prod`, on my local machine in order to distinguish from the backup copy on
GitHub. The following command allows to do so and verifies that everything
works as it should:

	$ git remote add prod <user>@<server-url>:website/<name>.git
	$ git ls-remote prod

In addition, I modified my website repository so that the theme is pointed to by
a symlink and not by a submodule. In this way, if the two repositories are in
the same directory, everything works as it should.

After this step, I wrote my Git hook, which turned out to be as follows:

{{<highlight bash>}}
	#!/bin/bash

	SOURCE_DIR=$HOME/website
	WORKING_DIRECTORY=/tmp/website_working_dir
	WEBSITE_REPO=missing_fantasy_exception
	THEME_REPO=hugo_theme_pickles
	PUBLIC_WWW=/var/www/html
	BACKUP_WWW=$HOME/website/backup
	MY_URL="https://egeretto.h4ck.me/"

	set -e

	rm -rf $WORKING_DIRECTORY
	rsync -aqz $PUBLIC_WWW/ $BACKUP_WWW
	trap "echo 'A problem occurred.  Reverting to backup.';\
		rsync -aqz --del $BACKUP_WWW/ $PUBLIC_WWW;\
		rm -rf $WORKING_DIRECTORY" EXIT

	mkdir -p $WORKING_DIRECTORY
	git clone $SOURCE_DIR/$WEBSITE_REPO.git $WORKING_DIRECTORY/$WEBSITE_REPO
	git clone $SOURCE_DIR/$THEME_REPO.git $WORKING_DIRECTORY/$THEME_REPO
	rm -rf $PUBLIC_WWW/*
	hugo -s $WORKING_DIRECTORY/$WEBSITE_REPO -d $PUBLIC_WWW -b $MY_URL
	rm -rf $WORKING_DIRECTORY
	trap - EXIT
{{</highlight>}}

As you can see, it should be able to handle gracefully errors that happen
during the building of the website and restore a backup copy that is created at
the beginning of the process. Apart from a couple of tweaks made by me due to
having two repositories instead of one, it is quite similar to the script
provided in the original guide.

As a side note, for all this to work I set the permissions of `/var/www/html/`
to `<user>:<user>`, since the user that handles the Git repository is the only
one that is going to write in it. The important thing is not to give write
access for that directory to the user with which the webserver is run, since in
that case getting control of the execution inside of the webserver would allow
for a direct modification of the website.

Once the script is working, it is sufficient to place it in
`<website-repo>.git/hooks/post-receive` for it to be executed every time new
commits are pushed to the repository thus triggering a rebuild.
