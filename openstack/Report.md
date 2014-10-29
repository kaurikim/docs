
# 참고자료 

* Services Insertion Model in Quantum 
** https://wiki.openstack.org/wiki/Neutron/ServiceInsertion
* Notions of Service Type Framework
** https://wiki.openstack.org/wiki/Neutron/ServiceTypeFramework
* Neutron Service Agent
** https://wiki.openstack.org/wiki/Neutron/ServiceAgent

# 용어 정리 

* WSGI is the Web Server Gateway Interface. It is a specification for web servers and application servers to communicate with web applications.
* Paste Deployment is a system for finding and configuring WSGI applications and servers. The primary interaction with Paste Deploy is through its configuration files.
* Routes is a Python reimplementation of the Rails routes system for mapping URLs to application actions, and conversely to generate URLs. Routes makes it easy to create pretty and concise URLs that are RESTful with little effort.

# Neutron LBaaS 분석 


http://42.62.73.30/wordpress/wp-content/uploads/2013/08/nova-server-request.jpeg

https://wiki.openstack.org/w/images/e/e0/Havana_LBaaS_Diagram.jpg

https://wiki.openstack.org/wiki/File:Service_agent_deployment.png

# 아래 흐름을 파악 

https://wiki.openstack.org/w/images/7/77/Call_dispatching_workflow.png


## Neutron Server 


* core serivce plugin 지정 
<pre>
core_plugin = neutron.plugins.openvswitch.ovs_neutron_plugin.OVSNeutronPluginV2
</pre>
* "/etc/neutron.conf" 파일에 활성화할 Service Plugin 지정. 아래는 LB, FW 서비스 를 활성화한 경우이다. 
<pre>
service_plugins =neutron.services.loadbalancer.plugin.LoadBalancerPlugin,neutron.services.firewall.fwaas_plugin.FirewallPlugin
service_provider=LOADBALANCER:Haproxy:neutron.services.loadbalancer.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default
</pre>

** https://wiki.openstack.org/w/images/f/f7/Single-plugin.png

# config.parse(sys.argv[1:]) 2.neutron/common/config.py:load_paste_app(“neutron”)
##  neutron/auth.py:pipeline_factory()
###  neutron/api/v2/router.py:APIRouter.factory()
#### neutron/api/extensions.py: plugin_aware_extension_middleware_factory()
#### neutron.auth:NeutronKeystoneContext.factory() 
#### keystoneclient.middleware.auth_token:filter_factory()

## Sequence of Loading LBaaS Service Plugin 

* NeutronManager:_load_service_plugins()
    * service_plugins 에 명시된 LoadBalancerPlugin 인스턴스 생성  
    * LoadBalancerPlugin 인스턴스 생성 시점에 service_provider 에 명시된 HaproxyOnHostPluginDrive() 생성 

<pre><code>

class LoadBalancerPlugin(ldb.LoadBalancerPluginDb,
                         agent_scheduler.LbaasAgentSchedulerDbMixin):
    """Implementation of the Neutron Loadbalancer Service Plugin.
                                                                                                                       
    This class manages the workflow of LBaaS request/response.
    Most DB related works are implemented in class
    loadbalancer_db.LoadBalancerPluginDb.
    """
    supported_extension_aliases = ["lbaas",
                                   "lbaas_agent_scheduler",
                                   "service-type"]


    # lbaas agent notifiers to handle agent update operations;
    # can be updated by plugin drivers while loading;
    # will be extracted by neutron manager when loading service plugins;
    agent_notifiers = {}

    def __init__(self):
        """Initialization for the loadbalancer service plugin."""

        qdbapi.register_models()
        self.service_type_manager = st_db.ServiceTypeManager.get_instance()
        self._load_drivers()
        
    def _load_drivers(self):
        """Loads plugin-drivers specified in configuration."""
        self.drivers, self.default_provider = service_base.load_drivers(
            constants.LOADBALANCER, self)
        
        # we're at the point when extensions are not loaded yet
        # so prevent policy from being loaded 
        ctx = context.get_admin_context(load_admin_roles=False)
        # stop service in case provider was removed, but resources were not
        self._check_orphan_pool_associations(ctx, self.drivers.keys())
</code></pre>

### HaproxyOnHostPluginDriver

* http://docs.openstack.org/api/openstack-network/2.0/content/Sche_List_LBaaS_agents_Hosting_Pool.html
* https://devcentral.f5.com/articles/f5-big-ip-openstack-integration-available-for-download

<pre><code> 

class HaproxyOnHostPluginDriver(abstract_driver.LoadBalancerAbstractDriver):                                           
    
    def __init__(self, plugin):
        # TOPIC_PROCESS_ON_HOST='lbaas_process_on_host_agent'
        self.agent_rpc = LoadBalancerAgentApi(TOPIC_LOADBALANCER_AGENT)                                                
        self.callbacks = LoadBalancerCallbacks(plugin)                                                                 
        
        # AMQP Connection
        self.conn = rpc.create_connection(new=True)                                                                    
        self.conn.create_consumer(                                                                                     
            TOPIC_PROCESS_ON_HOST,
            self.callbacks.create_rpc_dispatcher(),                                                                    
            fanout=False)
        self.conn.consume_in_thread()                                                                                  
        self.plugin = plugin
        self.plugin.agent_notifiers.update(
            {q_const.AGENT_TYPE_LOADBALANCER: self.agent_rpc})                                                         
        
        # cfg.CONF.loadbalancer_pool_scheduler_driver = 
        # 'neutron.services.loadbalancer.agent_scheduler.ChanceScheduler'
        self.pool_scheduler = importutils.import_object(                                                               
            cfg.CONF.loadbalancer_pool_scheduler_driver)    
</code></pre>


<pre>
neutron.services.loadbalancer.agent_scheduler [-] Pool 1c1e7f91-13db-4ced-9e89-723dcb6ac55b is scheduled to lbaas agent fd026b51-9bb1-45ee-ae32-f64891747ea6
</pre>
