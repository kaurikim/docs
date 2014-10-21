
# NeutronDevelopment

출처: https://wiki.openstack.org/wiki/NeutronDevelopment


## Before you Jump into the Code

Before developing a plugin yourself, make sure you know how OpenStack Networking (Neutron [formerly Quantum]) works by reading the OpenStack Networking Admin Guide and asking questions.

Then try to run it yourself using devstack . One example of how to run OpenStack Networking with devstack is here: NeutronDevstack

Subscribe to the openstack-dev email list. . Email sent to this list about OpenStack Network should have a subject line that starts with "[Neutron]".

You'll also want to take care of a few OpenStack-wide requirements, described in HowToContribute . In particular:

signing a Contributors License Agreement (http://wiki.openstack.org/CLA)
getting a launchpad account are very important (http://launchpad.net)

If you're feeling adventurous, you can sign up to see the set of bugs + changes flowing into Neutron:

subscribe to neutron bug mail: https://bugs.launchpad.net/neutron/+subscribe
sign up to see changes proposed to the openstack/neutron and openstack/python-neutron projects: https://review.openstack.org/#/settings/projects
Before trying to submit your first patch, please learn more about how the patch submission and review process works (GerritWorkflow) and read the NeutronHACKING document
