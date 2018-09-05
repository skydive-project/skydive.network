---
title: Network topology discovery with Ansible and Skydive
layout: blog-post
author: Sylvain Afchain
date: 05/09/2018
---

Since `Skydive` already has a `Python` client [library](/documentation/api-python) I thought it was "fun" to create an `Ansible` module leveraging it to add topology entities. In this blog post I will show how to use this module and how to use this module to provide real topology information.

## Nodes and Links

The module is part of the Skydive repository, so the first thing is to prepare the `Ansible` environment by download the module in the proper place.

{% highlight shell %}
mkdir skydive-ansible
cd skydive-ansible
mkdir library
curl -LO library/skydive_node.py http://
curl -LO library/skydive_edge.py http://
{% endhighlight %}

Now we have the module available, let's write a really simple playbook creating two nodes, one will be a "Top of Rack" switch
 and the other one will be a port. Of course we will create a link between them.

{% highlight yaml %}
vi playbook.yaml
{% endhighlight %}

{% highlight yaml %}
- name: Skydive Ansible
  hosts: localhost
  tasks:

    - name: Create TOR
      skydive_node:
        name: 'TOR'
        type: "fabric"
        metadata:
          Model: My favorite Switch brand
          ManagementIP: 10.0.0.1
      register: tor_result

    - name: Create port
      skydive_node:
        name: 'PORT'
        type: 'fabric'
      register: port_result
    
    - name: Link TOR and port  
      skydive_edge:
        node1: "{{ tor_result.UUID }}"
        node2: "{{ port_result.UUID }}"
        relation_type: ownership
        metadata:
          Type: unknown
{% endhighlight %}

You can use the `metadata` dict to add any arbitrary information you want.

Once executed we'll see the entities in the topology.

<p>
  <a href="/assets/images/blog/ansible-lib-1.png" data-lightbox="WebUI-1" data-title="Skydive WebUI">
    <img src="/assets/images/blog/ansible-lib-1.png"/>
  </a>
</p>

`node1` and `node2` can be either the `ID` of the nodes or a `Gremlin` expression.
The following playbook shows a mix of `ID` and a [Gremlin](/documentation/api-gremlin) expression to link the switch
port to a host interface. 

{% highlight yaml %}
- name: Skydive Ansible
  hosts: localhost
  tasks:

    - name: Create TOR
      skydive_node:
        name: 'TOR'
        type: "fabric"
        metadata:
          Model: My favorite Switch brand
          ManagementIP: 10.0.0.1
      register: tor_result

    - name: Create port
      skydive_node:
        name: 'PORT'
        type: 'fabric'
      register: port_result
    
    - name: Link TOR and port  
      skydive_edge:
        node1: "{{ tor_result.UUID }}"
        node2: "{{ port_result.UUID }}"
        relation_type: ownership

    - name: Link port and eth0  
      skydive_edge:
        node1: "{{ port_result.UUID }}"
        node2: "G.V().Has('Name', 'enp0s25')"
        relation_type: layer2
{% endhighlight %}

This gives the following topology.

<p>
  <a href="/assets/images/blog/ansible-lib-2.png" data-lightbox="WebUI-1" data-title="Skydive WebUI">
    <img src="/assets/images/blog/ansible-lib-2.png"/>
  </a>
</p>


## Ansible, Skydive and LLDP

Finally the following example will be a bit more concrete. It uses `LLDP` to retrieve physical topology information
and the `Skydive` module to inject them into the `Skydive` topology.

{% highlight yaml %}
- name: Skydive Ansible
  hosts: localhost
  tasks:

  - name: Gather information from lldp
    lldp:
    become: true

  - name: Create TOR
    skydive_node:
      name: "TOR - {{ lldp[item]['chassis']['name'] }}"
      type: "fabric"
      seed: "{{ lldp[item]['chassis']['name'] }}:{{ lldp[item]['chassis']['mac'] }}"
      metadata:
        MAC: "{{ lldp[item]['chassis']['mac'] }}"
        Description: "{{ lldp[item]['chassis']['descr'] }}"
        Address: "{{ lldp[item]['chassis']['mgmt-ip'] }}"
    register: tors
    with_items: "{{ lldp.keys() }}"

  - name: Create port
    skydive_node:
      name: "{{ lldp[item]['port']['descr'] }}"
      type: "fabric"
      seed: "{{ lldp[item]['port']['descr'] }}:{{ lldp[item]['port']['mac'] }}"
      metadata:
        MAC: "{{ lldp[item]['port']['mac'] }}"
    register: ports
    with_items: "{{ lldp.keys() }}"
    
  - name: Link port to TOR
    skydive_edge:
      node1: "{{ item.0.UUID }}"
      node2: "{{ item.1.UUID }}"
      relation_type: ownership
    with_together:
      - "{{ tors.results }}"
      - "{{ ports.results }}"

  - name: Link port to host interface
    skydive_edge:
      node1: "{{ item.0.UUID }}"
      node2: "G.V().Has('Type', 'host').Out().Has('Name', '{{ item.1 }}')"
      relation_type: ownership
    with_together:
      - "{{ ports.results }}"
      - "{{ lldp.keys() }}"
{% endhighlight %}

## conclusion

As we saw the `Skydive` module allows to leverage the 
[the Network Automation Ansible ecosystem](https://www.ansible.com/integrations/networks).
The current `Skydive` module is a first version that surely needs to be improved. And, as usual, contributions are more than welcome :)