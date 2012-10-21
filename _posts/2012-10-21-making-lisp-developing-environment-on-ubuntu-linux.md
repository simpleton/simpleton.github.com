---
layout: post
title: "Making lisp developing environment on Ubuntu Linux"
description: ""
category: emacs
tags: [emacs, lisp]
---
{% include JB/setup %}
# Install SBCL#
SBCL is almost the most popular Common Lisp implementation available for Linux.

	sudo apt-get install sbcl

# Install Emacs#
Using apt-get too. It quite easy~.

# Download and install QuickLisp#
You can Download Quicklisp from [quicklisp.lisp](http://beta.quicklisp.org/quicklisp.lisp).(It's in Beta version currently, but it's fully functional and really awesome).

nNext, run sbcl with Qucklisp.lisp

	sbcl --load quicklisp.lisp


After it has loaded, run command:

	(quicklisp-quickstart:install)

This command will download the whole quicklisp system and get it set up for you.Quicklisp will install by default in `~/quicklisp`; and you can modify the default path by passing `:path /the/path/your/wanted` to install quicklisp.

Finally , you should run :

	(ql:add-to-init-file)

That'll add quicklisp to your initial config file, so that as you run sbcl Quicklisp will be loaded and ready to work.

# Configure quicklisp in emacs environment#
Now, sbcl and quicklisp are already work perfectly together.And then you should make both work in emacs environment.

Firstlly, run this command in sbcl:

	(ql:quickload "quicklisp-slime-helper")

Then,you need add followings to `~/.emacs`:

	(setq inferior-lisp-program "sbcl")
	(load (expand-file-name "~/quicklisp/slime-helper.el"))

Finally, you can use quicklisp in slime environment.

