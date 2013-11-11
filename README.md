ansible-add-facts-examples
==========================

Examples of how aggregated facts could be exposed as facts and utilized in templates via an add_facts module.

Firewall example.
=================

A number of roles might be applied to a given host that might need to open ports in the firewall. As each role is played it can add the ports it wishes to expose to an aggregate variable 'exposed_ports' via the new add_facts command

```yml
  - name: Add cassandra ports to the exposed list
    add_facts: aggregate_key="exposed_ports" role_key="cassandra" ports="{{item}}"
    with_items: 
     - 7001
     - 9160
```

Note how it has a sub grouping keyed by the role that is adding the variables. The ports added are available in templates via exposed_ports['cassandra']['ports']

Another role might expose more ports in a similar fashion. 

```yml
  - name: Add jetty ports to the exposed list
    add_facts: aggregate_key="exposed_ports" role_key="jetty" ports="{{item}}"
    with_items:
      - 8080
      - 1099
```

Then at some later stage a template might want to iterate these aggregated port facts, in this case to create a firewall configuration to expose the ports. The template might look something like this 

```django
# Simple IPTables config to illustrate aggregated facts
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT DROP [0:0]

{% for role_with_exposed_ports in exposed_ports %}
# Ports exposed for '{{role_with_exposed_ports}}'
{% set ports_for_role = exposed_ports[role_with_exposed_ports] %}
{% for port in ports_for_role['ports'] %}
iptables -A INPUT -m state --state NEW -p tcp --dport {{port}} -j ACCEPT
{% endfor %}

{% endfor %}
COMMIT
```

Executing this would generate the following
```
# Simple IPTables config to illustrate aggregated facts
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT DROP [0:0]

# Ports exposed for 'jetty'
iptables -A INPUT -m state --state NEW -p tcp --dport 8080 -j ACCEPT
iptables -A INPUT -m state --state NEW -p tcp --dport 1099 -j ACCEPT

# Ports exposed for 'cassandra'
iptables -A INPUT -m state --state NEW -p tcp --dport 7001 -j ACCEPT
iptables -A INPUT -m state --state NEW -p tcp --dport 9160 -j ACCEPT

COMMIT

```

Motivation
=============
Exposing facts generically like this makes for more cohesive roles as all of the data for the role is defined in the role. It also reduces coupling between roles, without it the iptables role would need to know about each of the roles/hosts for which it needed to expose ports [for an example see here](https://github.com/jkleint/ansible-hadoop/blob/master/templates/iptables).

The level of indirection given by the exposed_ports makes the iptables role [open/closed](http://en.wikipedia.org/wiki/Open/closed_principle), that is open for extension but closed for modification; we can expose any ports we like in future roles without having to make modifications to the template. It reduces duplication and repetitive boilderplate and being more generic increases the reusability of roles across sites.