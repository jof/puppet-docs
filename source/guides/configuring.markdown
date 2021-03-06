---
layout: default
title: Configuring Puppet
---

Configuring Puppet
==================

Puppet's behavior can be customized with a [rather large collection of settings][configref]. Most of these can be safely ignored, but you'll almost definitely have to modify some of them. 

This document describes how Puppet's configuration settings work, and describes all of Puppet's auxiliary config files.

* * * 

[environments]: ./environment.html
[configref]: /references/stable/configuration.html
[versioned]: /references/

Puppet's Settings
-----------------

Puppet is able to automatically generate a reference of all its config settings (`puppet doc --reference configuration`), and the documentation site includes [archived references for every recent version of Puppet][versioned]. You will generally want to consult the [the most recent stable version's reference][configref]. 

When retrieving the value for a given setting, Puppet follows a simple lookup path, stopping at the first value it finds. In order, it will check:

* Values specified on the command line
* Values in environment blocks in `puppet.conf`
* Values in run mode blocks in `puppet.conf`
* Values in the main block of `puppet.conf`
* The default values

The settings you'll have to interact with will vary a lot, depending on what you're doing with Puppet. But at the least, you should get familiar with the following: 

* [`certname`](/references/stable/configuration.html#certname) --- The locally unique name for this node. If you aren't using DNS names to identify your nodes, you'll need to set it yourself.
* [`server`](/references/stable/configuration.html#server) --- The puppet master server to request configurations from.
* [`confdir`](/references/stable/configuration.html#confdir) --- One of Puppet's main working directories, which usually contains config files, manifests, modules, and certificates.
* [`vardir`](/references/stable/configuration.html#vardir) --- Puppet's other main working directory, which usually contains cached data and configurations, reports, and file backups.
* [`modulepath`](/references/stable/configuration.html#modulepath) --- The search path for Puppet modules.
* [`environment`](/references/stable/configuration.html#environment) --- On agent nodes, the [environment][environments] to request configuration in. 
* [`node_terminus`](/references/stable/configuration.html#nodeterminus) --- How puppet master should get node definitions; if you use an ENC, you'll need to set it to "exec."
* [`external_nodes`](/references/stable/configuration.html#externalnodes) --- The script to run for node definitions (if you chose a `node_terminus` of "exec"). 
* [`report`](/references/stable/configuration.html#report) --- Whether to send reports to the puppet master.
* [`reports`](/references/stable/configuration.html#reports) --- On the puppet master, which report handler(s) to use. 

`puppet.conf`
------------

Puppet's main config file is `puppet.conf`, which is located in Puppet's `confdir`. Under Puppet Enterprise, the confdir is `/etc/puppetlabs/puppet`; on most other systems, the confdir is `/etc/puppet` when running as root or the Puppet user and `~/.puppet` when running as a normal user. 


### File Format

`puppet.conf` uses an INI-like format, with `[config blocks]` containing indented groups of `setting = value` lines. Comment lines `# start with an octothorpe`; partial-line comments are not allowed. 

You can interpolate the value of a setting by using its name as a `$variable`. (Note that `$environment` has special behavior: most of the Puppet applications will interpolate their own environment, but puppet master will use the environment of the agent node it is serving.)

If a setting has multiple values, they should be a comma-separated list. "Path"-type settings made up of multiple directories should use the system path separator (colon, on most Unices). 

Finally, for settings that accept only a single file or directory, you can set the owner, group, and/or mode by putting their desired states in curly braces after the value.

Putting that all together:

    # a block:
    [main]
      # setting = value pairs:
      server = master.puppetlabs.lan
      certname = 005056c00008.localcloud.puppetlabs.lan
      
      # variable interpolation:
      rundir = $vardir/run
      modulepath = /etc/puppet/modules/$environment:/usr/share/puppet/modules
    [master]
      # a list:
      reports = store, http
      
      # a multi-directory modulepath:
      modulepath = /etc/puppet/modules:/usr/share/puppet/modules
      
      # setting owner and mode for a directory:
      vardir = /Volumes/zfs/vardir {owner = puppet, mode = 644}

### Config Blocks

Settings in different config blocks take effect under varying conditions. Settings in a more specific block can override those in a less specific block, as per the lookup path described above. 

#### The `[main]` Block

The `[main]` config block is the least specific. Settings here are always effective, unless overridden by a more specific block. 

#### `[agent]`, `[master]`, and `[user]` Blocks

These three blocks correspond to Puppet's run modes. Settings in `[agent]` will only be used by puppet agent, settings in `[master]` will be used by puppet master and puppet cert, and settings in `[user]` will be used by puppet apply. The Faces subcommands introduced in Puppet 2.7 default to the `user` run mode, but their mode can be changed at run time with the `--mode` option. Note that not every setting makes sense for every run mode, but specifying a setting in a block where it is irrelevant has no observable effect.

Prior to Puppet 2.6, these blocks were called `[puppetd]`, `[puppetmasterd]`, and `[puppet]`, respectively. Although these names still work, their use is deprecated. 

#### Per-environment Blocks

Blocks named for [environments][] are the most specific, and can override settings in the run mode blocks. 

Like with the `$environment` variable, puppet master treats environments differently from the other run modes: instead of using the block corresponding to its own `environment` setting, it will use the block corresponding to each agent node's environment. The puppet master's own environment setting is effectively inert. 

Command-Line Options
--------------------

You can override any config setting at runtime by specifying it as a command-line option to almost any Puppet application. (Puppet doc is the main exception.) 

Boolean settings are handled a little differently: use a bare option for a true value, and add a prefix of `no-` for false:

    # Equivalent to listen = true:
    $ puppet agent --listen
    # Equivalent to listen = false:
    $ puppet agent --no-listen

For non-boolean settings, just follow the option with the desired value: 

    $ puppet agent --certname magpie.puppetlabs.lan
    # An equals sign is optional:
    $ puppet agent --certname=magpie.puppetlabs.lan

Inspecting Settings
-------------------

Puppet agent, apply, and master all accept the `--configprint <setting>` option, which makes them print their local value of the requested setting and exit. In Puppet 2.7, you can also use the `puppet config print <setting>` action, and view values in different run modes with the `--mode` flag. Either way, you can view all settings by passing `all` instead of a specific setting. 

    $ puppet master --configprint modulepath
    # or:
    $ puppet config print modulepath --mode master
    
    /etc/puppet/modules:/usr/share/puppet/modules

Puppet agent, apply, and master also accept a `--genconfig` option, which behaves similarly to `--configprint all` but outputs a complete `puppet.conf` file, with descriptive comments for each setting, default values explicitly declared, and settings irrelevant to the requested run mode commented out. Having the documentation inline and the default values laid out explicitly can be helpful for setting up your config file, or it can be noisy and hard to work with; it comes down to personal taste. 

You can also inspect settings for specific environments with the `--environment` option:

    $ puppet agent --environment testing --configprint modulepath
    /etc/puppet/testing/modules:/usr/share/puppet/modules

(As implied above, this doesn't work in the master run mode, since the master effectively has no environment.)

Other configuration files
-------------------------

In addition to the main configuration file, there are five special-purpose config files you might need to interact with: `auth.conf`, `fileserver.conf`, `tagmail.conf`, `autosign.conf`, and `device.conf`.

### `auth.conf`

Access to Puppet's REST API is configured in `auth.conf`, the location of which is determined by the `rest_authconfig` setting. (Default: `/etc/puppet/auth.conf`.) It consists of a series of ACL stanzas, and behaves quite differently from `puppet.conf`; for full details, see the [REST access control documentation](./rest_auth_conf.html). 

    # Example auth.conf:
    
    path /
    auth any
    environment override
    allow magpie.lan
    
    path /certificate_status
    auth any
    environment production
    allow magpie.lan
    
    path /facts
    method save
    auth any
    allow magpie.lan
    
    path /facts
    auth yes
    method find, search
    allow magpie.lan, dashboard, redmaster.magpie.lan

### `fileserver.conf`

By default, `fileserver.conf` isn't necessary, provided that you only need to serve files from modules. If you want to create additional fileserver mount points, you can do so in `/etc/puppet/fileserver.conf` (or whatever is set in the `fileserverconfig` setting).

`fileserver.conf` consists of a collection of mount-point stanzas, and looks like a hybrid of `puppet.conf` and `auth.conf`:

    # Files in the /path/to/files directory will be served
    # at puppet:///mount_point/.
    [mount_point]
        path /path/to/files
        allow *.domain.com
        deny *.wireless.domain.com

See the [file serving documentation](./file_serving.html) for more details. 

Note that certname globs do not function as normal globs: an asterisk can only represent one or more subdomains at the front of a certname that resembles a fully-qualified domain name. (That is, if your certnames don't look like FQDNs, you can't use `autosign.conf` to full effect.

### `tagmail.conf`

Your puppet master server can send targeted emails to different admin users whenever certain resources are changed. This requires that you:

* Set `report = true` on your agent nodes
* Set `reports = tagmail` on the puppet master (`reports` accepts a list, so you can enable any number of reports)
* Set the `reportfrom` email address and either the `smtpserver` or `sendmail` setting on the puppet master
* Create a `tagmail.conf` file at the location specified in the `tagmap` setting

More details are available at the [tagmail report reference](http://docs.puppetlabs.com/references/stable/report.html#tagmail). 

The `tagmail.conf` file is list of lines, each of which consists of a tag, a colon, and an email address. The tag portion of a line can also be a !negated tag, a list of tags, or the word "all," which does exactly what it sounds like.

    all: zach@puppetlabs.com
    webserver, !mailserver: httpadmins@domain.com

### `autosign.conf`

The `autosign.conf` file (located at `/etc/puppet/autosign.conf` by default, and configurable with the `autosign` setting) is a list of certnames or certname globs (one per line) whose certificate requests will automatically be signed. 

    rebuilt.puppetlabs.lan
    *.magpie.puppetlabs.lan
    *.local

Note that certname globs do not function as normal globs: an asterisk can only represent one or more subdomains at the front of a certname that resembles a fully-qualified domain name. (That is, if your certnames don't look like FQDNs, you can't use `autosign.conf` to full effect.

As any host can provide any certname, autosigning should only be used with great care, and only in situations where you essentially trust any computer able to connect to the puppet master.

### `device.conf`

Puppet device, added in Puppet 2.7, configures network hardware using a catalog downloaded from the puppet master; in order to function, it requires that the relevant devices be configured in `/etc/puppet/device.conf` (configurable with the `deviceconfig` setting). 

`device.conf` is organized in INI-like blocks, with one block per device:

    [device certname]
        type <type>
        url <url>
    [router6.puppetlabs.lan]
        type cisco
        url ssh://admin:password@ef03c87a.local
