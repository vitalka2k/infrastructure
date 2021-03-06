Adblock Plus infrastructure
===========================

The Adblock Plus infrastructure uses [Puppet](http://puppetlabs.com/)
to set up servers, and to have a realistic development environment.

Our Puppet manifests are only tested with Ubuntu 12.04 right now.

Private files
-------------

Some parts of our infrastructure are, obviously, confidential. We have
htpasswd files, SSH keys and SSL certificates that we need to be
careful with.

That's why _modules/private_ is missing, and needs to be placed there
manually. We provide stub versions of all those files in
_modules/private-stub_, so just linking or copying that to
_modules/private_ will make everything work locally.

Development environment
-----------------------

As with our other projects, all changes to our infrastructure should
be made in a local development environment, and reviewed before
deployment. Thanks to Puppet, we can easily set up local VMs that
mirror our production environment.

The most convenient way to do this is to use Vagrant, as described
below.

### Requirements

* [VirtualBox](https://www.virtualbox.org/)
* [Vagrant](http://vagrantup.com/)
* _modules/private_ exists (see above)

### Start a VM

For each production server, we have a Vagrant VM with the same host
name.

To start the _filter1_ VM:

	vagrant up filter1

After you've made changes to Puppet manifests, you can update it like this:

	vagrant provision filter1

You can omit the VM name if you want to boot or provision all
VMs. This might take a while and eat quite a bit of RAM though.

### SSH to the server

You can use vagrant to connect as the vagrant user:

	vagrant ssh server5

If you want to test "real" SSH access you can use the test user account defined
in _private-stub_:

	ssh -i modules/private/files/id_rsa test@10.8.0.100

The default password for this user (required for the _sudo_ command) is "test".

Adding a server
---------------

To set up a new server, you should first add it to the development
environment and test the setup, then set up a corresponding production
server.

### Development environment

1. Add entries in _Vagrantfile_ and _manifests/vagrant.pp_

2. Add the host name to one of the manifests imported by
_manifests/nodes.pp_

3. Make sure the server uses the _nagios::client_ class and add a
_nagios\_host_ to _manifests/monitoringserver.pp_

### Production environment

1. Install Ubuntu Server 12.04 LTS
2. Perform an update and install Puppet

	apt-get -y update && apt-get -y upgrade && apt-get -y install puppet

3. Enable pluginsync (Add the following to the _main_ section in
   _/etc/puppet/puppet.conf_)

	pluginsync=true

4. Configure the master address (Add the following to the bottom of
	_/etc/puppet/puppet.conf_)

	[agent]
	server = puppetmaster.adblockplus.org

Now you can either set it up as a pure agent or as a master. The
master provides the configuration, agents fetch it from the master and
apply it locally. The master is also an agent, fetching configuration
from itself.

#### Puppet agent

1. Attempt an initial provisioning, this will fail

	puppet agent --test

2. On the master: List the certificates to get the name of the new
   agent's certificate

	puppet cert list

3. Still on the master: Sign the certificate, e.g. for serverx:

	puppet cert sign serverx

4. Back on the agent: Attempt another provisioning, it should work now

	puppet agent --test

#### Puppet master

1. Configure the certificate name (Add the following to the _master_
   section in _/etc/puppet/puppet.conf_)

	certname = puppetmaster.adblockplus.org

2. Install the required packages

	apt-get install puppetmaster mercurial

3. Clone the infrastructure repository

	hg clone ssh://hg@adblockplus.org/infrastructure /etc/puppet/infrastructure
	rmdir /etc/puppet/{modules,manifests,templates}
	ln -s /etc/puppet/infrastructure/manifests /etc/puppet/manifests
	ln -s /etc/puppet/infrastructure/modules /etc/puppet/modules

4. Make sure to put the private files in place (see above)

5. Provision the master itself

	puppet agent --test

Updating a production server
----------------------------

Puppet agent has to be rerun on the servers whenever their configuration is
changed. The _kick.py_ script automates and simplifies that task, e.g. the
following will provision all servers (requires Puppet and PyYAML):

	kick.py -u serveradmin all

Here _serveradmin_ is your user account on the servers, it will be used to
run Puppet on the servers via SSH (sudo privilege required). You can list any
host groups defined in _manifests/monitoringserver.pp_ or individual servers.
You can also use _-v_ flag to see verbose Puppet output or _-t_ flag to do a
dry run without changing anything.

Monitoring
----------

Monitoring is fully functional in any environment, including development.
Here, after bootstrapping the `server4` box, one can access the Nagios GUI
from the host machine via <https://nagiosadmin:nagiosadmin@10.8.0.99/>.

The monitoring service of our production environment, however, is accessible
via <https://monitoring.adblockplus.org/>.
Add yourself to _files/nagios-htpasswd_ in the _private_ module used on the
server, or have someone add you if you don't have access.


