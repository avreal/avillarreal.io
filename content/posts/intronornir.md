---
title: "An Introduction To Interacting With Nornir Output"
urltitle: "intronornir"
date: 2020-06-19T15:10:27-07:00
draft: false
showdate: true
---

Over the past year I've taken an interest in network automation tools and how I can apply them to my work to make tasks easier. There are low level libraries such as {{< rawhtml >}}<a href="http://www.paramiko.org/" target = "_blank"><strong>Paramiko</strong></a>{{< /rawhtml >}}, a python SSH implementation, that allow you to programmatically access your devices but require lots of overhead and additional configuration. We can go up one level of abstraction to libraries such as {{< rawhtml >}}<a href="https://github.com/ktbyers/netmiko" target = "_blank"><strong>Netmiko</strong></a>{{< /rawhtml >}} and{{< rawhtml >}}<a href="https://napalm-automation.net/" target = "_blank"><strong>NAPALM</strong></a>{{< /rawhtml >}}. These leverage connection libraries like Paramiko to manage connections to devices and interact with them. You can accomplish a lot with Netmiko/NAPALM, but you can go one step further with something like {{< rawhtml >}}<a href="https://www.ansible.com/" target = "_blank"><strong>Ansible</strong></a> {{< /rawhtml >}} to orchestrate your network and infrastructure. Ansible is widely used and has huge community support, but there are a few minor annoyances. Ansible playbooks are written in a domain specific language which makes logic complicated, it doesn't scale very well, and can be tricky to debug. I believe network engineers can benefit greatly from learning {{< rawhtml >}}<a href="https://nornir.readthedocs.io/en/latest/" target = "_blank"><strong>Nornir</strong></a>{{< /rawhtml >}}. Nornir is an automation framework that is functionally very similar to Ansible, but in pure python. Because it is pure python, it is much easier to debug and extend the functionality. It has threading built in and is highly scalable. In this post I'll go over some basics of the Nornir framework and show the output classes that you will interact with when using it. <!--more--> 


## Installation

Installation is very simple, use pip as follows and confirm you can import the package:

~~~
$pip3 install nornir

$python3
>>>import nornir.core
>>>
~~~

## Getting Started

The following was originally written on my github to teach coworkers about Nornir, I thought it would make a good post so I am copying it here with some minor changes. Check out my github{{< rawhtml >}}<a href="https://github.com/avreal/NetworkAutomationScenarios/tree/master/NornirBasics" target = "_blank"><strong>repo</strong></a> {{< /rawhtml >}} to see the config file, hosts, and other Nornir scenarios. This isn't meant to be a tutorial from scratch, I would recommend reading the {{< rawhtml >}}<a href="https://nornir.readthedocs.io/en/latest/tutorials/intro/index.html" target ="_blank"><strong>docs</strong></a> {{< /rawhtml >}} but this should serve as a good reference for interacting with Nornir objects and the results they generate.



First we instantiate the InitNornir class, here we can specify a config file or pass the contents as parameters. The config file specifies the location of my hosts, groups, and defaults files (among other parameters). We then use the run function to execute the task over the hosts in the inventory. In this case I am using get_facts from the NAPALM library.

~~~python
from nornir import InitNornir
from nornir.plugins.tasks.networking import napalm_get
from nornir.plugins.functions.text import print_result
import json

nr = InitNornir(config_file="./config.yaml")
task_result = nr.run(task=napalm_get,getters=["facts"])
~~~

Lets look at the output of type() for these 2 items.

~~~python
>>> type(nr)
<class 'nornir.core.Nornir'>
>>>
>>> type(task_result)
nornir.core.task.AggregatedResult
>>>
~~~

We will mostly be working with the AggregatedResult object, but there are some useful methods on the nornir.core.Nornir object shown below. These can be used to access the inventory dictionary generated during instantiation. I highly recommend exploring these methods further while experimenting with Nornir.

~~~
>>> type(nr.inventory)
<class 'nornir.core.inventory.Inventory'>

>>> dir(nr.inventory)
['__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', 
'__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', 
'__len__', '__lt__', '__module__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', 
'__repr__', '__setattr__', '__sizeof__', '__slots__', '__str__', '__subclasshook__', 
'_update_group_refs', 'add_group', 'add_host', 'children_of_group', 'defaults', 'dict', 
'filter', 'get_defaults_dict', 'get_groups_dict', 'get_hosts_dict', 'get_inventory_dict', 
'groups', 'hosts']
>>> 

>>> nr.inventory.hosts
{'Router1': Host: Router1, 'Router2': Host: Router2, 'Router3': Host: Router3, 'Router4': Host: Router4}

>>> nr.inventory.get_inventory_dict()
{'hosts': {'Router1': {'hostname': '192.168.100.10', 'port': 22, 'username': None, 'password': None, 'platform': None, 'groups': ['cisco_ios', 'r1r2'], 'data': {}, 'connection_options': {}}, 'Router2': {'hostname': '192.168.100.20', 'port': 22, 'username': None, 'password': None, 'platform': None, 'groups': ['cisco_ios', 'r1r2'], 'data': {}, 'connection_options': {}}, 'Router3': {'hostname': '192.168.100.30', 'port': 22, 'username': None, 'password': None, 'platform': None, 'groups': ['cisco_ios', 'r3r4'], 'data': {}, 'connection_options': {}}, 'Router4': {'hostname': '192.168.100.40', 'port': 22, 'username': None, 'password': None, 'platform': None, 'groups': ['cisco_ios', 'r3r4'], 'data': {}, 'connection_options': {}}}, 'groups': {'global': {'hostname': None, 'port': None, 'username': None, 'password': None, 'platform': None, 'groups': [], 'data': {'domain': 'global.local'}, 'connection_options': {}}, 'r1r2': {'hostname': None, 'port': None, 'username': None, 'password': None, 'platform': None, 'groups': ['cisco_ios', 'global'], 'data': {'asn': 1}, 'connection_options': {}}, 'r3r4': {'hostname': None, 'port': None, 'username': None, 'password': None, 'platform': None, 'groups': ['cisco_ios', 'global'], 'data': {'asn': 1}, 'connection_options': {}}, 'cisco_ios': {'hostname': None, 'port': None, 'username': None, 'password': None, 'platform': 'ios', 'groups': [], 'data': {}, 'connection_options': {}}}, 'defaults': {'hostname': None, 'port': None, 'username': 'admin', 'password': 'admin', 'platform': None, 'data': {}, 'connection_options': {}}}
>>> 
~~~

Suppose we wish to grab certain keys/values from the task result output for a particular device, we can dig further into the AggregatedResult object that was returned from nr.run. First lets do a normal print(task_result):

~~~python
>>> print(task_result)
AggregatedResult (napalm_get): {'Router1': MultiResult: [Result: "napalm_get"], 'Router2': MultiResult: [Result: "napalm_get"], 'Router3': MultiResult: [Result: "napalm_get"], 'Router4': MultiResult: [Result: "napalm_get"]}
~~~

Here we see that the AggregatedResult object is a dict-like object, we can access the individual MultiResult objects by doing task_result['*hostname*'].

~~~python
>>> task_result['Router1']
MultiResult: [Result: "napalm_get"]
~~~

Now we have a list-like object, in order to extract data from this we can call the .result method as follows:

~~~python
>>> task_result['Router1'].result
{'facts': {'uptime': 9600, 'vendor': 'Cisco', 'os_version': 'Linux Software (I86BI_LINUX-ADVENTERPRISEK9-M), Version 15.4(1)T, DEVELOPMENT TEST SOFTWARE', 'serial_number': '2048001', 'model': 'Unknown', 'hostname': 'ABC-001-0001', 'fqdn': 'ABC-001-0001.lab.com', 'interface_list': ['Ethernet0/0', 'Ethernet0/1', 'Ethernet0/2', 'Ethernet0/3', 'Ethernet1/0', 'Ethernet1/1', 'Ethernet1/2', 'Loopback0']}}
>>> 
~~~

Now we have a standard dictionary whose data we can access by using its key(s). See below:

~~~python
>>> output = task_result['Router1'].result
>>>
>>> output['facts']
{'uptime': 9600, 'vendor': 'Cisco', 'os_version': 'Linux Software (I86BI_LINUX-ADVENTERPRISEK9-M), Version 15.4(1)T, DEVELOPMENT TEST SOFTWARE', 'serial_number': '2048001', 'model': 'Unknown', 'hostname': 'ABC-001-0001', 'fqdn': 'ABC-001-0001.lab.com', 'interface_list': ['Ethernet0/0', 'Ethernet0/1', 'Ethernet0/2', 'Ethernet0/3', 'Ethernet1/0', 'Ethernet1/1', 'Ethernet1/2', 'Ethernet1/3','Loopback0']}
>>>
>>> output['facts']['uptime']
9600
~~~

Given all of the above, we can now iterate through the Aggregated result object with a nested loop to generate an output with the device hostname and all of the data gathered in the MultiResult object. There are several ways to achieve a similar result.

In this first loop, we print out the hostname as a string then follow it with a nested dictionary where we can then access keys as needed.

~~~python
>>> for host in task_result:
...     print(host)
...     for results in task_result[host]:
...             print(results)
... 
Router1
{'facts': {'uptime': 9600, 'vendor': 'Cisco', 'os_version': 'Linux Software (I86BI_LINUX-ADVENTERPRISEK9-M), Version 15.4(1)T, DEVELOPMENT TEST SOFTWARE', 'serial_number': '2048001', 'model': 'Unknown', 'hostname': 'ABC-001-0001', 'fqdn': 'ABC-001-0001.lab.com', 'interface_list': ['Ethernet0/0', 'Ethernet0/1', 'Ethernet0/2', 'Ethernet0/3', 'Ethernet1/0', 'Ethernet1/1', 'Ethernet1/2', 'Ethernet1/3', 'Loopback0']}}
Router2

...omitted...

Router3

...omitted...

Router4

...omitted...

~~~

In this example, we use the items() method (recall that AggregatedResult is a dict-like object) and then iterate through the dict_items object we obtain. Since this contains tuples we use an index to access the objects.

~~~python
>>> resultsitems = task_result.items()
>>> for hosts in resultsitems:
...     print(hosts[0])
...     print(hosts[1].result)
... 
Router1
{'facts': {'uptime': 9600, 'vendor': 'Cisco', 'os_version': 'Linux Software (I86BI_LINUX-ADVENTERPRISEK9-M), Version 15.4(1)T, DEVELOPMENT TEST SOFTWARE', 'serial_number': '2048001', 'model': 'Unknown', 'hostname': 'ABC-001-0001', 'fqdn': 'ABC-001-0001.lab.com', 'interface_list': ['Ethernet0/0', 'Ethernet0/1', 'Ethernet0/2', 'Ethernet0/3', 'Ethernet1/0', 'Ethernet1/1', 'Ethernet1/2', 'Ethernet1/3', 'Loopback0']}}
Router2

...omitted...

Router3

...omitted...

Router4

...omitted...

>>> 
~~~

In this example, we use a similar loop to the first example but format the output with json.

~~~python
import json

for hosts in task_result:
    print(hosts)
    for results in task_result[hosts]:
        print(json.dumps(results.result, indent=4))  

Router1
{
    "facts": {
        "uptime": 9600,
        "vendor": "Cisco",
        "os_version": "Linux Software (I86BI_LINUX-ADVENTERPRISEK9-M), Version 15.4(1)T, DEVELOPMENT TEST SOFTWARE",
        "serial_number": "2048001",
        "model": "Unknown",
        "hostname": "ABC-001-0001",
        "fqdn": "ABC-001-0001.lab.com",
        "interface_list": [
            "Ethernet0/0",
            "Ethernet0/1",
            "Ethernet0/2",
            "Ethernet0/3",
            "Ethernet1/0",
            "Ethernet1/1",
            "Ethernet1/2",
            "Ethernet1/3",
            "Loopback0"
        ]
    }
}
Router2

...omitted...

Router3

...omitted...

Router4

...omitted...

~~~

However nornir also includes a function for neatly printing the AggregatedResult, this is the print_result() function. This is great for configuration management and backup as it requires little effort to use but is very readable.

~~~python
>>> print_result(task_result)
napalm_get**********************************************************************
* Router1 ** changed : False ***************************************************
vvvv napalm_get ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
{ 'facts': { 'fqdn': 'ABC-001-0001.lab.com',
             'hostname': 'ABC-001-0001',
             'interface_list': [ 'Ethernet0/0',
                                 'Ethernet0/1',
                                 'Ethernet0/2',
                                 'Ethernet0/3',
                                 'Ethernet1/0',
                                 'Ethernet1/1',
                                 'Ethernet1/2',
                                 'Ethernet1/3',
                                 'Loopback0'],
             'model': 'Unknown',
             'os_version': 'Linux Software (I86BI_LINUX-ADVENTERPRISEK9-M), '
                           'Version 15.4(1)T, DEVELOPMENT TEST SOFTWARE',
             'serial_number': '2048001',
             'uptime': 9600,
             'vendor': 'Cisco'}}
^^^^ END napalm_get ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* Router2 ** changed : False ***************************************************
vvvv napalm_get ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO

...omitted...

^^^^ END napalm_get ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* Router3 ** changed : False ***************************************************
vvvv napalm_get ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO

...omitted...

^^^^ END napalm_get ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* Router4 ** changed : False ***************************************************
vvvv napalm_get ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO

...omitted...

^^^^ END napalm_get ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
~~~

# Conclusion

Nornir is an incredibly useful tool with tons of potential and this post barely scratches the surface of its capabilities. The value in using a tool like Nornir comes from what you can do with the data that is output from your tasks. The community isn't very large yet but with time I believe Nornir can become a contender to Ansible for network engineers looking to automate in their environment. Future posts will go over more Nornir basics and a look at real world scenarios using Nornir.


### Further Reading and Watching

{{< rawhtml >}}<a href="https://readthedocs.org/projects/nornir/" target = "_blank"><strong>Nornir ReadTheDocs</strong></a> {{< /rawhtml >}} <br/>

{{< rawhtml >}}<a href="https://github.com/nornir-automation/nornir" target = "_blank"><strong>Nornir Repo</strong></a> {{< /rawhtml >}}  <br/>

{{< rawhtml >}}<a href="https://www.youtube.com/watch?v=5SS5tTgacKg" target = "_blank"><strong>Cisco Live Presentation</strong></a> {{< /rawhtml >}}  <br/>

{{< rawhtml >}}<a href="https://www.youtube.com/watch?v=H7qs-jg6kVg" target = "_blank"><strong>Dmitry Figol Nornir Livestream</strong></a> {{< /rawhtml >}}  <br/>


