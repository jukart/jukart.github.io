=========
Making Of
=========

This page is hosted on `github <https://github.com>`_ and built using
`sphinx <http://sphinx-doc.org>`_.

Here is how I managed to build and deploy this page.

Github Repository
=================

First you need to setup a new repository with the name <gituser>.github.io.


Managing The Repository And Deployment
======================================

This was inspired by `Nikhil Marathe's blog <http://blog.nikhilism.com/2012/08/automatic-github-pages-generation-from.html>`_

Building for a user-page is a little bit different than building a page for a
repository. A user-page is rendered from the `master` branch not from
`gh-pages`. So I reverted the use of the branches in the Makefile to have the
sources in the branch `page` and build into `master` branch.

First create a new orphaned branch::

    >>> git checkout --orphan page
    >>> git rm -rf .

Now create the initial sphinx files::

    >>> sphinx-quickstart

Commit all the files.

Now add this to the end of your Makefile::

    # All files/directories needed for deployment
    PAGE_SOURCES = source Makefile

    deploy:
        git checkout master
        rm -rf build _sources _static
        git checkout page $(PAGE_SOURCES)
        git reset HEAD
        make html
        mv -fv build/html/* ./
        rm -rf $(PAGE_SOURCES) build
        git add -A
        git commit -m "Generated page for `git log master -1 --pretty=short --abbrev-commit`" && git push origin master ; git checkout page

With `make deploy` you are now able to completely deploy you page into
`master` branch::

    >>> make deploy

