---
layout: post
title: Using Ansible network-engine to simplify network automation tasks 
comments: true
---

Last week at Networking field day ([NFD18](http://techfieldday.com/event/nfd18/)), Peter Sprygada ([@privateip](https://twitter.com/privateip)) gave a fantastic presentation about the problems in network automation and new initiatives from Ansible to help network and automation engineers. 

In short - network automation teams are recreating similar Ansible playbooks since most of us have similar problems to solve. For instance, adding an application to the datacenter will require creation of VLANS, extending them over port channels, creating STP priorities etc. The Ansible network-engine strives to provide pre-canned, opinionated (meaning best-practices) roles that will make automation easier to kick start and reuse across most teams.

<!--more-->


[In this section of the video](https://youtu.be/9tP75pfiZV0?t=14m55s), Peter dives into the layout of the network-engine role and how it allows the creation of "network functions" ( _BTW, I highly recommend watching the entire video_ ). 

In this blog post, I am hoping to illustrate the idea of network functions through an example. The NFD18 presentation also seemed to raise questions/concerns about Ansible's approach, potentially causing it to move away from being a simple automation tool, to becoming a complex one - like a programming language. This was interesting feedback to me. I hope to illustrate that, in contrast, the approach actually makes writing playbooks extremely easy for network engineers through this post. As always, I welcome your feedback/disagreements.


### An illustrative use case

As network engineers how many times have we dealt with a ticket/request that reads: **"My super important application needs to talk to the database on port 5555. Make this happen!"**. Well, maybe it is worded a lot more politely.....maybe. This might require of us to crawl through hundreds of devices to validate existing ACLs, to ensure whether that the traffic is already being allowed.

What if we had a "network function" that did this for us? Given the source/destination tuple, this "function" would return all matched ACLs within the infrastructure. 


### Persona based automation development

In my opinion, one of the main reason for the popularity of Ansible for network automation, has been it's simplicity and easy of use. A network engineer can write a playbook using a `YAML` file that is very readable (and easily re-callable 6 months after writing it) to manage configuration or use operational state data.

For our present use case, say something like this:


{% raw %}

``` yaml
---
- name: CHECK FOR ACL THAT MATCHES RULE
  hosts: all
  connection: network_cli
  gather_facts: no

  tasks:

    - include_role:
        name: ios_check_acls
      vars:
        protocol: 'udp'
        action: 'permit'
        src_network: '10.100.100.2'
        src_mask: '255.255.255.255'
        dst_network: '3.0.0.128'
        dst_mask: '255.255.255.128'
        dst_port: '2222'

```

{% endraw %}

As a subject matter expert of the underlying network infrastructure, this playbook is easy to read and I should be able to run this against all my devices to get an output that looks like :

``` bash
TASK [ios_check_acls : DISPLAY MATCH] ***************************************************************************************************************************************
ok: [rtr1] => {
    "output.result": [
        {
            "ace_action": "permit",
            "ace_name": "PERMIT-ALL",
            "ace_number": "10",
            "rule_match": "any -> any:"
        }
    ]
}
ok: [rtr2] => {
    "output.result": "No match found"
}
ok: [rtr3] => {
    "output.result": [
        {
            "ace_action": "permit",
            "ace_name": "TESTER",
            "ace_number": "10",
            "rule_match": "10.100.100.0 0.0.0.127 -> 3.0.0.0 0.255.255.255:"
        }
    ]
}
ok: [rtr4] => {
    "output.result": "No match found"
}


```

> Note how there is no match found on routers **rtr2** and **rtr4**. Also note how more permissive ACLs are accounted for.

This is precisely what the Ansible `network-engine` role enables. Let's break down this playbook to understand what is happening here - a 'who does what' visual:

![](/assets/playbook_flow.png)


- The playbook is simple and is written by the network SME. This playbook is uncomplicated, easy to read and revisit.
- The `ios_check_acls`  is a "network function" role and is written by advanced Ansible developers (possibly value added re-sellers, vendors, independent community developers etc). 
- The `ios_check_acls` calls a "provider" role - `ios`. This role is written by advanced Ansible developers and potentially functions as an abstraction layer for Cisco IOS device commands, like parsing command output, loading declarative configuration etc. This abstraction allows role developers to use core modules from within Ansible to build opinionated automation tasks
- The `ios` role calls the "network-engine" role which provides core functionality like command parsing.



### A deeper dive into the roles


The directory structure of the resulting playbook looks as follows:

![](/assets/dir_layout.png)

> `check_access.yaml` is the playbook written by the network SME. 

Keep in mind that the network SME has simply downloaded the necessary role from ['ansible-galaxy'](https://galaxy.ansible.com) into that `roles` directory. The dependency definitions within the role will automatically install all the other roles needed, freeing the user from having to worry about installing them.

What does this network function role look like?

![](/assets/role_dir.png)

This should look familiar. The role has been implemented using custom modules, templates and a tasks. Herein lies the advantage of using this construct - module developers, vendors, value added re-sellers and other community contributors are now easily able to push opinionated roles into Ansible galaxy, building up on existing provider roles. This makes pushing out custom modules a lot faster as well.


The full source code for this demo is available within the [ network-engine-roledemo github repo](https://github.com/termlen0/network-engine-roledemo).


### Conclusion

The introduction of the Ansible network-engine and the push towards creating "network function" roles makes the end user - the network SME's life a lot easier. Installing tried and tested roles that address majority use cases will make writing network automation playbooks a breeze. This will enable the adoption of netDevOps like never before.

While the roles themselves might be complex to write, keep the end user in mind. As a network engineer do I really care whether `YAML` is or is not a programming language? Does it matter if logic and loops are implemented in the construction of roles? As an end-user if I can get  **supported**, best-practices based roles that make my life easier and allows me to focus on higher value outcomes for my business, I'd rather leave the debate to the pedantic folks out there and get stuff done.

> BTW, you can install a working copy of the role in this example as follows:

``` bash
ansible-galaxy install termlen0.ios_check_acls 
```

There really is no easy button but Ansible network-engine and network function roles make it one step easier for the folks in the trenches.

As always, I'd love to hear your thoughts and opinions about this direction. What do you think?




