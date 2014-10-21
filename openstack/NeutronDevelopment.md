
# NeutronDevelopment

출처: https://wiki.openstack.org/wiki/NeutronDevelopment



### Userful Next Steps

* 기본적인 컴포넌들의 이해를 위해서 문서를 아래 읽어라: 
    * http://docs.openstack.org/trunk/openstack-network/admin/content/ch_overview.html
* 각 plugin 들이 반드시 구현해야 되는 API 를 이해하기 위해서 아래 문서를 읽어라 
    * http://docs.openstack.org/api/openstack-network/2.0/content/index.html
* Neutron code (http://launchpad.net/neutron) 를 구하고, ADMIN GUIDE 에 따라 실행하라 
* Check out the existing plugins in the neutron/plugins directory to get an idea of what a plugin looks like in practice
* 어떤 넘들이 사용되는지  neutron/plugins 디렉토리를 점검하라 

### Plugin FAQ

Q: Can I run multiple plugins at the same time?
A: No, only a single plugin can be run at a time, for a given Neutron API. That is because a "plugin" is really chunk of code that called to "implement" a particular API call. Just because there is one plugin, however, does not mean that you can only talk to one type of switch. One "plugin" can have multiple "drivers" that talk to different types of switches. For example, the Cisco plugin talks to multiple types of switches. There is no formal driver interface, but we encourage people to write the code that talks to a switch in a general way such that other plugins will be able to leverage it. A "driver" will usually be code that is able to talk to a particular switch model or family of switches. "drivers" usually will have methods for standard provisioning operations, like putting a port on a particular VLAN as the result of an attachment.
A. Yes, use the "meta-plugin" that calls code from two different existing plugins (https://github.com/openstack/neutron/tree/master/neutron/plugins/metaplugin)

## API Extensions

API Extensions allow a plugin to extend the Neutron API in order to expose 
more information. This information could be required to implement advanced 
functionality that is specific to a particular plugin, or to expose a 
capability before it has been incorporated into an official Neutron API.

We strongly believe that the right way to "propose" new functionality for an 
official API is to first expose it as an extension. Code is a very concrete 
definition, and writing the code and an implementation of that code often 
exposes important details that might not have come up if the discussion of an 
API was merely abstract.

* Creating Extensions:
    * Extension files should be placed in the extensions folder located at: neutron/extensions .
    * The extension file should have a class with the same name as the filename. This class should implement the contract required by the extension framework. See ExtensionDescriptor class in neutron/api/extensions.py for details.
    * To stop a file in the extensions folder from being loaded as an extension, the filename should start with an "_". For an example of an extension file look at Foxinsocks class in neutron/tests/unit/extensions/foxinsocks.py. The unit tests in neutron/tests/unit/test_extensions.py document all the ways in which you can use extensions
* Associating plugins with extensions:
    * A Plugin can advertise all the extensions it supports through the 'supported_extension_aliases' attribute. Eg:
```
class SomePlugin:
...
supported_extension_aliases = ['extension1_alias',
                             'extension2_alias',
                             'extension3_alias']
Any extension not in this list will not be loaded for the plugin
```

* Extension Interfaces for plugins (optional). The extension can mandate an interface that plugins have to support with the 'get_plugin_interface' method in the extension. For an example see the FoxInSocksPluginInterface in foxinsocks.py.


There are three types of extensions in Neutron:

* Resources introduce a new "entity" to the API. In Neutron, current entities are "networks" and "ports".
* Action extensions tend to be "verbs" that act on a resource. For example, in Nova, there is a "server" resource, and a "rebuild" action on that server resource: http://docs.openstack.org/cactus/openstack-compute/developer/openstack-compute-api-1.1/content/Rebuild_Server-d1e3538.html#d2278e1430 . Core Neutron API doesn't really use "actions" as part of the core API, but extensions certainly could.
* Request extensions allow one to add new values to existing requests objects. For example, if you were doing a POST to create a server, but wanted to add a new 'widgetType' attribute for the server object.

Looking at Foxinsox, along with the tests that use it can be helpful here: * neutron/tests/unit/extensions/foxinsocks.py * neutron/tests/unit/test_extensions.py

You can also look at other extensions in neutron/extensions, or even nova extensions in nova/api/openstack/compute/contrib/ (these are documented here: http://nova.openstack.org/api_ext/index.html)

Usually the best thing to do is find another extension that does something similar to what you are doing, then use it as a template.

## Developing in Linux

1. Make a virtual machine (however you do that on your OS) 1. Login to it. 1. If you have an HTTP proxy, export it as http_proxy - it makes virtual env recreation much easier. 1. Install development dependencies:
```
apt-get install python-dev python-virtualenv git
yum install python-devel python-virtualenv git
```
1. Run some tests:
```
./run_tests.sh
```
