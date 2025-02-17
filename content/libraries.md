+++
title = "About Libraries"
draft = false
gh_repo = "chef-web-docs"
aliases = ["/libraries.html"]

[menu]
  [menu.infra]
    title = "Libraries"
    identifier = "chef_infra/cookbook_reference/libraries.md Libraries"
    parent = "chef_infra/cookbook_reference"
    weight = 50
+++

{{% libraries_summary %}}

Use a library to:

- Connect to a database
- Fetch secrets from a cloud provider
- Talk to an LDAP provider
- Do anything that can be done with Ruby

## Syntax

The syntax for a library varies because library files are created using
Ruby and are designed to handle custom situations. See the Examples
section below for samples.

## Template Helper Modules

{{% resource_template_library_module %}}

## Examples

The following examples show how to use cookbook libraries.

### Create a Namespace

A database can contain a list of virtual hosts that are used by
customers. A custom namespace could be created that looks something
like:

```ruby
# Sample provided by "Arjuna (fujin)". Thank you!

require 'sequel'

class Chef::Recipe::ISP
  # We can call this with ISP.vhosts
  def self.vhosts
    v = []
    @db = Sequel.mysql(
      'web',
      user: 'example',
      password: 'example_pw',
      host: 'dbserver.example.com'
    )
    @db[
      "SELECT virtualhost.domainname,
           usertable.userid,
           usertable.uid,
           usertable.gid,
           usertable.homedir
       FROM usertable, virtualhost
       WHERE usertable.userid = virtualhost.user_name"
      ].all do |query|
      vhost_data = {
        servername: query[:domainname],
        documentroot: query[:homedir],
        uid: query[:uid],
        gid: query[:gid],
      }
      v.push(vhost_data)
    end
    Chef::Log.debug('About to provision #{v.length} vhosts')
    v
  end
end
```

After the custom namespace is created, it could then be used in a
recipe, like this:

```ruby
ISP.vhosts.each do |vhost|
  directory vhost[:documentroot] do
    owner vhost[:uid]
    group vhost[:gid]
    mode '0755'
    action :create
  end

  directory "#{vhost[:documentroot]}/#{vhost[:domainname]}" do
    owner vhost[:uid]
    group vhost[:gid]
    mode '0755'
    action :create
  end
end
```

### Extend a Recipe

A customer record is stored in an attribute file that looks like this:

```ruby
mycompany_customers({
  bob: {
    homedir: '/home/bob',
    webdir: '/home/bob/web',
  },
}
)
```

A simple recipe may contain something like this:

```ruby
directory node["mycompany_customers"]["bob"]["webdir"] do
  owner 'bob'
  group 'bob'
  action :create
end
```

Or a less verbose version of the same simple recipe:

```ruby
directory customer(:bob)[:webdir] do
  owner 'bob'
  group 'bob'
  action :create
end
```

A simple library could be created that extends `Chef::Recipe::`, like
this:

```ruby
class Chef
  class Recipe
    # A shortcut to a customer
    def customer(name)
      node["mycompany_customers"][name]
    end
  end
end
```

### Loop Over a Record

A customer record is stored in an attribute file that looks like this:

```ruby
mycompany_customers({
  bob: {
    homedir: '/home/bob',
    webdir: '/home/bob/web',
  },
}
)
```

If there are many customer records in an environment, a simple recipe
can be used to loop over every customer, like this:

```ruby
all_customers do |name, info|
  directory info[:webdir] do
    owner name
    group name
    action :create
  end
end
```

A simple library could be created that extends `Chef::Recipe::`, like
this:

```ruby
class Chef
  class Recipe
    def all_customers(&block)
      node["mycompany_customers"].each do |name, info|
        block.call(name, info)
      end
    end
  end
end
```
