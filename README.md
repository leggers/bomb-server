Bomb Server Setup
===========

## [Ansible](http://docs.ansible.com/index.html) playbooks that set up a bomb-ass server for deploying Ruby on Rails apps using:

* A handy-dandy [Vagrant](vagrantup.com) box for testing
* An unprivileged "deploy" user to run the app (cool)
* [RVM](rvm.io) (really?)
* [Phusion Passenger aka mod_rails](phusionpassenger.com/) (are you serious?!)

There is nothing particularly notable about this server setup for production Rails apps, however doing it with Ansible was not something I found during my serious googling while developing this playbook.
The only setup required on the server is Ubuntu 13.10 with a user to which one can SSH that has password-less sudo privileges.
I wrote this to set up [Bombsheller's](http://shop.bombsheller.com/) VMs.

# Installation and Use

## Installation

To install Ansible, use the [Ansible installation docs](http://docs.ansible.com/intro_installation.html).
I recommend installing using pip because it was easy.
During installation I encountered a strange error with a mangled printline.
I eventually found out that this was due to Windows carriage returns in the code.
To fix this I used a [dos2unix](http://linuxcommand.org/man_pages/dos2unix1.html) on the files that threw the errors.
Alternatively, you could open the offending files with `vi` and follow [this tip](http://stackoverflow.com/questions/82726/convert-dos-line-endings-to-linux-line-endings-in-vim).

Clone this repository.

## Use

There are two playbooks, as explained below, with distinct uses.
One playbook sets up VMs to host and serve the site.
It installs all application dependencies and puts in place database secrets and other configuration files.
To use it, you need to have SSH access to a user with password-less sudo privileges (to install everything) and the Ansible Vault password (for database secrets).
The command is `ansible-playbook site.yml -i hosts -l <group_name>` where `<group_name>` is replaced by `staging` or `production`, for example.
You will also have to supply the Ansible Vault password, which Ansible will prompt you for upon running that comand.

The other playbook acts as a data synchronizing script.
It `mysqldump`s the `production` database, `rsync`s it to the local machine, rsync's the store's photos locally, then syncs them to the `staging` and `develop` environments.
To use it, you need to have SSH access to all three boxes's `deploy` user (or just `production`'s if you're only pulling to local machine).
The command is `ansible-playbook -i hosts shop-data-sync.yml`.
Because it moves data between hosts, you do not want to specify a specific group.

# Overview and Ansible Background

First, a short overview of how playbooks work (you can read more about them [in their documentation](http://docs.ansible.com/playbooks.html)):
An Ansible playbook is a series of server configuration `tasks` organized into `roles` to be executed over SSH using YAML syntax against a set of `hosts` organized into `groups`.

Wait, what?

In Ansible, you specify lists of `hosts`, or servers (like EC2 instances, or the server you have spinning in your basement), and organize them into `groups` (by, say, server purpose or geography). You can then choose with a high degree of precision exactly **which** servers you are configuring.

You then tell Ansible how to configure the various parts of your IT setup using lists of `tasks` (like installing packages or copying config files), and organize them into `roles` (say, database, or webserver). This allows you to control with a high degree of precision exactly **what** commands are executed on each server.

Tasks are organized into a certain file structure to denote and delimit `roles`.
A typical task is `apt: pkg=curl`.
As you can imagine, this uses [APT](http://en.wikipedia.org/wiki/Advanced_Packaging_Tool) to install the `curl` package.
Generic task syntax is `<module>: <options>` (with other other options possibly being on the next lines).
A `playbook` consists of a list of `tasks` or list of files to be executed.
They are run from top to bottom.

The great part about Ansible is that if `curl` is already installed, it will do nothing!
Another way to say that is Ansible strives to be **idemopotent**, meaning running the same playbook over and over again should yield the same results (i.e. a configured server will stay properly configured).
From a sysadmin perspective, this is incredible because it acts as something that **ensures** your server is properly set up in a human-readable way (YAML), instead of relying on the shell scripts and saavy.
[Here's a list of all ansible modules](http://docs.ansible.com/list_of_all_modules.html).
It is often handy to keep that page open while dealing with playbooks.

Ansible also has the ability to copy over files, even inserting arbitrary information (!) using [templates](http://docs.ansible.com/template_module.html) and [variables](http://docs.ansible.com/playbooks_variables.html), so your site- and machine-specific configuration can be automated as well. Pretty awesome.

# These Playbooks

Anyway, enough about Ansible, onto what happens in our playbooks.
There are two playbooks which encapsulate distinct tasks:
1) `site.yml` sets up a VM to serve a Ruby on Rails app using mod_rails and mySQL;
2) `shop-data-sync.yml` is used to sync the database and product image assets from the production server to Bombsheller's staging and development environments.
How to use the playbooks:
1) `ansible-playbook site.yml -i hosts -l staging`.
In English, this says "Ansible, run the playbook in `site.yml` against the hosts listed in the file `hosts` in the group `staging`."
2) `ansible-playbook -i hosts shop-data-sync.yml`
This says to run the `shop-data-sync.yml` playbook against all hosts in the hosts file.
If you look in the playbook you will see how it handles switching between the produciton and non-production hosts.

## Order of playbook operations for `site.yml`

Order of operations (you can follow along by opening `site.yml` and following the chain of execution to see how the flow works) is as follows:
(Note: it might be easiest to start by reading the common and database playbooks before trying to tackle the webserver ones.)

1) The first few lines of the `site.yml` are configuration.
Ansible will run against all hosts and use sudo unless told otherwise in the playbook or on the command line.
Ansible then looks in the files indicated to get some values for variables to be used later in the playbook.
For every listed item under `roles:`, Ansible looks in the `roles` folder for a folder of the same name.
It then executes the tasks listed in the `tasks` folder, starting with the file `main.yml`.

2) All tasks are executed for the `common` role.
Common means that it doesn't matter what type of server it is (database, webserver, whatever); these configurations should be ensured.
Right now, the only common task is to set server locale.
You'll see the `notice` statment. This calls the task by the same name in the `handlers` folder.

3) The webserver is set up.
This is by far the most complicated part of the playbook, so its plays are broken into into various files to separate concerns.
The major concerns are: setting up the `deploy` user account, getting a ruby installation (for which I use RVM in the `deploy` account), and getting a web server set up (for which I use Apache and Passenger).
You can read through those playbooks to see what they do.
The `authorized_key` directive copies an SSH key *from the local machine*.
The `file` and `template` directives look in the `file` and `template` directories of that role, respectively, for the files or templates listed.
There is a bit of webscraping going on the with `curl` command to ensure the latest version of mod_rails is installed.

4) The database is set up.
This is a standard MySQL setup, installing requisite packages, setting up user accounts and ensuring databases are created.
If you made it through the webserver playbooks this should be a piece of cake.

### Notes

You will notice, especially for setting up RVM, Ruby, and Passenger, that I use the `shell` module quite a bit, which seems to defeat the purpose of using idempotent modules because it simply executes whatever command is passed to it.
However, I strive to make them more robust (and not repeat themselves unnecessarilty) by using the `creates` option, which specifies a file or folder that should be created by the command.
It is that trick which really differentiates that setup from a shell script.
Yes, you could say that checking for folders' existence in the shell script is the same, but someone who doesn't know bashscript can read this and understand what it means.

## Order of playbook operations for `shop-data-sync.yml`

This file has far fewer things to do, so it made more sense to me to treat it as more of a script than a constellation of configuration files.

1) On the production server, this file:
    * performs a mysqldump of the current production data, timestamped by today's date
    * pulls that database to a local directory using the `synchronize` module (which is just a wrapper around the `rsync` command line utility)
    * pulls the product images to the local machine in the same manner as the database dump

2) On the non-production servers, as the unprivileged `deploy` user:
    * copies the database dump to the VM
    * imports the mysqldump file to update the database with the most current data
    * syncs production images to remote

3) On the non-production servers, as a privileged user (`leggers`, in this case):
    * symlinks the site configuration file, enabling it according to apache, recording the result
    * if the file had previously not been symlinked (i.e. site is enabled for the first time), restarts and reloads apache so it will begin serving the site