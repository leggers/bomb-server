Bomb Server Setup
===========

## [Ansible](http://docs.ansible.com/index.html) playbooks that set up a bomb-ass server for deploying Ruby on Rails apps using:

* A handy-dandy [Vagrant](vagrantup.com) box for testing
* An unprivileged "deploy" user (cool)
* [RVM](rvm.io) (really?)
* [Phusion Passenger aka mod_rails](phusionpassenger.com/) (are you serious?!)

There is nothing particularly notable about this server setup for production Rails apps, however doing it with Ansible was not something I found while doing some serious googling doing my development of this playbook.
The only setup required on the server is Ubuntu 13.10 with a user to which one can SSH that has password-less sudo privileges.
I wrote this to set up [Bombsheller's](http://shop.bombsheller.com/) VMs.

# Overview and Ansible Background

First, a short overview of how playbooks work (you can read more about them [in their documentation](http://docs.ansible.com/playbooks.html)):
An Ansible playbook is a series of `tasks` organized into `roles` to be executed over SSH using YAML syntax against a set of `hosts` organized into `groups`.

Wait, what?

In Ansible, you specify lists of `hosts`, or servers (like EC2 instances, or the server you have spinning in your basement), and organize them into `groups` (by, say, server purpose or geography). You can then choose with a high degree of precision exactly **which** servers you are configuring.

You then tell Ansible how to configure the various parts of your IT setup using `roles`, or a server's functional purpose (say, database, or webserver). This allows you to control with a high degree of precision exactly **what** is installed on each server.

Tasks are organized into a certain file structure to denote and delimit `roles`.
A typical task is `apt: pkg=curl`.
As you can imagine, this uses [APT](http://en.wikipedia.org/wiki/Advanced_Packaging_Tool) to install the `curl` package.
Generic task syntax is `<module>: <options>` (with other other options possibly being on the next lines).
The great part about Ansible is that if `curl` is already installed, it will do nothing!
Another way to say that is Ansible strives to be *idemopotent*, meaning running the same playbook over and over again should yield the same results.
From a sysadmin perspective, this is incredible because it acts as something that *ensures* your server is properly setup in a human-readable way, instead of relying on the shell scripts and saavy.
[Here's a list of all modules](http://docs.ansible.com/list_of_all_modules.html).
It is often handy to keep that page open while dealing with playbooks.

Ansible also has the capacity to copy over files, even inserting arbitrary information (!) using [templates](http://docs.ansible.com/template_module.html) and [variables](http://docs.ansible.com/playbooks_variables.html), so your site- and machine-specific configuration can be automated as well. Pretty awesome.

# This Playbook

Anyway, enough about Ansible, onto what happens in our playbooks.
How to use this playbook: `ansible-playbook site.yml -i hosts -l staging`.
In English, this says "Ansible, run the playbook in `site.yml` against the hosts listed in the file `hosts` in the group `staging`."

### Order of playbook operations

(You can follow along by opening `site.yml` and following the chain of file calls to see how the flow works) is as follows:

1) The first few lines of the `site.yml` are configuration.
Ansible will run against all hosts and use sudo unless told otherwise in the playbook or on the command line.
Ansible then looks in the file indicated to get some values for variables to be used later in the playbook.
For every listed item under `roles:`, Ansible looks in the `roles` folder for a folder of the same name.
It then executes the tasks listed in the `tasks` folder, starting with the file `main.yml`.

2) All common tasks are executed.
Common means that it doesn't matter what type of server it is (database, webserver, whatever), these configurations should be ensured.
Right now, the only common task is to set server locale.
You'll see the `notice` statment. This calls the task by the same name in the `handlers` folder.

3) The webserver is setup.
This is by far the most complicated part of the playbook, so its plays are broken into into various files to separate concerns.
The major concerns are: setting up the `deploy` user account, getting a ruby installation (for which I use RVM in the `deploy` account), and getting a web server set up (for which I use Apache and Passenger).
You can read through those playbooks to see what they do.
The `authorized_key` directive copies an SSH key *from the local machine*.
The `file` and `template` directives look in the `file` and `template` directories of that role, respectively, for the files or templates listed.

4) The database is setup.

You will notice, especially for setting up RVM, Ruby, and Passenger, that I use the `shell` module quite a bit, which seems to defeat the purpose of using idempotent modules because it simply executes whatever command is passed to it.
However, I strive to make them more robust (and not repeat themselves unnecessarilty) by using the `creates` option, which specifies a file or folder that should be created by the command.
It is that trick which really differentiates that setup from a shell script.
Yes, you could say that checking for folders' existence in the shell script is the same, but someone who doesn't know bashscript can read this and understand what it means.
