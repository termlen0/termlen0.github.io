---
layout: post
title: Part1 of a 2 part blog on using the Ansible network-engine's command parser 
comments: true
---


### A very brief introduction

The [network-engine](https://github.com/ansible-network/network-engine/blob/devel/docs/user_guide/README.md) role was made available through Ansible galaxy recently. One of the modules this role makes available for network engineers, is the [command parser](https://github.com/ansible-network/network-engine/blob/devel/docs/user_guide/command_parser.md). As the name implies, command parser enables the user to parse the output of _show_ commands - commands that network engineers know and love, that are "pretty" formatted but not structured. 

Until recently, I had only used [TextFSM](https://github.com/google/textfsm) to do this. While TextFSM works, it has a significant learning curve, IMO. Last week I decided to give the `command_parser` module a spin and right off the bat my impressions were:

1. If you are already comfortable using Ansible, you will feel at home working with the command parser

2. The learning curve is much shorter without needing to worry about learning another domain specific language


> Note: This is in no way pitting one against the other. **network-engine** provides a **textfsm_parser** as well as the **command_parser**

<!--more-->

My intention through the 2 part blog, is to write about how to build command parser templates using 2 examples. I hope it will help others and will serve as a reference when I need it later.

However, first thing first - The official documentation is available here: [command parser directives](https://github.com/ansible-network/network-engine/blob/devel/docs/directives/parser_directives.md)

### Diving in: parsing the output of `show ip interfaces brief` on a Cisco IOS device


Rather than reiterate what is in the documentation, lets dive in with an example. We'll start off with the playbook that captures the output of the `show ip interfaces brief` command.


``` yaml
---
- name: GENERATE A REPORT
  hosts: cisco
  gather_facts: no
  connection: network_cli

  roles:
    - ansible-network.network-engine

  tasks:
  - name: CAPTURE SHOW IP INTERFACE
    ios_command:
      commands:
        - show ip interface brief
    register: output
    
  - name: DISPLAY THE OUTPUT
    debug: var=output.stdout

```


This should hopefully look familiar to folks who are used to using Ansible for working with network devices. Pay attention to the 

``` yaml
  roles:
    - ansible-network.network-engine

```

This is the role that provides the `command_parser` library that we will be working with. This role can be downloaded and installed as per instructions in the links above.


On running the playbook (and limiting it to a single device):

``` shell
PLAY [GENERATE A REPORT ] ***************************************************************************

TASK [CAPTURE SHOW IP INTERFACE] ****************************************************************************
ok: [rtr1]

TASK [DISPLAY THE OUTPUT] *****************************************************************************************************************************************************************************************
ok: [rtr1] => {
    "output.stdout": [
        "Interface              IP-Address      OK? Method Status                Protocol\nGigabitEthernet1       172.16.244.50   YES DHCP   up                    up      \nLoopback0              192.168.1.101   YES manual up                    up      \nLoopback1              10.1.1.101      YES manual administratively down down    \nTunnel0                10.100.100.1    YES manual up                    up      \nTunnel1                10.200.200.1    YES manual up                    up      \nVirtualPortGroup0      192.168.35.101  YES TFTP   up                    up"
    ]
}

PLAY RECAP ********************************************************************************************************************************************************************************************************
rtr1                       : ok=2    changed=0    unreachable=0    failed=0  


```

So far so good. We got the output of the show command. Now we can send this "blob" of text to a command parser that can create structured data from it.



``` yaml
{% raw %}
---
- name: DYNAMIC REPORTING PART
  hosts: cisco
  gather_facts: no
  connection: network_cli

  roles:
    - ansible-network.network-engine

  tasks:
  - name: CAPTURE SHOW IP INTERFACE
    ios_command:
      commands:
        - show ip interface brief
    register: output

  - name: DISPLAY THE OUTPUT
    debug: var=output.stdout

  - name: PARSE THE RAW OUTPUT
    command_parser:
      file: "parsers/ios/show_ip_interface_brief.yaml"
      content: "{{ output.stdout[0] }}"

{% endraw %}


```


Now, we get into the meat of this blog. Using the `command_parser` module. 

### Writing our first parser

This is my workflow for creating a new parser:

1. Identify the regular expression/expressions needed to collect the data using regex101.com

2. Use the [parser directives](https://github.com/ansible-network/network-engine/blob/devel/docs/directives/parser_directives.md) to test out the regular expression

3. Iterate and refine


So let's begin with creating the regular expression. Let's say we want to capture the _Interface name, IP address, Admin state and Protocol State_ . The following regular expression is what I came up with:


``` 
^(\\S+)\\s+(\\d+\\.\\d+\\.\\d+\\.\\d+).*(up|administratively down).*(up|down)
```

On to step 2 testing this. The parser file defined by 

``` yaml
      file: "parsers/ios/show_ip_interface_brief.yaml"

```

is a YAML file. 


``` yaml
---
- name: PARSER META DATA
  parser_metadata:
    version: 1.0
    command: show ip interface brief
    network_os: ios

- name: MATCH PATTERN
  pattern_match:
    regex: "^(\\S+)\\s+(\\d+\\.\\d+\\.\\d+\\.\\d+).*(up|administratively down).*(up|down)"
    match_all: yes
  register: section
  export: yes

```

Here we are using the `pattern_match` directive. The regex is now applied to the blob of input passed in from the playbook task. The match groups are collected and stored into the variable called `section`. We use the `match_all` option in order to match all the capture groups.

Note the use of the `export: yes`. Without this directive the variable and value contained will not be sent back to the playbook for further processing. 

>As part of our development of the parser we might have to use the `export: yes` at multiple stages.


In order to test this, re-run the playbook but in `verbose` mode:


``` shell

TASK [PARSE THE RAW OUTPUT] ***************************************************************************************************************************************************************************************
ok: [rtr1] => {"ansible_facts": {"section": [{"matches": ["GigabitEthernet1", "172.16.244.50", "up", "up"]}, {"matches": ["Loopback0", "192.168.1.101", "up", "up"]}, {"matches": ["Loopback1", "10.1.1.101", "administratively down", "down"]}, {"matches": ["Tunnel0", "10.100.100.1", "up", "up"]}, {"matches": ["Tunnel1", "10.200.200.1", "up", "up"]}, {"matches": ["VirtualPortGroup0", "192.168.35.101", "up", "up"]}]}, "changed": false, "included": ["parsers/ios/show_ip_interface_brief.yaml"]}

PLAY RECAP ********************************************************************************************************************************************************************************************************
rtr1                       : ok=2    changed=0    unreachable=0    failed=0   

```

Great! We can already see the data being returned as list of dictionaries. Now within the command parser you have the option of cleaning up and presenting this data as a structured `json` object. Let's do that as the next step using the `json_template` directive:



``` yaml
{% raw %}
---
- name: PARSER META DATA
  parser_metadata:
    version: 1.0
    command: show ip interface brief
    network_os: ios

- name: MATCH PATTERN
  pattern_match:
    regex: "^(\\S+)\\s+(\\d+\\.\\d+\\.\\d+\\.\\d+).*(up|administratively down).*(up|down)"
    match_all: yes
  register: section

- name: GENERATE JSON DATA STRUCTURE
  json_template:
    template:
      - key: "{{ item.matches.0 }}"
        object:
        - key: data
          object:
            - key: name
              value: "{{ item.matches.0 }}"
            - key: ip
              value: "{{ item.matches.1 }}"
            - key: admin_state
              value: "{{ item.matches.2 }}"
            - key: protocol_state
              value: "{{ item.matches.3 }}"
  loop: "{{ section }}"
  export: yes
  register: ip_interface_facts

{% endraw %}
```


The previous match was registered into a variable called `section`. This is a list as seen from the verbose output above. This list can be looped over to work on individual elements using the `loop` directive. The `json_template` directive is used to define a nested dictionary with the key of each element being the name of the interface in this example.

> Note: The **export: yes** has been moved from the **section** variable to the **ip_interface_facts** variable. 


Once exported the variable becomes available within the playbook and can be used just like any other variable in Ansible. Let's go ahead and display this using `debug`:


``` yaml
{% raw %}
---
- name: DYNAMIC REPORTING PART
  hosts: cisco
  gather_facts: no
  connection: network_cli

  roles:
    - ansible-network.network-engine

  tasks:
  - name: CAPTURE SHOW IP ROUTE
    ios_command:
      commands:
        - show ip interface brief
    register: output

  - name: PARSE THE RAW OUTPUT
    command_parser:
      file: "parsers/ios/show_ip_interface_brief.yaml"
      content: "{{ output.stdout[0] }}"

  - name: DISPLAY THE DATA
    debug: var=ip_interface_facts
    
{% endraw %}

```



Now when we run the playbook:


``` shell
TASK [DISPLAY THE DATA] *******************************************************************************************************************************************************************************************
ok: [rtr1] => {
    "ip_interface_facts": [
        {
            "GigabitEthernet1": {
                "data": {
                    "admin_state": "up", 
                    "ip": "172.16.244.50", 
                    "name": "GigabitEthernet1", 
                    "protocol_state": "up"
                }
            }
        }, 
        {
            "Loopback0": {
                "data": {
                    "admin_state": "up", 
                    "ip": "192.168.1.101", 
                    "name": "Loopback0", 
                    "protocol_state": "up"
                }
            }
        }, 
        {
            "Loopback1": {
                "data": {
                    "admin_state": "administratively down", 
                    "ip": "10.1.1.101", 
                    "name": "Loopback1", 
                    "protocol_state": "down"
                }
            }
        }, 
        {
            "Tunnel0": {
                "data": {
                    "admin_state": "up", 
                    "ip": "10.100.100.1", 
                    "name": "Tunnel0", 
.
.
.
.
.
<output omitted for brevity>
```


Now, wasn't that easy!! At least I think it was a lot easier to work with personally than TextFSM. 


In part 2 of the blog series we will build a command parser for the `show interfaces` command on IOS devices. This will be a little more complex than the example in part 1 and will hopefully help demonstrate how really powerful the `command_parser` module is.



{% include twitter_plug.html %}
