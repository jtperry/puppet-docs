---
layout: pe2experimental
title: "PE 2.0 » Installing » Basic Installation"
---


Basic Installation
======

Starting the Installer
-----

To install PE, unarchive the tarball, navigate to the resulting directory, and run the `./puppet-enterprise-installer` script in your shell with root privileges. 

This will start the installer in interactive mode and guide you through customizing your installation. After you've finished, it will save your answers in a file called `answers.lastrun`, install the selected software, and configure and enable all of the necessary services. 

The installer can also be run non-interactively; [see the next chapter][automated] for details.

[automated]: ./install_automated.html

Customizing Your Installation
-----

The PE installer configures Puppet by asking a series of questions. Most questions have a default answer (displayed in brackets), which you can accept by pressing enter without typing a replacement. For questions with a yes or no answer, the default answer is capitalized (e.g. "`[y/N]`").

### Roles

First, the installer will ask which of PE's <!-- TODO replace this link -->[roles](./overview.html#roles) (puppet master, console, cloud provisioner, and puppet agent) to install. The roles you apply will determine which other questions the installer will ask. 

If you choose the puppet master or console roles, the puppet agent role will be installed automatically.

### Puppet Master Options

#### Certname

The certname is the puppet master's site-unique identifier. This defaults to its fully-qualified domain name, which is usually the best choice, but any arbitrary string can be used.

#### Valid DNS Names

The master's certificate contains a static list of valid DNS names, and agents won't trust the master if they contact it at an unlisted address. You should make sure that this list contains the DNS name or alias you'll be configuring your agents to contact.

The valid DNS names setting should be a comma-separated list of hostnames, and will default to the server's short hostname, its fully-qualified domain name, `puppet`, and `puppet.<your domain>`.

#### Location of the Console Server

If you are splitting the puppet master and console roles across different machines, the installer will ask you for the hostname and port of the console server.

### Console Options

The console is usually run on the same server as the puppet master, but can also be installed on a separate machine. **If you choose to do so, you will need to do some [additional configuration][consoleconfig] after installing.**

#### Port

You must choose a port on which to serve the console's web interface. If you aren't already serving web content from this machine, it will default to **port 443,** so you can reach it at `https://yourconsoleserver` without specifying a port.

If the installer detects another web server on the node, it will suggest the first open port at or above 3000.

#### User Name and Password

As the console's web interface is a major point of control for your infrastructure, access is restricted with a user name and password. Additional users and passwords can be added later with Apache's standard authentication tools. 

#### Inventory Certname and DNS Names (Optional)

If you are splitting the master and the console roles, the console will maintain an inventory service to collect facts from the puppet master. Like the master, the inventory service needs a unique certname and a list of valid DNS names. 

**Note:** you will need to [exchange certificates between the console and the puppet master after installation][consoleconfig] if you are splitting the roles.

#### Database

The console needs a MySQL database and user in order to operate. If a MySQL server isn't already present on this server, the installer can automatically configure everything the console needs; just confirm that you want to install a new database server, and configure the following settings:

* A password for MySQL's root user
* A name for the console's database
* A MySQL user name for the console
* A password for the console's user

If you don't install a new database server, you must manually create a database and MySQL user for the console and configure the settings above with the correct information.

You can create the necessary MySQL resources in a secondary shell session while the installer is waiting for input. The SQL commands you need will resemble the following:

    CREATE DATABASE console CHARACTER SET utf8;
    CREATE USER 'console'@'localhost' IDENTIFIED BY 'password';
    GRANT ALL PRIVILEGES ON console.* TO 'console'@'localhost';

Consult the MySQL documentation for more info.

A local database is best, but you can also create a database on a remote server. If you do, you'll need to answer `Y` when asked if your MySQL server is remote, and provide:

* The database server's hostname
* The database server's port (default: 3306)


### Puppet Agent Options

#### Certname

The certname is the agent node's site-unique identifier.

This defaults to the node's fully-qualified domain name, but any arbitrary string can be used. If hostnames change frequently at your site or are otherwise unreliable, you may wish to use UUIDs or hashed firmware attributes for your agent certnames.

#### Puppet Master Hostname

Agent nodes need the hostname of a puppet master server. This must be one of the valid DNS names you chose when installing the puppet master.

This setting defaults to `puppet`.


### Vendor Packages

Puppet Enterprise may need some extra system software from your OS vendor's package repositories. If any of this software isn't yet installed, the installer will list the packages it needs and offer to automatically install them. If you decline, it will exit so you can install the necessary packages manually before installing.

**A note about Java and MySQL under Enterprise Linux variants:** Java and MySQL packages provided by Oracle can satisfy the puppet master and console roles' Java and MySQL dependencies, but the installer can't make that decision automatically and will default to using the OS's packages. If you wish to use Oracle's packages instead of the OS's, you must first use RPM to manually install the `pe-virtual-java` and/or `pe-virtual-mysql` packages included with Puppet Enterprise: 

    # sudo rpm -ivh packages/pe-virtual-java-1.0-1.pe.el5.noarch.rpm

Find these in the installer's `packages/` directory. Note that these packages may have additional ramifications if you later install other software that depends on OS MySQL or Java packages. 


### Convenience Links

PE installs its binaries in `/opt/puppet/bin` and `/opt/puppet/sbin`, which aren't included in your default `$PATH`. If you want to make the Puppet tools more visible to all users, the installer can make symlinks in `/usr/local/bin` for the `facter, puppet, puppet-module, pe-man`, and `mco` binaries. 


Finishing Up
-----

<!-- todo FUTURE: Add links to docs on puppet security model or getting started info -->

### Signing Agent Certificates

Before nodes with the puppet agent role can fetch configurations, an administrator has to sign their certificate requests. This helps prevent unauthorized nodes from reading arbitrary configuration information from your site. 

on a new node, the installer will automatically submit a certificate request to the puppet master. Before the agent will be able to receive any configurations, a user will have to sign the certificate from the puppet master. To view the list of pending certificate signing requests, run:

    puppet cert list

To sign one of the pending requests, run:

    puppet cert sign <name>


[consoleconfig]: #exchanging-consolepuppet-master-certificates

### Exchanging Console/Puppet Master Certificates

If you chose to split the console and puppet master roles, you need to run the following extra commands after installing them both:
<!-- TODO: This busted when I tried. Find out why. -->

**On the console server:**

    # cd /opt/puppet/share/puppet-dashboard
    # sudo env PATH=/opt/puppet/sbin:/opt/puppet/bin:$PATH RAILS_ENV=production \
    rake cert:request

**On the puppet master server:**

    # sudo puppet cert sign pe-internal-dashboard
    # sudo puppet cert sign <inventory service's certname>

**On the Dashboard server (in the same shell as before, with the modified PATH):**

    # rake RAILS_ENV=production cert:retrieve
    # receive_signed_cert.rb <inventory service's certname> <puppet master's hostname>
    # service pe-httpd start

[noninteractive]: #non-interactive-installation

Advanced Installation
----

### Installer Options

The Puppet Enterprise installer will accept the following command-line flags:

* `-a ANSWER_FILE` -- Read answers from file and quit with error if an answer is missing.
* `-A ANSWER_FILE` -- Read answers from file and prompt for input if an answer is missing.
* `-D` -- Display debugging information.
* `-h` -- Display a brief help message.
* `-l LOG_FILE` -- Log commands and results to file.
* `-n` -- Run in 'noop' mode; show commands that would have been run during installation without running them.
* `-s ANSWER_FILE` -- Save answers to file and quit without installing.
