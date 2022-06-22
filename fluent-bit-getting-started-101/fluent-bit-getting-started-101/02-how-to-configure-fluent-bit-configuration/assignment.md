---
slug: how-to-configure-fluent-bit-configuration
id: 52kuzz1aeahq
type: challenge
title: Configuring Fluent Bit to read from a file
teaser: In this challenge we will configure Fluent Bit to read from a file
tabs:
- title: Terminal
  type: terminal
  hostname: fluent-bit-sandbox
- title: Fluent Bit Configuration
  type: code
  hostname: fluent-bit-sandbox
  path: /fluent-bit/etc/
- title: Fluent Bit Documentation
  type: website
  hostname: fluent-bit
  url: https://docs.fluentbit.io/manual/
difficulty: basic
timelimit: 600
---
🤖 Introduction
==============

In the last lesson we walked through how to run Fluent Bit from a command line. When running in production however there are many parameters that you may need to modify to gain maximum performance. In this lesson we will follow one of the most common scenarios for Fluent Bit - reading from a file.

In this case we will use a standard Apache Access log that's under the path `/var/log/access.log`

Accessing the configuration
==============

You will notice a new tab (Fluent Bit Configuration) that is now shown next to your terminal window. This points to the `/fluent-bit/etc/` directory. Additionally, you have the official Fluent Bit documentation in a tab window as well (Fluent Bit Documentation)

Within this directory you will find the `/fluent-bit/etc/fluent-bit.conf` file that you can modify. This directory also contains the default parsers file `parsers.conf` that contains out of the box parsers like docker, json, apache2, cri, and more.

Note: When modifying the configuration in the Fluent Bit Configuration tab you will need to click the save icon on the tab to save changes

Reading the configuration
==============
The `/fluent-bit/etc/fluent-bit.conf` shows the default configuration that Fluent Bit ships with when retrieving it from a Linux package. To better understand the configuration we need to understand the three major parts `[INPUT]`, `[FILTER]`, `[OUTPUT]`. These all designate the type of plugins configured


Modifying the configuration
===============
Let's add a new INPUT plugin to read the syslog data in from the beginning. To do this under the existing INPUT section we will add the following code block

```
[INPUT]
    name tail
    path /var/log/access.log
    read_from_head true
    parser apache2
```

Running Fluent Bit
=================
Let's run Fluent Bit the same way as we did before and this time use the parameter `-c` to indicate a configuration file.

```
/fluent-bit/bin/fluent-bit -c /fluent-bit/etc/fluent-bit.conf
```

Once complete we should see the lines of the Apache Access Log print on screen with JSON fields

