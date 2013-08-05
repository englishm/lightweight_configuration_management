# Lightweight Configuration Management 

(for Fun and for Profit)

# Background

<aside class="notes">
- let's start with some background
</aside>

# Puppet

<aside class="notes">
- started using CM ~1.5 years ago w/ Puppet on the World Bank project
- used Puppet to rebuild internal services when Deb5 went EOL
- started using Puppet on other client projects, moved away from using a Puppet master
</aside>

# Vagrant

<aside class="notes">
- Around that time..
- Started adding Vagrant to the mix to validate our configurations on throwaway nodes before provisioning production
- (I really like Vagrant. FYI: The author will be speaking at SoftwareGR in October)
</aside>

# Chef

<aside class="notes">
- we thought it would be good to know both, so we
- started experimenting with Chef, found it slightly faster, more community ties to RoRs Dev community
</aside>

# Capistrano

<aside class="notes">
- got tired of manually SCPing configs over and running puppet standalone or chef solo by hand each time
- started to use Capistrano to automate this
- (could have been fabric or whatever, but we already used Capistrano for deployment, etc.)
- we still used both Puppet and Chef regularly so we didn't want to commit to similar tools in either space
  (e.g. knife)
- we wanted something independent of particular provisioning tools
</aside>


# Provisioner Independent

![](images/swedish_chef.jpg)

<aside class="notes">
- Our Capistrano-based approach lets us use Puppet and Chef equally well.
- (though we're mostly using Chef these days, I have a fondness for the Puppet DSL and would happily pick it up again if a client project req. it)
</aside>

# Our Pattern, A Walkthrough

<aside class="notes">
- The current pattern…
	- includes:
		- Vagrantfile
		- Capistrano config
		- cookbooks/recipes (or modules/manifests)
	- operationally:
		- (walk through deploy.rb)
		- (walk through node JSON/runlist and recipes)
		- (touch on Vagrantfile and ssh config magicks, if time)
	- library mgmt:
		- bundler for gems
		- berkshelf for cookbooks
</aside>

# Demo

```bash
	git clone https://github.com/kuleszaj/chef-an-introduction
	cd chef-an-introduction
	bundle install
	bundle exec cap berks
	bundle exec cap vrails4 vg:up
	bundle exec cap vrails4 bootstrap
	bundle exec cap vrails4 chef
```

<aside class="notes">
- right now we're copying from a base when starting new projects
- we had an internal gem w/ a templatized version with options for puppet or chef, but it's out of date (anyone interested in contributing to that?)
- this expects ruby >=1.9 (we're using rbenv these days)
</aside>

# Gemfile

```ruby
	# -*- mode: ruby -*-
	# # vi: set ft=ruby :

	source "http://rubygems.org"

	# Required for capistrano
	gem "capistrano"

	# Not required, but may be useful for debugging
	#gem "pry"

	# Required/Useful for Chef
	gem "chef","11.4.4"
	gem "berkshelf"
	gem "foodcritic"
```

<aside class="notes">
</aside>

# Berksfile

```ruby
	# -*- mode: ruby -*-
	# vi: set ft=ruby :

	site :opscode

	cookbook "apt"
	cookbook "apache2"
	cookbook "database"
```

<aside class="notes">
</aside>

# Vagrantfile

```ruby
	# -*- mode: ruby -*-
	# vi: set ft=ruby :

	@user_configuration = {}
	load File.join(File.dirname(__FILE__),"config","stages.rb")

	# This is specifically design for VirtualBox. You could fairly easily adapt it to another Vagrant provider such as VMWare Fusion or AWS.
	Vagrant.configure("2") do |config|

	  config.ssh.forward_agent = true

	  # These types of options should be project specific:
	  config.vm.box = "precise64"
	  config.vm.box_url = "http://files.vagrantup.com/precise64.box"

	  @user_configuration["vagrants"].each do |name|
		config.vm.define "#{name}" do |vagrant|
		  vagrant.vm.hostname = "#{name}"
		  vagrant.vm.network :private_network, type: :dhcp
		end
	  end
	end
```

<aside class="notes">
</aside>


# Chef config

```ruby
	chef
	├── README.md
	├── cookbooks
	│   ├── apache2
	│   ├── apt
	│   ├── aws
	│   ├── build-essential
	│   ├── database
	│   ├── mysql
	│   ├── openssl
	│   ├── postgresql
	│   └── xfs
	├── rails4.json
	├── site-cookbooks
	│   ├── README.md
	│   └── rails4
	├── solo.rb
	└── vrails4.json
```

<aside class="notes">
- coobooks directory populated when berkshelf runs
- site-cookbooks are project-specific
- rails4.json is the main entry point for chef (holds the runlist)
</aside>

# rails4.json

```json
	{
	  "mysql":
		{
		  "server_root_password": "somethingsecret",
		  "server_repl_password": "somethingsecret",
		  "server_debian_password": "somethingsecret"
		},
	  "run_list":
		[
		  "recipe[apache2]",
		  "recipe[mysql::client]",
		  "recipe[mysql::ruby]",
		  "recipe[mysql::server]",
		  "recipe[rails4]"
		]
	}

```

<aside class="notes">
- main entry point for a chef run (holds the runlist)
- also holds some attributes (mysql passwords in this case)
- vrails4.json used for vagrant (v prefix is our convention)
- not entirely dry, but lets us keep things like backups and monitoring from running on not production nodes
</aside>


# Capistrano config

```
	config/
	├── README.md
	├── base/
	│   ├── berkshelf.rb
	│   ├── bootstrap.rb
	│   ├── chef.rb
	│   ├── ssh.rb
	│   ├── vg.rb
	│   └── warning.rb
	├── custom/
	│   ├── README.md
	│   └── custom_example.rb
	├── deploy.rb
	├── ssh_configs/
	│   └── example_ssh_config
	└── stages.rb
```

<aside class="notes">
- deploy.rb wires everything up (used to contain everything in base/ too)
- ssh configs in standard format ("installed" to `~/.ssh/project-name/hostname_ssh_config`)
- ssh configs generated and destroy for Vagrants
- vg - vagrant stuff
- berkshelf, bootstrap, chef - chef stuff (could be swapped out for puppet stuff)
- custom - project specific stuff (clean separation from generic concerns)
- warning - you're deploying to production! (waits to let you rethink this)
</aside>

# Benefits of Our Approach

- Lightweight (no master or server)
- Two important indirections
	- target node (local / remote)
	- Also easy to plug in another provisioning tool (we know Puppet and Chef both work, others probably do too)
- Uses tools most Rails developers are already familiar with (Ruby, Capistrano)
- Uses standard SSH config files to describe hosts (increasing portability and reusability)

<aside class="notes">
- local / remote is important, using the same interface for both makes it easy to build on 

</aside>

# Thanks!

# Credits

"The Swedish Chef" image - Andrew Kuchling
http://www.flickr.com/photos/akuchling/190704045/

# Questions?

<aside class="notes">
</aside>
