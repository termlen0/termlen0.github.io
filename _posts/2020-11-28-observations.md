---
layout: post
title: The idea of a "CareTaker" for git centeric network operations
comments: true
---

Many organizations have embraced or are beginning to embrace the
configuration management of their network devices through code. A
common pattern that I've come across consists of:

1. A git repository for scrubbed, last-known-good-configs
2. Another git repository that holds the infrastructure as code (IaC)

Operators create feature branches off of the second repo, and submit
pull-requests. These changes are then subject to various tests (manual
or automatic) before being approved and merged into the master branch,
which then gets deployed onto the end points.

But the idea of GitOps goes further than that – it uses tools to
compare the actual production state of your endpoints with what's
under source control and then it tells you when they don't match. This
post introduces the idea of a "CareTaker" that ensures that detects
out-of-band (OOB) changes made to the network endpoints and alerts the
operator to make a manual decision.


<!--more-->

This blog post will present an implementation of such a "CareTaker"
using an Ansible collection. A demo of this implementation is seen
here:

<iframe width="560" height="315" src="https://www.youtube.com/embed/wiUFKtR91Rw" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

### Steady State Operations

The steady state operations might look as follows:
![Steady state Git Ops](/assets/caretaker_steady_state.png )

The operator creates a pull request against the IaC repository and
after this is merged, Ansible pushes the configuration to the end
points.

### An OOB change is introduced

During production operations, there might be situations where an
engineer might have to go in and make changes manually to the
endpoints. Possibly due to:

- Incidents caused by bugs or failed links or failed hardware.
- Configurations that are not yet under the purview of the IaC
automation.

The above two are just 2 potential examples. There might be numerous
valid reasons to make these OOB changes.

The potential also exists for someone making an incorrect
change. Maybe an engineer who is not very familiar with the
operational protocols of the organization.

TL;DR an OOB change has now been introduced into the network and
running configs are no longer in synch with the last-known-good
configurations.


### Impact of the OOB on Git centric operations

Let's say the OOB change was made to the data-model under IaC control - to
say shutdown a bgp neighbor to address unreachable Autonomous Systems
through that neighbor. A very valid production reason to go implement
the fix. With this fix in place, the IaC model no longer reflects
what's running. Nor does the last-known-good config match what's
running. Say, the IaC repo had the BGP configs for a router defined as
follows:

``` yaml
---
bgp_config:
  bgp_as: 65000
  router_id: 192.168.2.2
  log_neighbor_changes: true
  address_family:
  neighbors:
    - neighbor: "{{ "{{ remote_public_ip "}} }}"
      remote_as: 65000
  networks:
    - prefix: 10.25.25.0
      masklen: 24
    - prefix: 10.25.26.0
      masklen: 24
```


At this point, say, an engineer who is unaware of this fix,
tries to advertise a new network to this IaC; by adding a new network
to the list of advertised networks

``` yaml
    - prefix: 10.25.27.0
      masklen: 24
```

This change, might slip through normal testing because everything is
valid about it (syntax, value of network IP etc). If this change gets
propagated to the network, it will revert the manual fix put in place
earlier and re-introduce the issue.

Another scenario to consider for the impact may be configurations that
are not being tracked by the IaC. The above IaC is only tracking BGP
configurations. However, there might be changes made on the device
(say to the IGP) that can potentially impact the effect of this
configuration. While this scenario is less likely, you would want your
automation systems to make sure that the running configurations at
least match your last-known-good. If this is not the case, it should
alert you about an OOB configuration and allow the operator to
manually decide whether the last-know-good config should be replaced
with running or vice versa before pushing their changes onto
production.



### Caretaker

    Captain Kathryn Janeway : Did you ever consider allowing the
    Ocampa to care for themselves?
    The Caretaker : [laughs pitifully] They're children.

                                         - Star Trek Voyager, S1E1

To address this, I created a demo collection called the caretaker:
https://galaxy.ansible.com/termlen0/caretaker

> This is an illustrative demo collection and not to be used in
> production. Currently, it is only implemented for EOS devices.

The collection uses a role that basically compares the configurations
from the last-known-good configuration repo with the current running
configs of the devices. If it detects a difference, it simply
fails. That's it! Very simple.

How do you invoke this in your playbook?

``` yaml
# deploy_bgp.yml
---
- name: Deploy IaC configs to BGP routers
  hosts: arista
  gather_facts: no
  pre_tasks:
    - name: Invoke Caretaker
      include_role:
        name: termlen0.caretaker.caretaker

  tasks:

    - name: Configure the routing protocol
      arista.eos.eos_bgp:
        config: "{{ "{{ bgp_config "}} }}"
        operation: replace
```

So the idea here is that caretaker would act as a **meta task** that
is always included in every deployment playbook. This way, the
engineer implementing the change can have absolute confidence that the
configuration being deployed will only succeed if there are no OOB
changes on the network devices.

### Using Caretaker with Tower

The simplicity of this collection lies in the fact that it is easy to
add to existing playbooks. Using Ansible Tower's workflows, the
deployments can now become a lot more intelligent.

![Tower WorkFlow](/assets/caretaker_workflow.png )

The above workflow uses deploy playbook with the caretaker
collection. During normal operations - meaning, no OOB changes - the
playbook would run normally, succeed, and the new running configs will
be backed up to the last-known-good config repo (green path).

The playbook will fail if caretaker detects OOB changes. It then uses
approval nodes to alert the operator. A human needs to intervene at
this point and decide whether the running configs should be replaced
with the last-known-good configs before deploying the new change or
alternately, the last-known-good configs should be updated before the
changes are made.


### Some additional notes

#### What are approval nodes?

[Bianca](https://www.ansible.com/blog/author/bianca-henderson) has a
detailed blog post on [Approval
Workflows](https://www.ansible.com/blog/how-to-add-approval-steps-to-ansible-tower-workflows)


#### Integration with GitHub

Tower workflows can be invoked via webhooks. So for our use case
example, the operator is simply making a Pull Request to the BGP
data-model and once merged, it kicks off the Tower workflow.

![Workflow Webhook](/assets/caretaker_webhook.png )

[Tim Appnel](https://www.ansible.com/blog/author/timothy-appnel) has a
2-part in depth
[blog](https://www.ansible.com/blog/intro-to-automation-webhooks-for-red-hat-ansible-automation-platform)
on using Web Hooks in Ansible Tower and is a great read for Git Ops in
general.


#### ChatOps

Ansible Tower has so many built-in notification options. Tools like
Slack are popular among teams and organizations that are embracing
DevOps. For this demo, each time human attention is needed (for the
approval nodes), the workflow sends a slack team notification:

![Alt Text](/assets/caretaker_chatops.png )


### Conclusion

Though the idea behind the blog is to highlight how a collection like
caretaker can be used to increase confidence in Git centric
operations, it also illustrates a secondary idea. About Ansible
collections in general. This shows how higher abstractions of
automation can now be bundled as a collection, ready to be consumed by
playbooks.
