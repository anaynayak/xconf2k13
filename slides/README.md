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

### Do something only if after another file exists

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

  <code>
    include_recipe "recipe1"
    if File.exists("/opt/my_dependency")
      execute do
        command "some-command /opt/my_dependency"
      end
    end
  </code>
</pre>

----

## But
### Doesn't work

----

### Run it again
## Works !!

----

## Convergence Time
## Run Time

----

<pre>
  <code class="ruby">
    template_file "/opt/my_dependency" do
      source "my_dependency.erb"
      variables :something => node["my_dep"]["something"]
    end

    execute do
      command "some-command /opt/my_dependency"
      only_if File.exists("/opt/my_dependency")
    end
  </code>
</pre>

---

## Freedom from Constraints

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
