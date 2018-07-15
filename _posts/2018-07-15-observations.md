---
layout: post
title: Part2 of a 2 part blog on using the Ansible network-engine's command parser 
comments: true
---

In [Part 1](https://termlen0.github.io/2018/06/26/observations/) of this 2 part series, I illustrated how to invoke the `command_parser` module using the `network-engine` role from Ansible. I then used it to illustrate how to build a simple parser, leveraging regex to convert unstructured device command output to structured data.

In this post, I'll build on it, highlighting the command parser options that makes wrangling _any_ complex device output into structured output. In particular this post will deep-dive into the following directives:

- `pattern_group`
- `extend`


**The playbook and complete parser is available in [this git repo.](https://github.com/termlen0/parser_demo)**

<!--more-->

### Understanding the output pattern

We used `pattern_match` in the last example to parse the output of the `show ip interface brief` command. 
{% raw %}

``` yaml
- name: MATCH PATTERN
  pattern_match:
    regex: "^(\\S+)\\s+(\\d+\\.\\d+\\.\\d+\\.\\d+).*(up|administratively down).*(up|down)"
    match_all: yes
  register: section
```
{% endraw %}

Note that we used the `match_all` parameter to match each regex group. This worked well for matching against the following command output, because we were working with a single line at a time.

``` bash
rtr1#sh ip int bri
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet1       172.16.230.103  YES DHCP   up                    up      
Loopback0              192.168.1.101   YES manual up                    up      
rtr1#
```

What if we wanted to loop over entire sections in the output? Let's illustrate using an example:

``` bash
rtr1#sh interfaces 
GigabitEthernet1 is up, line protocol is up 
  Hardware is CSR vNIC, address is 0e56.892c.0434 (bia 0e56.892c.0434)
  Internet address is 172.16.230.103/16
  MTU 1500 bytes, BW 1000000 Kbit/sec, DLY 10 usec, 
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation ARPA, loopback not set
  Keepalive set (10 sec)
  Full Duplex, 1000Mbps, link type is auto, media type is Virtual
  output flow-control is unsupported, input flow-control is unsupported
  ARP type: ARPA, ARP Timeout 04:00:00
  Last input 00:00:00, output 00:00:00, output hang never
  Last clearing of "show interface" counters never
  Input queue: 0/375/0/0 (size/max/drops/flushes); Total output drops: 0
  Queueing strategy: fifo
  Output queue: 0/40 (size/max)
  5 minute input rate 0 bits/sec, 0 packets/sec
  5 minute output rate 1000 bits/sec, 1 packets/sec
     204069 packets input, 21320843 bytes, 0 no buffer
     Received 0 broadcasts (0 IP multicasts)
     0 runts, 0 giants, 0 throttles 
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored
     0 watchdog, 0 multicast, 0 pause input
     205563 packets output, 26944541 bytes, 0 underruns
     0 output errors, 0 collisions, 1 interface resets
     0 unknown protocol drops
     0 babbles, 0 late collision, 0 deferred
     0 lost carrier, 0 no carrier, 0 pause output
     0 output buffer failures, 0 output buffers swapped out
Loopback0 is up, line protocol is up 
  Hardware is Loopback
  Internet address is 192.168.1.101/24
  MTU 1514 bytes, BW 8000000 Kbit/sec, DLY 5000 usec, 
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation LOOPBACK, loopback not set
  Keepalive set (10 sec)
  Last input never, output never, output hang never
  Last clearing of "show interface" counters never
.
.
.
.
.
<output omitted for brevity>
```

The `show interfaces` command displays a wealth of information per interface. Let's say we want to capture the following information:

- Interface name
- MTU
- txload
- rxload


### Building a list 
The first step is to build a list using `pattern_match` where each element of the list corresponds to all the information about an interface.

{% raw %}

``` yaml
---
- name: parser meta data
  parser_metadata:
    version: 1.0
    command: show interface
    network_os: ios

- name: match sections
  pattern_match:
    regex: "^(\\S+) is.*(?:up|down),"
    match_all: yes
    match_greedy: yes
  register: section
  export: yes

```

{% endraw %}

> The `match_greedy` will match everything after the regex match. The `match_all` will match each interface as a distinct match and add it to the list variable called section. 

Debugging this output in the playbook reveals the contents of the `section` variable:


{% raw %}

``` bash

TASK [DISPLAY THE PARSED OUTPUT] **************************************************************************************************ok: [rtr1] => {
    "section": [
        "GigabitEthernet1 is up, line protocol is up \n  Hardware is CSR vNIC, address is 0e56.892c.0434 (bia 0e56.892c.0434)\n  Internet address is 172.16.230.103/16\n  MTU 1500 bytes, BW 1000000 Kbit/sec, DLY 10 usec, \n     reliability 255/255, txload 1/255, rxload 1/255\n  Encapsulation ARPA, loopback not set\n  Keepalive set (10 sec)\n  Full Duplex, 1000Mbps, link type is auto, media type is Virtual\n  output flow-control is unsupported, input flow-control is unsupported\n  ARP type: ARPA, ARP Timeout 04:00:00\n  Last input 00:00:00, output 00:00:00, output hang never\n  Last clearing of \"show interface\" counters never\n  Input queue: 0/375/0/0 (size/max/drops/flushes); Total output drops: 0\n  Queueing strategy: fifo\n  Output queue: 0/40 (size/max)\n  5 minute input 
rate 1000 bits/sec, 2 packets/sec\n  5 minute output rate 1000 bits/sec, 2 packets/sec\n     206723 packets input, 21601205 bytes, 0 no buffer\n     Received 0 broadcasts (0 IP multicasts)\n     0 runts, 0 giants, 0 throttles \n     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored\n     0 watchdog, 0 multicast, 0 pause input\n     208526 packets output, 27353095 bytes, 0 underruns\n     0 output errors, 0 collisions, 1 interface resets\n     0 unknown protocol drops\n     0 babbles, 0 late collision, 0 deferred\n     0 lost carrier, 0 no carrier, 0 pause output\n     0 output buffer failures, 0 output buffers swapped out\n", 
        "Loopback0 is up, line protocol is up \n  Hardware is Loopback\n  Internet address is 192.168.1.101/24\n  MTU 1514 bytes, BW 8000000 Kbit/sec, DLY 5000 usec, \n     reliability 255/255, txload 1/255, rxload 1/255\n  Encapsulation LOOPBACK, loopback not set\n  Keepalive set (10 sec)\n  Last input never, output never, output hang never\n  Last clearing of \"show interface\" counters never\n  Input queue: 0/75/0/0 (size/max/drops/flushes); Total output drops: 0\n  Que
.
.
.
.
.
<output omitted for brevity>
```
{% endraw %}

> Note that all info about Gig1 is the first element of this list. Loopback0 is the second element and so on.


### Building a pattern group

Now that we have list, we can use the same design patterns we are familiar with in Ansible, using loops. Using the `pattern_group` directive we can now build a new nested variable that uses the `pattern_match` directive.


{% raw %}

``` yaml
- name: match interface values
  pattern_group:
    - name: match name
      pattern_match:
        regex: "^(\\S+)"
        content: "{{ item }}"
      register: name

    - name: match mtu
      pattern_match:
        regex: "MTU (\\d+)"
        content: "{{ item }}"
      register: mtu

    - name: match txload
      pattern_match:
        regex: "reliability.*txload\\s(\\S+),.*"
        content: "{{ item }}"
      register: txload

    - name: match rxload
      pattern_match:
        regex: "reliability.*rxload\\s(\\S+)"
        content: "{{ item }}"
      register: rxload
  loop: "{{ section }}"
  register: interfaces
  export: yes

```

{% endraw %}


Effectively, we are looping through the data of each interface and creating a pattern group (registering it as a variable called `interfaces`). Note the similarity of pattern using the `item` variable while working with loops in Ansible. Each of the `pattern_match` directives nested within the `pattern_group` is using a regex to match the data we wanted at the start.

On re-running the playbook with the updated parser, our debug output looks as follows:

``` bash
TASK [DISPLAY THE PARSED OUTPUT] **************************************************************************************************
ok: [rtr1] => {
    "interfaces": [
        {
            "mtu": {
                "matches": [
                    "1500"
                ]
            }, 
            "name": {
                "matches": [
                    "GigabitEthernet1"
                ]
            }, 
            "rxload": {
                "matches": [
                    "1/255"
                ]
            }, 
            "txload": {
                "matches": [
                    "1/255"
                ]
            }
        }, 
        {
            "mtu": {
                "matches": [
                    "1514"
                ]
            }, 
            "name": {
                "matches": [
                    "Loopback0"
                ]
            }, 
            "rxload": {
                "matches": [
                    "1/255"
                ]
            }, 
            "txload": {
                "matches": [
                    "1/255"
                ]
            }
        }, 
.
.
.
.
.
<output omitted for brevity>
```

### Creating a JSON output

Now that we have our regular expressions sorted out and we can see that the data we are interested in is being returned as a structured object, all that remains is to build a JSON object similar to how we did it in [Part1](https://termlen0.github.io/2018/06/26/observations/).

{% raw %}

``` yaml
- name: generate json data structure
  json_template:
    template:
      - key: "{{ item.name.matches.0 }}"
        object:
        - key: config
          object:
            - key: name
              value: "{{ item.name.matches.0 }}"
            - key: mtu
              value: "{{ item.mtu.matches.0 }}"
            - key: rxload
              value: "{{ item.rxload.matches.0 }}"
            - key: txload
              value: "{{ item.txload.matches.0 }}"
  loop: "{{ interfaces }}"
  export: yes
  register: interface_facts
```
{% endraw %}


Re-running the playbook and debugging `interface_facts` shows us the final, desired output.


``` bash
TASK [DISPLAY THE PARSED OUTPUT] **************************************************************************************************
ok: [rtr1] => {
    "interface_facts": [
        {
            "GigabitEthernet1": {
                "config": {
                    "mtu": 1500, 
                    "name": "GigabitEthernet1", 
                    "rxload": "1/255", 
                    "txload": "1/255"
                }
            }
        }, 
        {
            "Loopback0": {
                "config": {
                    "mtu": 1514, 
                    "name": "Loopback0", 
                    "rxload": "1/255", 
                    "txload": "1/255"
                }
            }
        }, 

```


This is why I think `command_parser` is such a powerful tool to parse complex show command output, using the same Ansible design patterns we are familiar with, without needing to learn yet another domain specific syntax or language. As network engineers we now have a tool that abstracts away the underlying complexity of programming.


Alright now on to some more `command_parser` goodness!.....

### Building customized "facts" without needing to write a module!

**NOTE: THE FOLLOWING WORKS _ONLY_ WITH THE DEVEL BRANCH OF THE NETWORK-ENGINE ROLE AT THE TIME OF THIS POST**

If you have used the network modules in Ansible you are probably familiar with the `*_facts` modules. Modules like `ios_facts`, `eos_facts`, `junos_facts` etc. While you can have limited control over the output from these modules (by using the `gather_subset` directive), you are by and large restricted to what the module creators consider **"facts"**.

In the following section, let's look at a powerful directive from the `command_parser` module called `extend` to learn how we can build our own customized facts parser - without needing to write a single line of code!!

We'll use the parsers we created in Part1 and the above one to create a single variable called `ios_interface_facts`. 

The basic changes needed to the parsers are as follows:

{% raw %}

###### show\_ip\_interfaces.yaml

``` yaml
  extend: ios_interface_facts

```

{% endraw %}


and

{% raw %}

###### show_interfaces.yaml

``` yaml
  export_as: dict
  extend: ios_interface_facts

```

{% endraw %}

> These directives are added to the `json_template` section of the parser. Please see the github  [repo](https://github.com/termlen0/parser_demo) for details. You will need to uncomment them to test.


With this in place, re-running the playbook and debugging the `ios_interfaces_facts` variable shows how the output of both the above parsers are made available in a single variable.

> Note: Run the playbook invoking the **demo_extend** tag if you are testing with the sample [repo](https://github.com/termlen0/parser_demo)


``` bash
TASK [DISPLAY THE GENERATED FACT] ****************************************************************************************************************
ok: [rtr1] => {
    "ios_interface_facts": {
        "interface_facts": {
            "GigabitEthernet1": {
                "config": {
                    "mtu": 1500, 
                    "name": "GigabitEthernet1", 
                    "rxload": "1/255", 
                    "txload": "1/255"
                }
            }, 
            "Loopback0": {
                "config": {
                    "mtu": 1514, 
                    "name": "Loopback0", 
                    "rxload": "1/255", 
                    "txload": "1/255"
                }
            }, 
            "Loopback1": {
.
.
.
.
.
<output omitted for brevity>
        "ip_interface_facts": [
            {
                "GigabitEthernet1": {
                    "data": {
                        "admin_state": "up", 
                        "ip": "172.16.230.103", 
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
                        "admin_state": "up", 
                        "ip": "10.1.1.101", 
.
.
.
.
.
<output omitted for brevity>
```

As you can see the `ios_interfaces_facts` is a nested object that contains 2 keys `interface_facts` - the parsed output of the _show interfaces_ command - and the `ip_interface_facts` - the parsed output of the _show ip interface brief_ command.

Hopefully this quick illustration of some of the powerful directives within command parser are helpful to you. I'd love to hear what your thoughts are on this approach and perceived ease/difficulty with this method of parsing device command output.
