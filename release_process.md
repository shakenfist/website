Shaken Fist's release process
=============================

Shaken Fist is now split across a number of repositories to simplify development and usage.
Unfortunately, that complicated the release process. This page documents the current release
process.

Stage 1: Testing
----------------

Before release, you should ensure that all of the clouds currently supported in the deploy
repository work. These are important to new users and testers, so we should keep them working.
For reference, they are at ```deploy/ansible/terraform```. At the time of writing those are:

* aws
* aws-single node
* gcp
* metal
* openstack

Note that to test these you need access to all those clouds, as well as needing to know the
cloud specific values for each cloud. Unfortunately I can't tell you those because they vary
with your specific cloud accounts.

Stage 2: Tag and release shakenfist/shakenfist
----------------------------------------------

Tag the release, and push it to github:

```
git tag -s 0.1 -m "Release v0.1"
git push origin 0.1
```

Create the branch for cherry picks and minor releases:

```
git checkout -b 0.1-devel
git push origin 0.1-devel
```

Stage 3: Tag and release shakenfist/deploy and shakenfist/client-go
-------------------------------------------------------------------

As for shakenfist/shakenfist, but in the different repos.

Stage 4: Bump dependancy in shakenfist/terraform-provider-shakenfist, and then release
--------------------------------------------------------------------------------------

Bump the version number client-go in go.mod, and then release as per above.

TODO
----

We should be pushing to pypi as well.
