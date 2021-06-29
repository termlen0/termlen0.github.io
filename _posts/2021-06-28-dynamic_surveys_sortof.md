---
layout: post
title: Dynamic Surveys Sortof....
comments: true
---

Many organizations have requested the need for Dynamic Surveys in Ansible Tower. This would allow them to branch off on the fly as needed as well as populate items that have just been added. This blog will show you how to populate new items into surveys without having to do it by hand everytime the file changes. This could be do to for example a new OCP cluster being added or network interface being turned up and ready for use and populated in the troubleshooting field option of a lower Tier team members playbook access. Another example is if a new VMWare vCluster is added and it needs to be added as a choice for creating new vm's within.


<!--more-->

This blog post will present an implementation of taking information gained from a dynamic source and uploading it to a survey file. This will require a playbook be ran in order to update the playbook that the survey resides within however when adding for instance those interfaces or a long list of web servers to a specific load balancer it will be well worth running an extra playbook.

### Survey's as they are now

Survey's are currently static and are only changeable by the user though the tower interface and api. This allows for restoration and changes from two sources but at the same time is not automatic and requires human intervention. This can become a problem when a person adds a new interface that would otherwise be ignored from the ability to troubleshoot by Tier 1 if that line were to go down. This also works for VMWare vCluster if a server admin needs to add a new server to the new vCluster that was newly created.

### A change occurs 

First you turn up 10 new interfaces for connectivitiy between data centers or even access connectivity to new servers. After that you would need to go into every survey that allows for troubleshooting of interfaces by the lower Tier groups and add all of those interfaces to the surveys. This can be missed and is a tedious action for something that is supposed to be automated. You do have a choice of allowing the user to type in the interface number however that introduces a risk if for instance your troubleshooting playbook does a shut/no shut and they mistype the inter-datacenter connectivitiy off-limits interface thus causing a a possible major outage if something does not reconverge correctly. This also allows server admins self service within vmWare for adding vm's to things like the correct vCluster.



### Dynamic Update of these Surveys

Using a Jinja2 template we can simply update the survey pulling in the new items that are generated from the update playbook.


``` yaml
---
- hosts: all
  gather_facts: false
  collections:
    - awx.awx

  tasks:
    - name: Update vCenter job template
      awx.awx.tower_job_template:
        name: "Survey_Update"
        survey_enabled: true
        survey_spec: "{{ lookup('template', '{{ playbook_dir }}/templates/update_survey.j2') }}"
        validate_certs: no
```

### Jinja Template

This can be populated by a CSV file or by live data gathered through other means such as data modeling. In the below example we are using a playbook to gather the vCluster info from a VMWare vCenter and populate that to a file and then looping the fact for name. This will then populate the name field and update the survey. 

``` jinja2
{
  "name": "Survey_Update",
  "description": "Update survey with new vCenter Clusters",
  "spec": [
    {
      "type": "multiplechoice",
      "question_name": "Choose which vCenter Cluster you would like to use.",
      "question_description": "Choose desired value.",
      "variable": "vmware_cluster",
      "choices": [
{% for result in cluster_info %}
  {% for name in result %}
    <td>{{ name }}</td>
  {% endfor %}
{% endfor %}
      ],
      "required": true,
      "default": "Choose vCenter Cluster"
    }
  ]
}
```
### Conclusion

This does not truly make surveys dynamic but gives you an option to update them in a better way. It allows for more automation as we work toward resolving this problem within the product itself. 

Eric McLeroy
Principal Solutions Architect Ansible
