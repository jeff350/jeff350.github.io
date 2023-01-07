layout: post
title: "Leveraging SaltStack formulas with Gitfs"
date: 2023-01-06
categories: Salt saltstack-formulas Config-Management 

<a href="https://docs.saltproject.io/salt/user-guide/en/latest/index.html">Salt</a> is a confiruration management tool that I use in my homelab.

In this post I am going to discuss how I leverage the <a href="https://github.com/saltstack-formulas">salt formulas</a> maintained by the salt formulas team. This team maintains a large number of formulas that are designed to be plug and play with the saltstack ecosystem.

In order to leverage the salt formulas they must be included from one of the <a href="https://github.com/saltstack/salt/tree/master/salt/fileserver">fileserver backends</a>. Available options are:

- Azure blob storage service
- git
- Mercurial
- minionfs (backend which serves files pushed to the Master)
- roots (default: Master's local filesystem)
- Amazon s3 buckets
- SVN

The easiest and quickest way to use salt formulas is to use the `roots` fileserver backend and simply clone the git repo's from the github repos to your salt master at `/srv/salt` but this is less than ideal. 

The salt formula's team recommends forking their formula repos to ensure that it will not change you with out you exlicitly updating the code. For me this mean's mirroring the formulas that I want to use on my salt master into an internal GitLab server. But for the sake of simplicty I will be showing how to set this up directly with the upstream repos.

We will specifically look at these 4:
- https://github.com/saltstack-formulas/apt-cacher-formula
- https://github.com/saltstack-formulas/openssh-formula
- https://github.com/saltstack-formulas/salt-formula
- https://github.com/saltstack-formulas/sudoers-formula

The first step is <a href="https://docs.saltproject.io/en/latest/topics/tutorials/gitfs.html#installing-dependencies">Installing Dependencies</a> This will will be evaluating if pygit2 and GitPython will work better for the OS that your salt master is installed on.

I am going to leverage pygit2. To install this dependency on a onedir install of salt.

```
salt-pip install pygit2
```

Next we will enable the git fileserver backend and configure salt to use pygit2 in the salt master config /etc/salt/master.d/gitfs.conf by adding

```
gitfs_provider: pygit2

fileserver_backend:
  - git
  - roots
```

At this point everything is configured and we are ready to add our states. To do this we need to simply add them as gitfs_remotes in the salt master config.

```
gitfs_remotes:
  - https://github.com/saltstack-formulas/sudoers-formula.git:
    - root: sudoers
    - mountpoint: salt://sudoers
    - disable_saltenv_mapping: True
    - base: v0.25.0
  - https://github.com/saltstack-formulas/apt-cacher-formula.git:
    - root: apt-cacher/ng
    - mountpoint: salt://apt-cacher/ng
    - disable_saltenv_mapping: True
    - base: v0.7.3
  - https://github.com/saltstack-formulas/openssh-formula:
    - root: openssh
    - mountpoint: salt://openssh
    - disable_saltenv_mapping: True
    - base: v3.0.3
  - https://github.com/saltstack-formulas/salt-formula:
    - root: salt
    - mountpoint: salt://salt
    - disable_saltenv_mapping: True
    - base: v1.12.0

```

Note that we are adding a few extra config options here.
- `root`: specifies files to be served from a subdirectory within the repository. We want this since these formulas have a lot of testeing files that are not needed for execution of the state. including these would add unnecessary files to the salt fileserver. 
- `mountpoint`: prepend the specified path to the files served from gitfs. we use this to ensure states can be called by the proper names
- `disable_saltenv_mapping`: disables saltenv mappings for specific branches since we just want the latest release.
- `base`: defines the specific release to use, this will ensure a new version is not automatically pulled down


from here you should be good to go. You can now configure the pillar data for each of these states and run them against your minions.
