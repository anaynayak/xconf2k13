# Cowboy Coder's guide to Infrastructure Nightmares

---


## #1 Traceability is overrated

----

Lets install nginx
<pre>
<code class="ruby">
package "nginx" do
  action :install
end
</code>
</pre>

----

<pre>
<code class="bash">
knife ssh "roles:app" interactive -a name
</code>
</pre>

<pre>
<code>
knife-ssh> nginx -v
app-b-01      nginx version: nginx/1.4.1
app-c-01      nginx version: nginx/1.2.6
app-a-01      nginx version: nginx/1.4.1
app-d-01      nginx version: nginx/1.2.7
app-e-01      nginx version: nginx/1.4.1
…
</code>
</pre>

----

Enforce package version
<pre>
<code class="ruby">
package "nginx" do
  version node['nginx']['version']
  action :install
end
</code>
</pre>

----

"Why do I need to run chef-client twice"

<pre>
<code class="ruby">
cookbook_file "/etc/yum.repos.d/new-version.repo" do
  source "custom"
  mode 00644
end

yum_package "app" do
  action :install
end
</code>
</pre>

----

Clear local cache before trying to install the application
<pre>
<code class="ruby">
cookbook_file "/etc/yum.repos.d/custom.repo" do
  source "custom"
  mode 00644
end

yum_package "custom-app" do
  action :install
  flush_cache [:before]
end
</code>
</pre>

---

## #2 Make it difficult to test in isolation

----

### Make ample use of the following

* Chef search
* Make calls over the network to multiple third-party services. e.g. AWS
* Scripts should not be repeatable. In case of failure, ensure that it takes sufficient knowledge of code to recover.

----

* How do you test things locally?
* How does the unavailability of any third-party services affect a chef-client run on your servers?
* In case of script failures, do they continue from where it left off ? Or do you need to restore state manually?

---

## #3 Use Ruby.
## DSL is for the Uncool !

----

### Do something only if another file exists

----

### Ruby allows me to do that !
<pre>
  <code class="ruby">
    #recipe1
    template_file "/opt/my_dependency" do
      source "my_dependency.erb"
      variables :something => node["my_dep"]["something"]
    end
  </code>
</pre>
<pre>
  <code class="ruby">
    #recipe2
    if File.exists("/opt/my_dependency")
      execute do
        command "some-command /opt/my_dependency"
      end
    end
  </code>
</pre>

----

### Works
## Cowboys #FTW

----

After few days
## Spawn a new client
### Doesn't work

----

## Run it again
### Works !!
Cowboy Quote here !

----

## Convergence Time
## Run Time

----

<pre>
  <code class="ruby">
    #recipe1
    template_file "/opt/my_dependency" do
      source "my_dependency.erb"
      variables :something => node["my_dep"]["something"]
    end
  </code>
</pre>
<pre>
  <code class="ruby">
    #recipe2
    execute do
      command "some-command /opt/my_dependency"
      only_if File.exists("/opt/my_dependency")
    end
  </code>
</pre>

---

## #4 Freedom from Constraints

----

roles/mongodb.rb
<pre>
  <code>
    name "mongodb"
    description "Mongodb role"
    run_list %w(recipe[ntp] recipe[mongodb])
  </code>
</pre>

----

## Lets add some logforwarder fu

<pre>
  <code>
    name "mongodb"
    description "Mongodb role"
    run_list %w(recipe[ntp] recipe[mongodb] recipe[logstash::logforwarder])
  </code>
</pre>

----

## Yea.. we have these things called environments

----

Only add it to certain environments
<pre>
  <code>
    name "mongodb"
    description "Mongodb role"
    runlist_without_logforwarder = %w(recipe[ntp] recipe[mongodb] recipe[logstash::logforwarder])
    env_run_list "production" => runlist_without_logforwarder,
             "staging" => runlist_without_logforwarder + ["recipe[logstash::logforwarder]"],
             "qa" => runlist_without_logforwarder +["recipe[logstash::logforwarder]"]
             "_default" => runlist_without_logforwarder
  </code>
</pre>

----


## Why you no promote to production?

----

* Create role specific cookbooks
<pre>
  <code>
    #cookbooks/db/recipes/default.rb
    include_recipe "ntp"
    include_recipe "mongodb"
    include_recipe "logforwarder"
  </code>
</pre>

* Version them
* Manage chef version promotion through a single place.

---

## #5 The #nodocumentation movement
<p class="fragment fade-in">a.k.a. job security</p>

----

### Think about the answers to the following:
<p class="fragment fade-in">How do I configure all my AWS instances to have a specific version of ruby even before the first chef-client run?</p>

<p class="fragment fade-in">How do I allow access from a new IP address
in my Security groups?</p>

<p class="fragment fade-in">Is there a particular sequence in which
instances in an environment need to be configured so that they all work
correctly?</p>

----

#### Document everything that cannot be interpreted from the code:

* Custom AMI
* Manual setup required to use an integration endpoint
* Known bugs a.k.a. 'Just restart the service' or 'Just retrigger it'
* Environment(s) and corresponding integration endpoints
* Monitoring, logging and other systems
* Machine setup

---

## #6 Refactoring

----

<pre>
  <code class="ruby">
    template_file "/etc/my_app/conf" do
      source "app_conf.erb"
      variables  some_conf => node['my_app']['some_conf'],
        :db_ip => node['db']['ip'],
        :db_password => node['db']['password'],
        :third_party_service_username => node['third_party_service']['username'],
        :third_party_service_password => node['third_party_service']['password'],
        :twitter_oauth_key => node['twitter']['key'],
        :twitter_oauth_secret => node['twitter']['secret']

      notifies :restart, 'service[my_app]'
    end
  </code>
</pre>

----

## Who says cowboys don't refactor !

----

<pre>
  <code class="ruby">
    template_file "/etc/my_app/conf" do
      source "app_conf.erb"       #Has a line to include other confs now
      variables  some_conf => node['my_app']['some_conf']
      notifies :restart, 'service[my_app]'
    end
  </code>
</pre>
<pre>
  <code class="ruby">
    template_file "/etc/my_app/conf.d/db" do
      source "db.conf.erb"
      variables :db_ip => node['db']['ip'],
        :db_password => node['db']['password']
    end
  </code>
</pre>

----

<pre>
  <code class="ruby">
    template_file "/etc/my_app/conf.d/third_party_service" do
      source "third_party_service.conf.erb"
      variables :third_party_service_username => node['third_party_service']['username'],
        :third_party_service_password => node['third_party_service']['password']
    end
  </code>
</pre>
<pre>
  <code class="ruby">

    template_file "/etc/my_app/conf.d/twitter" do
      source "twitter.conf.erb"
      variables :twitter_oauth_key => node['twitter']['key'],
        :twitter_oauth_secret => node['twitter']['secret']
    end
  </code>
</pre>

----

### Wow !! Modular Conf
### Cowboys #FTW

----

# But
## 1 Month later

----

#### Somebody tries to change the 3rd party conf
## App doesn't know about it !

----

### Its there in the conf file !

----

## Look closely

<pre>
  <code class="ruby">
    template_file "/etc/my_app/conf.d/third_party_service" do
      source "third_party_service.conf.erb"
      variables :third_party_service_username => node['third_party_service']['username'],
        :third_party_service_password => node['third_party_service']['password']
    end
  </code>
</pre>

----

### No Notification to
# Restart !

---

## #6 Ignorance is bliss

----

Lets install django

<pre>
  <code>
    
    execute "install django" do
      command "pip install django==1.0.4"
    end
	
    ...
  </code>
</pre>

----

Why is my chef-client run so slow?

----

<pre>
  <code>
    
    execute "install django" do
      command "pip install django==1.0.4"
      not_if "pip freeze | grep django==1.0.4"
    end
    ...
  </code>
</pre>

----

Download a remote file (but not always!)

<pre>
  <code>
    remote_file "#{Chef::Config[:file_cache_path]}/large-file.tar.gz" do

      source "http://www.example.org/large-file.tar.gz"
      <span class="fragment fade-in">checksum "3a7dac00b1" #RTFM</span>
    end
    ...
  </code>
</pre>
