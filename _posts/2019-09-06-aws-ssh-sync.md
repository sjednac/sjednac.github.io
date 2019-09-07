---
layout: post
title: "Syncing SSH config with AWS"
excerpt: "My take on accessing EC2 instances in an ever-changing server environment."
tags: [ssh, amazon, aws, ec2, cloud, infrastructure, devops, automation]
modified: 2016-09-17
comments: true
---

Most of my recent projects are dependent on a multi-region, [AWS](https://aws.amazon.com/ec2/) environment, that is spawned using an infrastructure-as-code approach (usually [Terraform](https://www.terraform.io/)). The servers are managed by [AutoScaling Groups](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html) and can be terminated at any point in time, within some reason of course. 

This is pretty great, as it allows us to spawn a complete new environment in a relatively short time, and gives the benefit of self-healing and horizontal scaling on demand. Assuming you have the right tooling for monitoring and logging around it - everyone is happy.

That being said, the best way to debug "hard" problems is - still - a direct connection using [SSH](https://www.freebsd.org/doc/handbook/openssh.html) to a problematic node (at least in my book). The thing is, that while the environment structure is well-defined, the target IP may be long out of date (servers can come and go) and - on top of it - connection params may vary across the environment itself. For example, you may want to use a different credentials, setup a dedicated [bastion host](https://aws.amazon.com/quickstart/architecture/linux-bastion/) or do something SSH-specific for a **subset** of instances in general.

I usually end up with a lot of shell scripts around such setup, so I decided to unify the approach with a small [Python script](https://github.com/sbilinski/aws-ssh-sync) instead.

![](https://imgs.xkcd.com/comics/automation.png){: .center-image }

## Features overview

I've looked into [gianlucaborello/aws-ssh-config](https://github.com/gianlucaborello/aws-ssh-config) before going for my own implementation, but it didn't quite fit my requirements.

Here's what I ended up with:

* Always query an explicit list of AWS regions. 
* Filter EC2 instances, if needed. This allows to apply certain SSH settings to a **subset** of servers.
* Index any duplicates, so that an **AutoScaling Group** with the following nodes `[foo,foo,foo]`, can be treated as `[foo0,foo1,foo2]`.
* Take **all** config settings directly from the command line. For example: implicit AMI to user mapping is intentionally not provided. The idea is to mimic a **functional** approach for obtaining this kind of config. You provide some params, call the script and get the active config in return.
* Write config to `stdout` or to a shared config file (read ahead).
* Use sensible defaults where possible (e.g. `ec2-user` for the username).
* Test each release using `botocore` stubber to mimic actual AWS behavior. 
* Publish to [PyPI](https://pypi.org/project/aws-ssh-sync/) for a (hopefully) painless setup.

## File output

While writing to `stdout` is how the world is supposed to work, it's not always practical. In a perfect scenario, the config structure would look as follows:

```
~/.ssh/config
~/.ssh/config.d/workproject-staging
~/.ssh/config.d/workproject-production
~/.ssh/config.d/homestuff
...
```

The `~/.ssh/config` file would include the following directive in such case:
```
Include config.d/*
```

I've struggled with **Oh My Zsh** and **Visual Studio Code** integration issues within such approach, so I decided to provide an alternate output mode - writing to a file with `config-key` substitution (so that you can call the script repeatedly, with different params, and use a shared config). I know - fixing `Include` support upstream is probably a better idea, but this is literally a few lines of code in Python (+ testing).

Note that this feature is completely **optional**.

## Synchronization

The general idea is to run a **single command** with different params (i.e. potentially multiple times) to get all your infrastructure state in one place. You can setup a `cron` job for it or have a handy alias on the CLI, that you can execute at any point when you need it. 

In my case, I use the latter approach, and put something like this in my shell's profile:

```bash
# Slightly rude wrapper for AWS SSH Sync
function ass() {
    PROFILES=("stg" "prd")
    REGIONS=("eu-west-1" "us-east-1")

    for PROFILE in "${PROFILES[@]}"; do
        aws_ssh_sync --profile $PROFILE \
                     --region "${REGIONS[@]}" \
                     --address "public_private" \
                     --config-key $PROFILE \
                     --output-file "$HOME/.ssh/config" \
                     --name-prefix $PROFILE \
                     --region-prefix \
                     --server-alive-interval 90 \
                     --skip-strict-host-checking
    done
}
```

In this example, each `PROFILE` has a single `aws_ssh_sync` command associated to multiple `REGIONS` at once. You could extend it to as many calls as you want of course (e.g. to include the `--ec2-filter-name` switch). 

After executing `ass`, all outputs will be written to `$HOME/.ssh/config`, giving a unified view of the infrastructure for your shell and any other tools that depend on it.

![](/images/aws-ssh-sync.gif){: .center-image }

## Conclusion 

The script currently supports a limited subset of `ssh_config(5)` params, that I use on a daily basis. If you're missing something - please feel free to open a [pull request](https://github.com/sbilinski/aws-ssh-sync/blob/master/CONTRIBUTING.md). 

The project can be found on [GitHub](https://github.com/sbilinski/aws-ssh-sync).
