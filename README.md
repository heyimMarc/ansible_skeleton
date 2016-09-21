# ansible_skeleton
ansible skeleton from docs.ansible.com

## Best Practices
Here are some tips for making the most of Ansible and Ansible playbooks.

### Content Organization
The following section shows one of many possible ways to organize playbook content.

Your usage of Ansible should fit your needs, however, not ours, so feel free to modify this approach and organize as you see fit.

One thing you will definitely want to do though, is use the “roles” organization feature, which is documented as part of the main playbooks page. See [Playbook Roles and Include Statements](http://docs.ansible.com/ansible/playbooks_roles.html). 
You absolutely should be using roles. Roles are great. Use roles. Roles! Did we say that enough? Roles are great.

#### Directory Layout
The top level of the directory would contain files and directories like so:

```
production                \# inventory file for production servers
staging                   \# inventory file for staging environment

group_vars/
   group1                 \# here we assign variables to particular groups
   group2                 \# ""
host_vars/
   hostname1              \# if systems need specific variables, put them here
   hostname2              \# ""

library/                  \# if any custom modules, put them here (optional)
filter_plugins/           \# if any custom filter plugins, put them here (optional)

site.yml                  \# master playbook
webservers.yml            \# playbook for webserver tier
dbservers.yml             \# playbook for dbserver tier

roles/
    common/               \# this hierarchy represents a "role"
        tasks/            \#
            main.yml      \#  <-- tasks file can include smaller files if warranted
        handlers/         \#
            main.yml      \#  <-- handlers file
        templates/        \#  <-- files for use with the template resource
            ntp.conf.j2   \#  <------- templates end in .j2
        files/            \#
            bar.txt       \#  <-- files for use with the copy resource
            foo.sh        \#  <-- script files for use with the script resource
        vars/             \#
            main.yml      \#  <-- variables associated with this role
        defaults/         \#
            main.yml      \#  <-- default lower priority variables for this role
        meta/             \#
            main.yml      \#  <-- role dependencies

    webtier/              \# same kind of structure as "common" was above, done for the webtier role
    monitoring/            ""
    fooapp/               \# ""
```

#### Use Dynamic Inventory With Clouds
If you are using a cloud provider, you should not be managing your inventory in a static file. See [Dynamic Inventory](http://docs.ansible.com/ansible/intro_dynamic_inventory.html).

This does not just apply to clouds – If you have another system maintaining a canonical list of systems in your infrastructure, usage of dynamic inventory is a great idea in general.

#### How to Differentiate Staging vs Production
If managing static inventory, it is frequently asked how to differentiate different types of environments. The following example shows a good way to do this. Similar methods of grouping could be adapted to dynamic inventory (for instance, consider applying the AWS tag “environment:production”, and you’ll get a group of systems automatically discovered named “ec2_tag_environment_production”.

Let’s show a static inventory example though. Below, the production file contains the inventory of all of your production hosts.

It is suggested that you define groups based on purpose of the host (roles) and also geography or datacenter location (if applicable):

```
# file: production

[atlanta-webservers]
www-atl-1.example.com
www-atl-2.example.com

[boston-webservers]
www-bos-1.example.com
www-bos-2.example.com

[atlanta-dbservers]
db-atl-1.example.com
db-atl-2.example.com

[boston-dbservers]
db-bos-1.example.com

# webservers in all geos
[webservers:children]
atlanta-webservers
boston-webservers

# dbservers in all geos
[dbservers:children]
atlanta-dbservers
boston-dbservers

# everything in the atlanta geo
[atlanta:children]
atlanta-webservers
atlanta-dbservers

# everything in the boston geo
[boston:children]
boston-webservers
boston-dbservers
```

#### Group And Host Variables
This section extends on the previous example.

Groups are nice for organization, but that’s not all groups are good for. You can also assign variables to them! For instance, atlanta has its own NTP servers, so when setting up ntp.conf, we should use them. Let’s set those now:

```
---
# file: group_vars/atlanta
ntp: ntp-atlanta.example.com
backup: backup-atlanta.example.com
```

Variables aren’t just for geographic information either! Maybe the webservers have some configuration that doesn’t make sense for the database servers:

```
---
# file: group_vars/webservers
apacheMaxRequestsPerChild: 3000
apacheMaxClients: 900
```

If we had any default values, or values that were universally true, we would put them in a file called group_vars/all:

```
---
# file: group_vars/all
ntp: ntp-boston.example.com
backup: backup-boston.example.com
```

We can define specific hardware variance in systems in a host_vars file, but avoid doing this unless you need to:

```
---
# file: host_vars/db-bos-1.example.com
foo_agent_port: 86
bar_agent_port: 99
```

Again, if we are using dynamic inventory sources, many dynamic groups are automatically created. So a tag like “class:webserver” would load in variables from the file “group_vars/ec2_tag_class_webserver” automatically.

#### Top Level Playbooks Are Separated By Role
In site.yml, we include a playbook that defines our entire infrastructure. Note this is SUPER short, because it’s just including some other playbooks. Remember, playbooks are nothing more than lists of plays:

```
---
# file: site.yml
- include: webservers.yml
- include: dbservers.yml
```

In a file like webservers.yml (also at the top level), we simply map the configuration of the webservers group to the roles performed by the webservers group. Also notice this is incredibly short. For example:

```
---
# file: webservers.yml
- hosts: webservers
  roles:
    - common
    - webtier
```

The idea here is that we can choose to configure our whole infrastructure by “running” site.yml or we could just choose to run a subset by running webservers.yml. This is analogous to the “–limit” parameter to ansible but a little more explicit:

```
ansible-playbook site.yml --limit webservers
ansible-playbook webservers.yml
```

#### Task And Handler Organization For A Role
Below is an example tasks file that explains how a role works. Our common role here just sets up NTP, but it could do more if we wanted:

```
---
# file: roles/common/tasks/main.yml

- name: be sure ntp is installed
  yum: name=ntp state=installed
  tags: ntp

- name: be sure ntp is configured
  template: src=ntp.conf.j2 dest=/etc/ntp.conf
  notify:
    - restart ntpd
  tags: ntp

- name: be sure ntpd is running and enabled
  service: name=ntpd state=started enabled=yes
  tags: ntp
```

Here is an example handlers file. As a review, handlers are only fired when certain tasks report changes, and are run at the end of each play:

```
---
# file: roles/common/handlers/main.yml
- name: restart ntpd
  service: name=ntpd state=restarted
```

See [Playbook Roles and Include Statements](http://docs.ansible.com/ansible/playbooks_roles.html) for more information.

#### What This Organization Enables (Examples)

Above we’ve shared our basic organizational structure.

Now what sort of use cases does this layout enable? Lots! If I want to reconfigure my whole infrastructure, it’s just:

```
ansible-playbook -i production site.yml
```

What about just reconfiguring NTP on everything? Easy.:

```
ansible-playbook -i production site.yml --tags ntp
```

What about just reconfiguring my webservers?:

```
ansible-playbook -i production webservers.yml
```

What about just my webservers in Boston?:

```
ansible-playbook -i production webservers.yml --limit boston
```

What about just the first 10, and then the next 10?:

```
ansible-playbook -i production webservers.yml --limit boston[1-10]
ansible-playbook -i production webservers.yml --limit boston[11-20]
```

And of course just basic ad-hoc stuff is also possible.:

```
ansible boston -i production -m ping
ansible boston -i production -m command -a '/sbin/reboot'
```

And there are some useful commands to know (at least in 1.1 and higher):

```
# confirm what task names would be run if I ran this command and said "just ntp tasks"
ansible-playbook -i production webservers.yml --tags ntp --list-tasks

# confirm what hostnames might be communicated with if I said "limit to boston"
ansible-playbook -i production webservers.yml --limit boston --list-hosts
```

#### Deployment vs Configuration Organization
The above setup models a typical configuration topology. When doing multi-tier deployments, there are going to be some additional playbooks that hop between tiers to roll out an application. In this case, ‘site.yml’ may be augmented by playbooks like ‘deploy_exampledotcom.yml’ but the general concepts can still apply.

Consider “playbooks” as a sports metaphor – you don’t have to just have one set of plays to use against your infrastructure all the time – you can have situational plays that you use at different times and for different purposes.

Ansible allows you to deploy and configure using the same tool, so you would likely reuse groups and just keep the OS configuration in separate playbooks from the app deployment.

### Staging vs Production
As also mentioned above, a good way to keep your staging (or testing) and production environments separate is to use a separate inventory file for staging and production. This way you pick with -i what you are targeting. Keeping them all in one file can lead to surprises!

Testing things in a staging environment before trying in production is always a great idea. Your environments need not be the same size and you can use group variables to control the differences between those environments.

### Rolling Updates
Understand the ‘serial’ keyword. If updating a webserver farm you really want to use it to control how many machines you are updating at once in the batch.

See [Delegation, Rolling Updates, and Local Actions](http://docs.ansible.com/ansible/playbooks_delegation.html).

