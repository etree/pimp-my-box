# ansible-carve

Automated install of carve-core etc. via
[Ansible](http://docs.ansible.com/).

**Contents**

  * [Introduction](#introduction)
  * [How to Use This?](#how-to-use-this)
      * [Installing Ansible](#installing-ansible)
      * [Checking Out the Code](#checking-out-the-code)
      * [Setting Up Your Environment](#setting-up-your-environment)
      * [Running the Playbook](#running-the-playbook)
      * [Starting rTorrent](#starting-rtorrent)
      * [Activating Firewall Rules](#activating-firewall-rules)
      * [Changing Configuration Defaults](#changing-configuration-defaults)
      * [Enabling Optional Applications](#enabling-optional-applications)
      * [Installing and Updating ruTorrent](#installing-and-updating-rutorrent)
  * [Advanced Configuration](#advanced-configuration)
      * [Using the System Python Interpreter](#using-the-system-python-interpreter)
      * [Using the bash Completion Handler](#using-the-bash-completion-handler)
      * [Extending the Nginx Site](#extending-the-nginx-site)
  * [Trouble-Shooting](#trouble-shooting)
      * [SSH Error: Host key verification failed](#ssh-error-host-key-verification-failed)
  * [Implementation Details](#implementation-details)
      * [Secure Communications](#secure-communications)
  * [References](#references)


## Introduction

Ansible is a tool that allows you to install a complete setup remotely on one or any number of *target hosts*,
from the comfort of your own workstation.
The setup is described in so called *playbooks*,
before executing them you just have to add a few values like the name of your target host.

The playbooks contained in this repository install the following components:

* Security hardening of your server.

Optionally:

* More things

Each includes a default configuration, so you end up with a fully working system.

The Ansible playbooks and related commands have been tested on Debian Jessie, Ubuntu Trusty, and Ubuntu Lucid
They should work on other platforms too, especially when they're Debian derivatives, but you might have to make some modifications.
Files are mostly installed into the user account `bts` and only a few global configuration files are affected. If you run this against a host
with an existing installation, make sure that there are no conflicts.

## How to Use This?

Here's the steps you need to follow to get a working installation on your target host.
Note that this cannot be an Ansible or Linux shell 101, so for details refer to
the usual sources
like [The Debian Administrator's Handbook](http://debian-handbook.info/browse/stable/),
[The Linux Command Line](http://linuxcommand.org/tlcl.php)
and [The Art of Command Line](https://github.com/jlevy/the-art-of-command-line#the-art-of-command-line),
and the [Ansible Documentation](http://docs.ansible.com/#ansible-documentation).


### Installing Ansible

Ansible has to be installed on the workstation from where you control your target hosts.
This can also be the target host itself, if you don't have a Linux or Mac OSX desktop at hand.
See the [Ansible Documentation](http://docs.ansible.com/intro_installation.html)
for how to install it using the package manager of your platform.
Make sure you get the right version that way, the playbooks are tested using Ansible *1.9.4*,
and Ansible 2 might not work (yet).

Another way to install Ansible is to put it into your home directory.
The following commands just require Python to be installed to your system,
and the installation is easy to get rid of (everything is contained within a single directory).

```sh
# just to make sure you have the packages you need
sudo apt-get install build-essential python-virtualenv python-dev

# install Ansible 1.9
base="$HOME/.local/venvs"
mkdir -p "$base"
/usr/bin/virtualenv "$base/ansible"
cd "$base/ansible"
bin/pip install "ansible==1.9.4"

# create a configuration file
if test '!' -f ~/.ansible.cfg; then
    curl -o ~/.ansible.cfg "https://raw.githubusercontent.com/ansible/ansible/stable-1.9/examples/ansible.cfg"
    ${EDITOR:-vi} ~/.ansible.cfg # uncomment and change line containing 'roles_path'
    # roles_path      = $HOME/.ansible/roles:/etc/ansible/roles
fi

# make Ansible commands available by default
test -d "$HOME/bin" || { mkdir -p "$HOME/bin"; exec -l $SHELL; }
ln -s "$PWD/bin"/ansible* "$HOME/bin"
cd; ansible --version
```

To get it running on Windows is also possible,
by [using CygWin](https://servercheck.in/blog/running-ansible-within-windows)
(untested, success stories welcome).


### Checking Out the Code

To work with the playbooks, you of course need a local copy.
Unsurprisingly, you also need ``git`` installed for this.

```sh
which git || sudo apt-get install git
mkdir ~/src; cd ~/src
git clone "https://github.com/BlackTieSkiRentals/ansible-carve.git"
cd "ansible-carve"
```


### Setting Up Your Environment

Now with Ansible installed and having a local working directory,
you next need to configure the target host
via a ``hosts`` file in your working directory (the so-called *inventory*).
The ``hosts-example`` file shows how this has to look like,
enter the name of your target instead of ``my-box.example.com``
(and anywhere else ``my-box`` appears further below).


```ini
[box]
my-box.example.com
```

You also need a file with the specifics of your box in ``host_vars/my-box/main.yml``,
the so-called host variables. There is an example in
[host_vars/rpi/main.yml](https://github.com/ansible-carve/blob/master/host_vars/rpi/main.yml)
which works with a default *Raspberry Pi* setup that comes with a password-less sudo account.
Normally you'd add `ansible_sudo_pass` in `host_vars/my-box/secrets.yml`, or else use
`-K` on the command line to prompt for the password.

Next, we check your setup and that Ansible is able to connect to the target and do its job there.
Make sure you have working SSH access based on a pubkey login first (see
[here](https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys)).
Otherwise, use `--ask-pass` in combination with a password login
– for any details with this, consult the Ansible documentation.

Then call the command as shown after the ``$``,
and it should print what OS you have installed on the target(s),
like shown in the example.

```sh
$ ansible all -i hosts -m setup -a "filter=*distribution*"
rpi | success >> {
    "ansible_facts": {
        "ansible_distribution": "Debian",
        "ansible_distribution_major_version": "7",
        "ansible_distribution_release": "wheezy",
        "ansible_distribution_version": "7.8"
    },
    "changed": false
}
```

If anything goes wrong, add ``-vvvv`` to the ``ansible`` command for more diagnostics,
and also check your `~/.ssh/config` and the Ansible connection settings in your `host_vars`.

Here is an example `~/.ssh/config` snippet that provides details on
how to connect to the ``rpi`` host:

```ini
Host rpi
    HostName 192.168.1.2
    User pi
    IdentityFile ~/.ssh/id_rsa
    IdentitiesOnly yes
    # The following is unsecure, for a Raspberry PI it allows easy card swapping...
    CheckHostIP no
    UserKnownHostsFile /dev/null
    StrictHostKeyChecking no
```

To give you an idea why this way to do things is way superior to the usual
*“call a bash script to set up things once and never update them again”*,
you can now also easily call commands over a fleet of machines, *in parallel*:

```sh
ansible all -i hosts -f 9 -a "sudo apt-get -qq update"
```


### Running the Playbook

To execute the playbook, call ``ansible-playbook -i hosts site.yml``.
If you added more than one host into the ``box`` group and want to only address one of them,
use ``ansible-playbook -i hosts -l ‹hostname› site.yml``.
Add (multiple) ``-v`` to get more detailed information on what each task does.

NOTE: the SSL certificate generation is not fully automatic yet, run the command shown in
the error message you'll get, as `root` in the `/etc/nginx/ssl` directory – once the
certificate is created, re-run the playbook and it should progress beyond that point.
Of course, you can also copy a certificate you got from other sources to the paths
`/etc/nginx/ssl/cert.key` and `/etc/nginx/ssl/cert.pem`.
See [this blog post](https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html)
if you want *excessive* detail on secure HTTPS setups.


### Activating Firewall Rules

If you want to set up firewall rules using the
[Uncomplicated Firewall](https://en.wikipedia.org/wiki/Uncomplicated_Firewall) (UFW) tool,
then call the playbook using this command:

```sh
# See above regarding adding the '-l' option to select a single host
ansible-playbook -i hosts site.yml -t ufw -e ufw=true
```

This will install the `ufw` package if missing, and set up all rules needed by apps installed
using this project. Note that activating the firewall is left as a manual task, since you can
make a remote server pretty much unusable when SSH connections get disabled by accident – only
a rescue mode or virtual console can help to avoid a full reinstall then, if you have no
physical access to the machine.

So to activate the firewall rules, use this in a `root` shell on the *target host*:

```sh
egrep 'ssh|22' /lib/ufw/user.rules
# Make sure the output contains
#   ### tuple ### limit tcp 22 0.0.0.0/0 any 0.0.0.0/0 in
# followed by 3 lines starting with '-A'.

ufw enable  # activate the firewall
ufw status verbose  # show all the settings
```


### Changing Configuration Defaults

Once created, the config files are only overwritten when you provide
`-e force_cfg=yes` on the Ansible command line!


### Enabling Optional Applications

To activate the optional applications, add these settings to your `host_vars`:

 * `foo_enabled: yes` for Foo.
 * `bar_enabled: yes` for Bar.

Then run the playbook again.


## Advanced Configuration

### Using the System Python Interpreter

By default, Python 2.7.10 is installed because that version handles SSL connections
according to current security standards; the version installed in your system often
does not. This has an impact on e.g. FlexGet's handling of ``https`` feeds.

If you want to use the system's Python interpreter, add these variables to your host vars:

```ini
pyenv_enabled: false
python_bin: /usr/bin/python2
venv_bin: /usr/bin/virtualenv
```

### Extending the Nginx Site

The main Nginx server configuration includes any `/etc/nginx/conf.d/carve-*.include`
files, so you can add your own locations in addition to the default `/carve` one.
The main configuration file is located at `/etc/nginx/sites-available/carve`.

Use a `/etc/nginx/conf.d/upstream-*.conf` file in case you need to add your own `upstream` definitions.


## Trouble-Shooting

### SSH Error: Host key verification failed

If you get this error, one easy way out is to first enter the following command
and then repeat your failing Ansible command:

```sh
export ANSIBLE_HOST_KEY_CHECKING=False
```


## Implementation Details

## References

### Server Hardening

 * [Secure Secure Shell](https://stribika.github.io/2015/01/04/secure-secure-shell.html)
 * https://github.com/sfromm/ansible-rkhunter
