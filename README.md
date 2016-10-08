# update-blocklist

A script to update an iptables-based firewall to block inbound traffic using a static list maintained locally and/or one or more dynamic lists. 


## Introduction

This script updates rules used by an iptables-based firewall to block inbound traffic.  You can use it to filter incoming traffic based on a static list maintained locally and/or one or more dynamic lists available on the web, such as [DShield.org's Highly Predictive Blacklist](http://www.dshield.org/hpbinfo.html) and [The Spamhaus Don't Route Or Peer List](http://www.spamhaus.org/drop/).  The static list allows you to tailor rules to individual machines while dynamic lists help you stay up-to-date with current threats. 

Each time you run it, **update-blocklist** repopulates the firewall blocklist (a special user-defined chain through which inbound traffic passes).  If a dynamic list is retrieved and, optionally, verified successfully, a copy will be saved for review and re-use.  It will be re-used in the event the option `--no-gets` is given or a current copy can not be retrieved or verified. 

**update-blocklist** is written in Perl.  It should work on any Linux system with Perl 5 and iptables.  In addition, it can be configured to use [GNU Privacy Guard](http://www.gnupg.org/) (aka GnuPG), `md5sum`, or `sha256sum` to verify the contents of a dynamic blocklist so you may need to have working that as well.  Finally, it requires the following Perl modules:

* `Carp`
* `File::Copy`
* `File::Temp`
* `Getopt::Long`
* `IPC::Run`
* `LWP::Debug`
* `LWP::UserAgent`
* `Net::IP`

If your system does not have these modules installed already, visit [CPAN](http://search.cpan.org/) for help.  Note that `IPC::Run`, `LWP::Debug`, `LWP::UserAgent`, and `Net::IP` are not included with the default Perl distribution so you may need to install them yourself; you can find `LWP::Debug` and `LWP::UserAgent` as part of [the LWP library](http://search.cpan.org/dist/libwww-perl/). 


## Configuring Dynamic Blocklists

For each dynamic blocklist you wish to support with this script, you must have an element in the array `@bl_dynamic`.  This element, known as a _hash_ in Perl, should itself contain several other elements -- key / value pairs -- as follows:

| Key | Value |
| --- | ----- |
| `label` | A string used as the label for the blocklist. Optional. |
| `url` | A string used as the URL from which the blocklist is to be retrieved. Any protocol scheme supported by LWP is valid; eg, http, https, ftp, etc. **Required**. |
| `verify` | A hash indicating that the contents of the blocklist should be verified. Optional. |
| `method` | A string within the `verify` hash indicating the method used to verify the blocklist contents -- either `gpg`, `md5`, or `sha256`. Required if the blocklist is to be verified. |
| `url` | A string within the `verify` hash indicating the url from which the GPG signature or SHA256 / MD5 checksum can be retrieved for verifying the blocklist. Required if the blocklist is to be verified. |
| `uid` | A string within the `verify` hash indicating the uid of the GPG key used to sign the blocklist. Optional if the blocklist is to be verified using GnuPG. |
| `parse` | An anonymous subroutine that accepts a line from the blocklist and returns a rule specification, if possible. **Required**. |
| `copy` | A string used as the file name in which a copy of the blocklist will be stored. **Required**. |

Consult the [sample configuration file](update-blocklist.conf) to see how these various elements are used. 


## Installation

* Retrieve [the script](update-blocklist) and save it locally.
* Retrieve [the sample configuration file](update-blocklist.conf) and save it locally; eg, as `/usr/local/etc/update-blocklist.conf`.
* Verify ownership and permissions on the script and configuration file - they should be owned by root.
* Edit the script / configuration file and set `$ENV{PATH}` and `$proxy`according to your environment.  You may also wish to adjust the location
of the perl interpreter in the first line of the script as well as `@bl_dynamic`, `$bl_static`, `$ipt_chain`, `$ipt_default_target`, and `$optimize` to suit your tastes. 
* Create empty files specified by `@bl_dynamic` and `$bl_static`.  Make sure that they are owned by root and that untrusted users can not modify them!
* If necessary, add any GnuPG / PGP keys to root's keyring so you can verify the contents of any dynamic blocklists as desired.
* Use `iptables` to create a user-defined chain named `BLOCKLIST` and route inbound traffic through it.  For example, suppose you have an input chain named `INETIN` that accepts traffic from the Internet and you want to apply the blocklist to that traffic.  In that case, you would run the following commands to add a blocklist to your firewall:

  * `iptables -N BLOCKLIST`
  * `iptables -A BLOCKLIST -j RETURN`
  * `iptables -I INETIN 1 -j BLOCKLIST`

 The first command creates the blocklist; the second ensures that unmatched packets (which initially means all packets) continue to be processed in the chain that called `BLOCKLIST`, and the third inserts a rule in the chain `INETIN` to route all traffic through the blocklist before doing anything else. When you're satisfied things are working, be sure to save the rules using `iptables-save` so everything will work when you restart iptables.


## Use

If you are using any dynamic blocklists, you will probably want to call **update-blocklist** once or twice a day using cron to keep the blocklist up-to-date.  In addition, you may wish to call it from the startup script for your firewall.  And of course, you can call it from the commandline whenever you make changes to the static list.

The static list is a simple text file, as in the sample blocklist.static.  Blank and comment lines are ignored; others should have one or more whitespace-delimited fields as follows:

* a source specification (eg, an IP address or network address range in CIDR format). NB: this is required.
* an optional jump target (eg, DROP, REJECT, etc). NB: if no jump target is specified, a default one will be used.
* an optional string to be appended as-is to the iptables command.

See the iptables man page for more information on these fields.

There are several commandline arguments you can use to override variables defined in the script itself:

| Option | Meaning |
| ------ | ------- |
| `-d, --debug` | Display debugging messages while running, including the iptables commands that would be run to repopulate the blocklist. Note, however, that no changes are actually made. |
| `-n, --no-gets` | Update the blocklist without attempting to retrieve new copies of dynamic blocklists. |
| `-o, --optimize` | Optimize the rules by checking for overlap. NB: you can negate this option; eg, '--no-optimize'. |
| `-s, --static-only` | Update the blocklist using only the static list. Note: this causes any blocks from dynamic lists to be removed. |

Notes:
* The `--no-gets` option is useful if your firewall starts before your network connection (for example, with PPPoE and a DSL connection).

Examples:

| Invocation | Meaning |
| ---------- | ------- |
| `update-blocklist` | updates firewall blocklist.|
| `update-blocklist -d` | displays commands used to update firewall blocklist without actually doing so. |
| `update-blocklist -s` | updates firewall blocklist using only static rules. NB: this causes any dynamic rules to be removed. |
| `update-blocklist -n` | updates firewall blocklist but doesn't retrieve new copies of dynamic lists. |


## Known Bugs and Caveats

Currently, I am not aware of any bugs in this script.

Rule optimization can slow things down significantly.  Disable it if you don't mind the inefficiencies that entails or prefer to handle it
manually.

If you encounter a problem using this script, I encourage you to enable debug mode (eg, add `-d` to your commandline) and examine the output it produces before contacting me.  Often, this will enable you to resolve the problem yourself


## Copyright and License

Copyright (c) 2003-2016, George A.  Theall.
All rights reserved.

This script is free software; you can redistribute it and/or modify it under the same terms as Perl itself.
