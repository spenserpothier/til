---
layout: post
title:  "Chef Test Kitchen Converge with Private Cookbook Dependencies"
date: Thu Apr 21 11:58:44 2016
categories: chef test-kitchen devops
---

Today I was attempting to write a wrapper cookbook that used some private cookbooks. In an effort to do TDD and not waste AWS resources, I was also using test-kitchen. When running `chef exec kitchen converge default-centos-72` (using centos 7.2 with a default suite) I kept getting the following error:

```
-----> Chef Omnibus installation detected (install only if missing)
       Transferring files to <default-centos-72>
       Starting Chef Client, version 12.9.38
       resolving cookbooks for run list: ["bastion-host::default"]

       ================================================================================
       Error Resolving Cookbooks for Run List:
       ================================================================================

       Missing Cookbooks:
       ------------------
       No such cookbook: sssd

       Expanded Run List:
       ------------------
       * sssd-wrapper::default

       Platform:
       ---------
       x86_64-linux
```
From this output I figured the cookbook wasn't getting downloaded (since in my `.kitchen.yml` file it's provisioning using `chef_zero`) and that I needed to get the cookbooks somewhere local so that kitchen could copy them onto the VM to run the convergence.
I discovered that `berks` has a `vendor` command that would allow me to download the cookbooks to a directory, and then I found that I could add a `cookbook_path` to my `.kitchen.yml` for kitchen to look for cookbooks.

Below are the additions I made:


```ruby
# Berksfile
cookbooks 'sssd', git: 'https://my.company.com/cookbooks/sssd.git'
# Cookbook custom-ntp is a dependency of sssd, which is also in a private repo
cookbooks 'custom-ntp', git: 'https://my.company.com/cookbooks/custom-ntp.git'
```

```ruby
# .kitchen.yml
provisioner:
  name: chef_zero
  cookbooks_path:
    - ./cookbooks
```

Here is a snippet of the directory structure before I started:

```
.
├── Gemfile
├── .kitchen.yml
├── bin
├── cfn
├── cookbooks
│   └── sssd-wrapper
│     └── Berksfile

```

I then went to my sssd-wrapper directory and ran `berks vendor ../` and that gave me the following:

```
.
├── Gemfile
├── bin
├── cfn
├── cookbooks
│   ├── sssd-wrapper
│   ├── chef-vault
│   ├── chef_handler
│   ├── custom-ntp
│   ├── ntp
│   ├── sssd
│   └── windows
```

At this point I was able run my original command and all of the cookbooks were successfully run and I could carry on with testing my wrapper cookbooks attributes as needed
