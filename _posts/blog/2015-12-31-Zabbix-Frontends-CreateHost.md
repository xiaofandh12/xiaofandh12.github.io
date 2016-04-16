---
layout: post
title: Zabbix前端源码分析之创建Host
description: Zabbix Frontends,Create Host
category: blog
---

## 前端操作介绍

创建Host步骤：选择Configuration -> Hosts,点击Create host。

![Zabbix Frontends Create Host](/images/2015-12-31-Zabbix-Frontends-CreateHost/ZabbixFrontendsCreateHost.png)

点击完Create host后，出现创建Host的界面，在此界面需要填写Host、Templates、IPMI、Macros、Host inventory的信息。

![Zabbix Frontends Create Host 1](/images/2015-12-31-Zabbix-Frontends-CreateHost/ZabbixFrontendsCreateHost1.png)

## 辅助文件

### API.php

我们可以调用API.php中定义的相关函数，得到对应的类，再使用此类中的方法来处理请求(A map of classes that should handle the corresponding API objects requests)。

* API.php

    ```
    class API {
        ...
        /**
         *A map of classes that should handle the corresponding API objects requests.
         *@var array
         */
        protected static $classMap = array(
            'action' => 'CAction',
            'alert' => 'CAlert',
            'apiinfo' => 'CAPIInfo',
            'application' => 'CApplication',
            'configuration' => 'CConfiguration',
            'dcheck' => 'CDCheck',
            'dhost' => 'CDHost',
            'discoveryrule' => 'CDiscoveryRule',
            'drule' => 'CDRule',
            'dservice' => 'CDService',
            'event' => 'CEvent',
            'graph' => 'CGraph',
            'graphitem' => 'CGraphItem',
            'graphprototype' => 'CGraphPrototype',
            'host' => 'CHost',
            'hostgroup' => 'CHostGroup',
            'hostinterface' => 'CHostInterface',
            'history' => 'CHistory',
            'hostinterface' => 'CHostInterface',
            'image' => 'CImage',
            'iconmap' => 'CIconMap',
            'item' => 'CItem',
            'itemprototype' => 'CItemPrototype',
            'maintenance' => 'CMaintenance',
            'map' => 'CMap',
            'mediatype' => 'CMediatype',
            'proxy' => 'CProxy',
            'service' => 'CService',
            'screen' => 'CScreen',
            'screenitem' => 'CScreenItem',
            'script' => 'CScript',
            'template' => 'CTemplate',
            'templatescreen' => 'CTemplateScreen',
            'templatescreenitem' => 'CTemplateScreenItem',
            'trigger' => 'CTrigger',
            'triggerprototype' => 'CTriggerPrototype',
            'user' => 'CUser',
            'usergroup' => 'CUserGroup',
            'usermacro' => 'CUserMacro',
            'usermedia' => 'CUserMedia',
            'httptest' => 'CHttpTest',
            'webcheck' => 'CHttpTest'
        );

        ...

        /**
         * @return CDiscoveryRule
         */
        //如通过API::DiscoveryRule()即可得到类CDiscoveryRule，API中其它的函数有类似的作用。
        public static function DiscoveryRule(){
            return self::getObject('discoveryrule');
        }

        ...

        /**
         * @return CGraph
         */
        public static function Graph(){
            return self::getObject('graph');
        }

        /**
         * @return CGraphItem
         */
        public static function GraphItem(){
            return self::getObject('graphitem');
        }

        /**
         * @return CGraphPrototype
         */
        public static function GraphPrototype(){
            return self::getObject('graphprototype');
        }

        ...

        /**
         * @return CHost
         */
        public static function Host(){
            return self::getObject('host');
        }

        /**
         * @return CHostPrototype
         */
        public static function HostPrototype(){
            return self::getObject('hostprototype');
        }

        /**
         * @return CHostGroup
         */
        public static function HostGroup(){
            return self::getObject('hostgroup');
        }

        /**
         * @return CHostInterface
         */
        public static function Host(){
            return self::getObject('host');
        }

        ...

        /**
         * @return CItem
         */
        public static function Item(){
            return self::getObject('item');
        }

        /**
         * @return CItemPrototype
         */
        public static function ItemPrototype(){
            return self::getObject('itemprototype');
        }

        ...

        /**
         * @return CTrigger
         */
        public static function Trigger(){
            return self::getObject('trigger');
        }

        /**
         * @return CTriggerPrototype
         */
        public static function TriggerPrototype(){
            return self::getObject('triggerprototype');
        }

        ...

        /**
         * @return CUserMacro
         */
        public static function UserMacro(){
            return self::getObject('usermacro');
        }

        ...
    ```

### Manager.php

我们可以通过调用Manager.php得到数据库管理对象，利用数据库管理对象可以创建可被存储的实例(A class for creating a storing instances of DB objects managers)。

* Manager.php

    ```
    /**
     * A class for creating a storing instances of DB objects managers.
     */
    class Manager extends CFactoryRegistry {
        public static function getInstance($class = __CLASS__){
            return parent::getInstance($class);
        }

        /**
         * @return CApplicationManager
         */
        public static function Application(){
            return self::getInstance()->getObject('CApplicationManager');
        }

        /**
         * @return CHistoryManager
         */
        public static function History(){
            return self::getInstance()->getObject('CHistoryManager');
        }

        /**
         * @return CHttpTestManager
         */
        public static function HttpTest(){
            return self::getInstance()->getObject('CHttpTestManager');
        }
    }
    ```
### DB.php

DB.php中提供了可以向数据库中做增删改查的函数。

## 主流程

### Hosts.php

Hosts.php先从前端获取数据，做一些初始化工作，然后调用```API::Host->create($host)```来创建Host。

* Hosts.php

    ```
    ...
    else {
        $macros = get_request('macros', array());
        $interfaces = get_request('interfaces', array());
        $templates = get_request('templates', array());
        $groups = get_request('groups', array());

        $linkedTemplates = $templates;
        $templates = array();
        foreach ($linkedTemplates as $templateId) {
            $templates[] = array('templateid' => $templateId);
        }

        foreach ($interfaces as $key => $interface) {
            if (zbx_empty($interface['ip']) && zbx_empty($interface['dns'])) {
                unset($interface[$key]);
                continue;
            }

            if ($interface['isNew']) {
                unset($interfaces['$key]['interfaceid']);
            }
            unset($interfaces[$key]['isNew']);
            $interfaces[$key]['main'] = 0;
        }

        $interfaceTypes = array(INTERFACE_TYPE_AGENT, ITERFACE_TYPE_SNMP, INTERFACE_TYPE_JMX, INTERFACE_TYPE_IPMI);
        foreach ($interfaceTypes as $type) {
            if (isset($_REQUEST['mainInterfaces'][$type])) {
                $interfaces[$_REQUEST['mainInterfaces'[$type]]['main'] = '1';
            }
        }

        // ignore empty new macros, i.e., macros rows that have not been filled
        foreach ($macros as $key => $macro) {
            if (!isset($macro['hostmacroid']) && zbx_empty($macro['macro']) && zbx_empty($macro['value'])) {
                unset($macros[$key]);
            }
        }

        foreach ($macros as $key => $macro) {
            //transform macros to uppercase {$aaa} => {$AAA}
            $macros[$key]['macro'] = zbx_strtoupper($macro['macro']);
        }

        //create new group
        if (!zbx_empty($_REQUEST['newgroup'])) {
            if (!$newGroup = API::HostGroup()->create(array('name' => $_REQUEST['newgroup']))) {
                throw new Exception();
            }

            $groups[] = reset($newGroup['groupids']);
        }

        $groups = zbx_toObject($groups, 'groupid');

        $host = array(
            'host' => $_REQUEST['host'],
            'name' => $_REQUEST['visiblename'],
            'status' => $_REQUEST['status'],
            'proxy_hostid' => get_request('proxy_hostid', 0),
            'ipmi_authtype' => get_request('ipmi_authtype'),
            'ipmi_privilege' => get_request('ipmi_privilege'),
            'ipmi_username' => get_request('ipmi_username'),
            'ipmi_password' => get_request('ipmi_password'),
            'groups' => $groups,
            'templates' => $templates,
            'interfaces' => $interfaces,
            'macros' => $macros,
            'inventory' => (get_request('inventory_mode') != HOST_INVENTORY_DISABLED) ? get_request('host_inventory', array()) : null,
            'inventory_mode' => get_request('inventory_mode')
        );

        if (!$createNew) {
            $host['templates_clear'] = zbx_toObject(get_request('clear_tmplates', array()), 'templateid');
        }
    }
    if ($createNew) {
        // 通过API.php的类映射功能，调用CHost.php中类CHost的create方法，向create方法中传入的参数为$host。
        $hostIds = API::Host()->create($host);

        if ($hostIds) {
            $hostId = reset($hostIds['hostids']);
        }
        else {
            throw new Exception();
        }

        add_audit_ext(AUDIT_ACTION_ADD, AUDIT_RESOURCE_HOST, $hostId, $host['host'], null, null, null);
    }
    ...
    ```

#### CHost.php

在Hosts.php中通过调用```API::Host->create($host)```来创建Host，通过API.php的映射，最终会调用CHost.php中定义的```CHost->create($host)```来创建Host。

在```CHost->create($host)```中，先调用```DB::insert('hosts_groups', $groupsToAdd)```往hosts_groups表中插入数据；然后通过```$result = API::Host()->massAdd($options)```往相应表中插入数据，关于这个函数会在下面做进一步讲解；最后调用```DB::insert('host_inventory', array($hostInventory), false)```往host_inventory中插入数据。

* CHost.php: CHost->create($host)

    ```
    /**
     * Add Host
     * 
     * @param array  $hosts multidimensional array with Hosts data
     * @param string $hosts ['host'] Host name.
     * @param array  $hosts ['groups'] array of HostGroup objects with IDs add Host to.
     * @param int    $hosts ['port'] Port. OPTIONAL
     * @param int    $hosts ['status'] Host Status. OPTIONAL
     * @param int    $hosts ['useip'] Use IP. OPTIONAL
     * @param string $hosts ['dns'] DNS. OPTIONAL
     * @param string $hosts ['ip'] IP. OPTIONAL
     * @param int    $hosts ['proxy_hostid'] Proxy Host ID. OPTIONAL
     * @param int    $hosts ['ipmi_authtype'] IPMI authentication type. OPTIONAL
     * @param int    $hosts ['ipmi_privilege'] IPMI privilege. OPTIONAL 
     * @param string $hosts ['ipmi_username'] IPMI username. OPTIONAL
     * @param string $hosts ['ipmi_password'] IPMI password. OPTIONAL
     * 
     * @return boolean
     */
    public function create($hosts) {
        $hosts = zbx_toArray($hosts);
        $hostids = array();

        $this->checkInput($hosts, __FUNCTION__);

        foreach ($hosts as $host) {
            $hostid = DB::insert('hosts', array($host));
            $hostids[] = $hostid = reset($hostid);

            $host['hostid'] = $hostid;

            // save groups
            // groups must be added before calling massAdd() for permission validation to work
            $groupsToAdd = array();
            foreach ($host['groups'] as $group) {
                $groupsToAdd[] = array(
                    'hostid' => $hostid,
                    'groupid' => $group['groupid']
                );
            }
            //调用DB.php中类DB的insert函数，往hosts_groups表格中插入数据
            DB::insert('hosts_groups', $groupsToAdd);

            //初始化$options，并为其赋值
            $options = array();
            $options['hosts'] = $host;

            if (isset($host['templates']) && !is_null($host['templates'])) {
                $options['templates'] = $host['templates'];
            }

            if (isset($host['macros'] && !is_null($host['macros'])) {
                $options['macros'] = $host['macros'];
            }

            if (isset($host['interfaces']) && !is_null($host['interfaces'])) {
                $options['interfaces'] = $host['interfaces'];
            }

            //调用CHost.php中类CHost的massAdd函数，传入的参数为$options
            $result = API::Host()->massAdd($options);
            if (!$result) {
                self::exception();
            }

            if (!empty($host['inventory'])) {
                $hostInventory = $host['inventory'];
                $hostInventory['hostid'] = $hostid;
                $hostInventory['inventory_mode'] = isset($host['inventory_mode'])
                $hostInventory['inventory_mode'] = isset($host['inventory_mode'])?$host['inventory_mode']:HOST_INVENTORY_MANUAL;
                //调用DB.php中类DB的insert函数，向host_inventory表格中插入数据
                DB::insert('host_inventory', array($hostInventory), false);
            }
        }

        return array('hostids' => $hostids);
    }
    ```

```CHost->massAdd(array $data)```先使用```$this->isWritable($hostIds)```来检查权限；接下来再通过```API::HostInterface()->massAdd()```向interface表中插入数据；最后调用```parent::massAdd($data)```往数据库相应表中插入数据，这里的```parent```是指CHost类的父类，即CHostGeneral。

* CHost.php: CHost->massAdd(array $data)
    
    ```
    /**
     * Additionally allows to create new interfaces on hosts.
     * 
     * Checks write permissions for hosts.
     * 
     * Additional supported $data parameters are:
     * - interfaces - an array of interfaces to create on the hosts
     * - templates  - an array of templates to link to the hosts, overrides the CHostGeneral::massAdd() 'templates' parameter
     *
     * @param array $data
     * 
     * @return array
     */
     public function massAdd(array $data) {
         $hosts = isset($data['hosts']) ? zbx_toArray($data['hosts']) : array();
         $hostIds = zbx_objectValues($hosts, 'hostid');

         // check permissions
         if (!$this->isWritable($hostIds)) {
             self::exception(ZBX_API_ERROR_PERMISSIONS, _('You do not have permission to perform this operation.'));
         }

         // add new interfaces
         if (!empty($data['interfaces])) {
             API::HostInterface()->massAdd(array(
                'hosts' => $data['hosts'],
                'interfaces' => zbx_toArray($data['interfaces'])
            ));
         }

         // rename the "templates" parameter to the common "templates_link"
         if (isset($data['templates'])) {
             $data['templates_link'] = $data['templates'];
             unset($data['templates']);
         }

         $data['templates'] = array();

         //类CHost的父类为CHostGeneral，这里是在调用父类CHostGeneral的massAdd函数，传入的参数为$data，类CHostGeneral定义于文件CHostGeneral.php中   
         return parent::massAdd($data);
     }
    ```
#### CHostGeneral.php

CHostGeneral.php中讲到类CHostGeneral有三个作用：add hosts to groups, link templates to hosts, add new macros to hosts.

add hosts to groups通过调用```API::HostGroup()->massAdd()```实现；
link templates to hosts通过调用```$this->link()```实现；
add new macros to hosts通过调用```API::UserMacro()->create($hostMacrosToAdd)```来实现。

* CHostGeneral.php: CHostGeneral->massAdd(array $data)

    ```
    /**
     * Allows to:
     * - add hosts to groups;
     * - link templates to hosts;
     * - add new macros to hosts;
     * 
     * Supported $data parameters are:
     * - hosts          - an array of hosts to be updated
     * - templates      - an array of templates to be updated
     * - groups         - an array of host groups to add the host to
     * - templates_link - an array of templates to link to the hosts
     * - macros         - an array of macros to create on the host
     *
     * @param array $data
     *
     * @return array
     */
    public function massAdd(array $data) {
        $hostIds = zbx_objectValues($data['hosts'], 'hostid');
        $templateIds = zbx_objectValues($data['templates'], 'templateid');

        $allHostIds = array_merge($hostIds, $templateIds);

        // add groups
        if (!empty($data['groups'])){
            API::HostGroup()->massAdd($options = array(
                'hosts' => $data['hosts'],
                'templates' => $data['templates'],
                'groups' => $data['groups']
            ));
        }

        // link templates
        if (!empty($data['templates_link'])){
            if (!API::Host()->isWritable($allHostIds)){
                self::exception(ZBX_API_ERROR_PERMISSIONS,
                    _('No permissions to referred object or it does not exist!')
                );
            }
            
            //调用类CHostGeneral的方法link
            $this->link(zbx_objectValues(zbx_toArray($data['templates_link']), 'templateid'), $allHostIds);
        }

        // create macros
        if (!empty($data['macros'])) {
            $data['macros'] = zbx_toArray($data['macros']);

            $hostMacrosToAdd = array();
            foreach ($data['macros'] as $hostMacro) {
                foreach ($allHostIds as $hostid) {
                    $hostMacro['hostid'] = $hostid;
                    $hostMacrosToAdd[] = $hostMacro;
                }
            }

            API::UserMacro()->create($hostMacrosToAdd);
        }
        $ids = array('hostids' => $hostIds, 'templateids' => $templateIds);
        return array($this->pkOption() => $ids[$this->pkOption()]);
    }
    ```

```CHostGenreal->link```的作用如前所属就是link templates to hosts，它调用了很多类中的方法来实现，从这里到本文结束都是在讲```CHostGeneral->link```中调用的各方法的实现，因其太多不一一解释，仅列出源码，我们要记住一点的就是它们总的来说是在实现"link templates to hosts"这个目标。

* CHostGeneral.php: CHostGeneral->link

    ```
    protected function link(array $templateIds, array $targetIds) {
        //类CHostGeneral的父类为CHostBase，这里调用的是类CHostBase的link方法，类CHostBase定义于文件CHostBase.php中
        $hostLinkageInserts = parent::link($templateIds, $targetIds);

        foreach ($hostsLinkageInserts as $hostTplIds) {
            Manager::Application()->link($hostTplIds['templateid'],$hostTplIds['hostid']);

            API::DiscoveryRule()->syncTemplates(array(
                'hostids' => $hostTplIds['hostid'],
                'templateids' => $hostTplIds['templateid']
            ));

            API::Itemprototype()->syncTemplates(array(
                'hostids' => $hostTplIds['hostid'],
                'templateids' => $hostTplIds['templateid']
            ));

            API::HostPrototype()->syncTemplates(array(
                'hostids' => $hostTplIds['hostid'],
                'templateids' => $hostTplIds['templateid']
            ));

            API::Item()->syncTemplates(array(
                'hostids' => $hostTplIds['hostid'],
                'templateids' => $hostTplIds['templateid']
            ));

            Manager::HttpTest()->link($hostTplIds['templateid'], $hostTplIds['hostid']);
        }

        // we do linkage in two separate loops because for triggers you need all items already created on host
        foreach ($hostsLinkageInserts as $hostTplIds) {
            API::Trigger()->syncTemplates(array(
                'hostids' => $hostTplIds['hostid'],
                'templateids' => $hostTplIds['templateid']
            ));

            API::TriggerPrototype()->syncTemplates(array(
                'hostids' => $hostTplIds['hostid'],
                'templateids' => $hostTplIds['templateid']
            ));

            API::GraphPrototype()->syncTemplates(array(
                'hostids' => $hostTplIds['hostid'],
                'templateids' => $hostTplIds['templateid']
            ));

            API::Graph()->syncTemplates(array(
                'hostids' => $hostTplIds['hostid'],
                'templateids' => $hostTplIds['templateid']
            ));
        }

        foreach ($hostsLinkageInserts as $hostTplIds){
            API::Trigger()->syncTemplateDependencies(array(
                'templateids' => $hostTplIds['templateid'],
                'hostids' => $hostTplIds['hostid']
            ));
        }

        return $hostsLinkageInserts;
    }
    ```

###CHostGeneral->link()方法

#### CHostBase.php

* CHostBase.php: CHostBase->link

    ```
    /**
     * Links the templates to the given hosts.
     *
     * @param array $templateIds
     * @param array $targetIds  an array of host IDs to link the templates to
     *
     * @return array    an array of added hosts_templates rows, with 'hostid' and 'templateid' set for each row
     */
    protected function link(array $templateIds, array $targetIds) {
        if (empty($templateIds)) {
            return;
        }

        // permission check
        if (!API::Template()->isReadable($templateIds)) {
            self::exception(ZBX_API_ERROR_PERMISSIONS, _('No permissions to referred object or it does not exist!'));
        }

        // check if someone passed duplicate templates in the same query
        $templateIdDuplicates = zbx_arrayFindDuplicates($templateIds);
        if (!zbx_empty($templateIdDuplicates)) {
            $duplicatesFound = array();
            foreach ($templateIdDuplicates as $value => $count) {
                $duplicatesFound[] = _s('template ID "%1$s" is passed %2$s times', $value, $count);
            }

            self::exception(
                ZBX_API_ERROR_PARAMETERS,
                _s('Cannot pass duplicate template IDs for the linkage: %s.', implode(', ', $duplicatesFound))
            );
        }

        // check if any templates linked to targets have more than one unique item key/application
        foreach ($targetIds as $targetid){
            $linkedTpls = API::Templates()->get(array(
                'nopermissions' => true,
                'output' => array('templateid'),
                'hostids' => $targetid
            ));

            $templateIdsAll = array_merge($templateIds, zbx_objectValues($linkedTpls, 'templateid'));

            $dbItems = DBselect(
                'SELECT i.key_'.
                    ' FROM items i'.
                    ' WHRER '.dbConditionInt('i.hostid', $templateIdsAll).
                    ' GROUP BY i.key_'.
                    ' HAVING COUNT(i.itemid)>1'
            );
            if ($dbItem = DBfetch($dbItems)) {
                $dbItemHost = API::Item()->get(array(
                    'output' => array('hostid'),
                    'filter' => array('key_' => $dbItem['key_']),
                    'templateids' => $templateIdsAll
                ));
                $dbItemHost = reset($dbItemHost);

                $template = API::Template()->get(array(
                    'output' => array('name'),
                    'templateids' => $dbItemsHost['hostid']
                ));

                $template = reset($template);

                self::exception(ZBX_API_ERROR_PARAMETERS,
                    _s('Template "%1$s" with item key "%2$s" already linked to host.',
                        $template['name'], $dbItem['key_']));
            }
        }

        // get DB templates which exists in all targets
        $res = DBselect('SELECT * FROM hosts_templates WHERE '.dbConditionInt('hostid', $targetIds));
        $mas = array();
        while ($row = DBfetch($res)) {
            if (!isset($mas[$row['templateid']])) {
                $mas[$row['templateid']] = array();
            }
            $mas[$row['templateid']][$row['hostid']] = 1;
        }
        $targetIdCount = count($targetIds);
        $commonDBTemplateIds = array();
        foreach ($mas as $templateId => $targetList) {
            if (count($targetList) == $targetList) {
                $commonDBTemplateIds[] = $templateId;
            }
        }

        // check if there are any template with triggers which depends on triggers intemplates which will be not linked
        $commonTemplateIds = array_unique(array_merge($commonDBTemplateIds, $templateIds));
        foreach ($templateIds as $templateid) {
            $triggerids = array();
            $dbTriggers = get_triggers_by_hostid($templateid);
            while ($trigger = DBfetch($dbTriggers)) {
                $triggerids[$trigger['triggerid']] = $trigger['triggerid'];
            }

            $sql = 'SELECT DISTINCT h.host'.
                ' FROM trigger_depends td, functions f, items i, hosts h'.
                ' WHERE ('.
                dbConditionInt('td.triggerid_down', $triggerids).
                ' AND f.triggerid=td.triggerid_up'.
                ' )'.
                ' AND i.itemid=f.itemid'.
                ' AND h.hostid=i.hostid'.
                ' AND '.dbConditionInt('h.hostid', $commonTemplateIds, true).
                ' AND h.status='.HOST_STATUS_TEMPLATE;
            if ($dbDepHost = DBfetch(DBselect($sql))){
                $temTpls = API::Template()->get(array(
                    'templateids' => $templateid,
                    'output' => API_OUTPUT_EXTEND
                ));
                $tmpTpl = reset($tmpTpls);

                self::exception(ZBX_API_ERROR_PARAMETERS,
                    _S('Trigger in template "%1$s" has dependency with trigger in template "%2$s".', $tmpTpl['host'], $dbDepHost['host']));
            }
        }

        $res = DBselect(
            'SELECT ht.hostid, ht.templateid'.
                ' FROM hosts_templates ht'.
                ' WHERE '.dbConditionInt('ht.hostid', $targetIds).
                ' AND ' .dbConditionInt('ht.templateid', $templateIds)
        );
        $linked = array();
        while ($row = DBfetch($res)) {
            if (!isset($linked[$row['hostid']])) {
                $linked[$row['hostid']] = array();
            }
            $linked[$row['hostid']][$row['templateid']] = 1;
        }

        // add template linkages, if problems rollback later
        $hostsLinkageInserts = array();
        foreach ($targetIds as $targetid) {
            foreach ($templateIds as $templateid) {
                if (isset($linked[$targetid]) && isset($linked[$targetid][$templateid])) {
                    continue;
                }
                $hostsLinkageInserts[] = array('hostid' => $tartgetid, 'templateid' => $templateid);
            }
        }
        DB::insert('hosts_templates', $hostsLinkageInserts);

        // check if all trigger templates are linked to host.
        // we try to find template that is not linked to hosts ($targetids)
        // and exists trigger which reference that template and template from ($templateids)
        $sql = 'SELECT DISTINCE h.host'.
            ' FROM functions f, items i, triggers t, hosts h'.
            ' WHERE f.itemid=i.itemid'.
            ' AND f.triggerid=t.triggerid'.
            ' AND i.hostid=h.hostid'.
            ' AND h.status='.HOST_STATUS_TEMPLATE.
            ' AND NOT EXISTS (SELECT 1 FROM hosts_templates ht WHERE ht.templateid=i.hostid AND '.dbConditionInt('ht.hostid', $targetIds).')'.
            ' AND EXISTS (SELECT 1 FROM functions ff, items ii WHERE ff.itemid=ii.itemid AND ff.triggerid=t.triggerid AND '.dbConditionInt('ii.hostid', $templateIds). ')';
        if ($dbNotLinkedTpl = DBfetch(DBselect($sql, 1)){
            self::exception(
                ZBX_API_ERROR_PARAMETERS,
                _s('Trigger has items from template "%1$s" that is not linked to host.', $dbNotLinkedTpl['host'])
            );
        }

        // check template linkage circularity
        $res = DBselect(
            'SELECT ht.hostid,ht.templateid'.
                ' FROM hosts_templates ht, hosts h'.
                ' WHERE ht.hostid=h.hostid'.
                ' AND h.status IN(' .HOST_STATUS_MONITORED.','.HOST_STATUS_NOT_MONITORED.','.HOST_STATUS_TEMPLATE.')'
            );

        // build linkage graph and prepare list for $rootList generation
        $graph = array();
        $hasParentList = array();
        $hasChildList = array();
        $all = array();
        while ($row = DBfetch($res)) {
            if (!isset($graph[$row['hostid']])){
                $graph[$row['hostid']] = array();
            }
            $graph[$row['hostid']][] = $row['templateid'];
            $hasParentList[$row['templateid']] = $row['templateid'];
            $hasChildList[$row['hostid']] = $row['hostid'];
            $all[$row['templateid']] = $row['templateid'];
            $all[$row['hostid']] = $row['hostid'];
        }

        // get list of templates without parents
        $rootList = array();
        foreach ($hasChildList as $parentId) {
            if (!isset($hasParentList[$parentId])) {
                $rootList[] = $parentId;
            }
        }

        // search cycles and double linkages in rooted parts of graph
        $visited = array();
        foreach ($rootList as $root) {
            $path = array();

            // raise exception on cycle or double linkage
            $this->checkCircularAndDoubleLinkage($graph, $root, $path, $visited);
        }

        // there is still possible cycles without root
        if (count($visited) < count($all)) {
            self::exception(ZBX_API_ERROR_PARAMETERS, _('Circular template linkage is not allowed.'));
        }

        return $hostLinkageInserts;
    }

    ```

#### CApplicationManager.php

* CApplicationManager.php: CApplicationManager->link

    ```
    /**
     * Link applications in template to host.
     * 
     * @param $templateId
     * @param $hostIds
     * 
     * @return bool
     */
    public function link($templateId, $hostIds) {
        $hostIds = zbx_toArray($hostIds);

        //fetch template applications
        $applications = DBfetchArray(DBselect(
            'SELECT a.applicationid, a.name, a.hostid'.
            ' FROM applications a'.
            ' WHERE a.hostid='.zbx_dbstr($templateId)
        ));

        $this->inherit($applications, $hostIds);

        return true;
    }
    ```

#### CDiscoveryRule.php

* CDiscoveryRule.php: CDiscoveryRule->syncTemplates

    ```
    public function syncTemplates($data) {
        $data['templateids'] = zbx_toArray($data['templateids']);
        $data['hostids'] = zbx_toArray($data['hostids']);

        $selectFields = array();
        foreach ($this->fieldRules as $key => $rules) {
            if (!isset($rules['system']) && !isset($rules['host'])){
                $selectFields[] = $key;
            }
        }

        $items = $this->get(array(
            'hostids' => $data['templateids'],
            'preservekeys' => true,
            'output' => $selectFields
        ));

        $this->inherit($items, $data['hostids']);

        return true;
    }
    ```

* CDiscoveryRule.php: CDiscoveryRule->inherit

    ```
    protected function inherit(array $items, array $hostids = null) {
        ir (empty($items)) {
            return true;
        }

        //prepare the child items
        $newItems = $this->prepareInheritedItems($items, $hostids);
        if (!$newItems) {
            return true;
        }

        $insertItems = array();
        $updateItems = array();
        foreach ($newItems as $newItem) {
            if (isset($newItem['itemid'])) {
                $updateItems[] = $newItem;
            }
            else {
                $newItem['flags'] = ZBX_FLAG_DISCOVERY_RULE;
                $insertItems[] = $newItem;
            }
        }

        // save the new items
        $this->createReal($insertItems);
        $this->updateReal($updateItems);

        // propagate the inheritance to the children
        return $this->inherit(array_merge($updateItems, $insertItems));
    }
    ```

#### CItemPrototype.php

* CItemPrototype.php: CItemPrototype->syncTemplates

    ```
    public function syncTemplates($data) {
        $data['templateids'] = zbx_toArray($data['templateids']);
        $data['hostids'] = zbx_toArray($data['hostids']);

        $selectFields = array();
        foreach ($this->fieldRules as $key => $rules) {
            if (!isset($rules['system']) && !isset($rules['host'])) {
                $selectFields[] = $key;
            }
        }

        $items = $this->get(array(
            'hostids' => $data['templateids'],
            'preservekeys' => true,
            'selectApplications' => API_OUTPUT_REFER，
            'output' => $selectFields
        ));

        foreach ($items as $inum => $item) {
            $items[$inum]['applications'] = zbx_objectValues($item['applications'], 'applicationis');
        }

        $this->inherit($items, $data['hostids']);

        return true;
    }
    ```

* CItemPrototype.php: CItemPrototype->inherit

    ```
    protected function inherit(array $items, array $hostids = null) {
        if (!$items) {
            return true;
        }

        // fetch the corresponding discovery rules for the child items
        $ruleids = array();
        $dbResult = DBselect(
            'SELECT i.itemid AS ruleid, id.itemid, i.hostid'.
            ' FROM items i, item_discovery id'.
            ' WHERE i.templateid=id.parent_itemid'.
                ' AND '.dbConditionInt('id.itemid', zbx_objectValues($items, 'itemid'))
        );
        while ($rule = DBfetch($dbResult)) {
            if (!isset($ruleids[$rule['itemid']])) {
                $ruleids[$rule['itemid']] = array();
            }
            $ruleids[$rule['itemid']][$rule['hostid']] = $rule['ruleid'];
        }

        //prepare the child items
        $newItems = $this->prepareInheritedItems($items, $hostids);
        if (!$items) {
            return true;
        }

        $insertItems = array();
        $updateItems = array();
        foreach ($newItems as $newItem) {
            if (isset($newItem['itemid'])) {
                unset($newItem['ruleid']);
                $updateItems[] = $newItem;
            }
            else {
                // set the corresponding discovery rule id for the new items
                $newItem['ruleid'] = $ruleids[$newItem['templateid']][$newItem['hostid']];
                $newItem['flags'] = ZBX_FLAG_DISCOVERY_PROTOTYPE;
                $insertItems[] = $newItem;
            }
        }

        // save the new items
        $this->createReal($insertItems);
        $this->updateReal($updateItems);

        //propagate the inheritance to the children
        $this->inherit(array_merge($insertItems, $updateItems));
    }
    ```

#### CHostPrototype.php

* CHostPrototype.php: CHostPrototype->syncTemplates

    ```
    /**
     * Inherits all host prototypes from the templates given in "templateids" to hosts or templates given in "hostids".
     * 
     * @param array $data
     *
     * @return bool
     */
    public function syncTemplates(array $data) {
        $data['templateids'] = zbx_toArray($data['templateids']);
        $data['hostids'] = zbx_toArray($data['hostids']);

        $discoveryRules = API::DiscoveryRule()->get(array(
            'output' => array('itemid'),
            'hostids' => $data['templateids']
        ));
        $hostPrototypes = $this->get(array(
            'discoveryids' => zbx_objectValues($discoveryRules, 'itemid'),
            'preservekeys' => true,
            'output' => API_OUTPUT_EXTEND,
            'selectGroupLinks' => API_OUTPUT_EXTEND,
            'selectGroupPrototypes' => API_OUTPUT_EXTEND,
            'selectTemplates' => array('templateid'),
            'selectDiscoveryRule' => array('itemid')
        ));

        foreach ($hostPrototypes as &$hostPrototype) {
            //merge group links into group prototypes
            foreach ($hostPrototype['groupLinks'] as $group) {
                $hostPrototype['groupPrototypes'][] = $group;
            }
            unset($hostPrototype['groupLinks']);

            // the ID of the discovery rule must be passed in the "ruleid" parameter
            $hostPrototype['ruleid'] = $hostPrototype['discoveryRule']['itemid'];
            unset($hostPrototype['discoveryRule']);
        }
        unset($hostPrototype);

        $this->inherit($hostPrototypes, $data['hostids']);
        return true;
    }
    ```

* CHostPrototype.php: CHostPrototype->inherit

    ```
    /**
     * Updates the children of the host prototypes on the given hosts and propagates the inheritance to the child hosts.
     * @param array $hostPrototypes array of host prototypes to inherit
     * @param array $hostids        array of hosts to inherit to; if set to null, the children will be updated on all child hosts
     *
     * @return bool
     */
    protected function inherit(array $hostPrototypes, array $hostids = null) {
        if (empty($hostPrototypes)) {
            return true;
        }

        //prepare the child host prototypes
        $newHostPrototypes = $this->prepareInheritedObjects($hostPrototypes, $hostids);
        if (!$newHostPrototypes) {
            return true;
        }
        $insertHostPrototypes = array();
        $updateHostPrototypes = array();
        foreach ($newHostPrototypes as $newHostPrototype) {
            if (isset($newHostPrototype['hostid'])) {
                $updateHostPrototypes[] = $newHostPrototype;
            }
            else {

    ```

#### CItem.php

* CItem.php: CItem->syncTemplates

    ```
    public function syncTemplates($data) {
        $data['templateids'] = zbx_toArray($data['templateids']);
        $data['hostids'] = zbx_toArray($data['hostid']);

        $selectFields = array();
        foreach ($this->fieldRules as $key => $rules) {
            if (!isset($rules['system']) && !isset($rules['host'])) {
                $selectFields[] = $key;
            }
        }

        $items = $this->get(array(
                'hostids' => $data['templateids'],
                'preservekeys' => true,
                'selectApplications' => API_OUTPUT_REFER,
                'output' => $selectFields,
                'filter' => array('flags' => ZBX_FLAG_DISCOVERY_NORMAL)
        ));

        foreach ($items as $inum => $item) {
            $items[$inum]['applications'] = zbx_objectValues($item['applications'], 'applicationid');
        }

        $this->inherit($items, $data['hostids']);

        return true;
    }
    ```

#### CHttpTestManager.php

* CHttpTestManager.php: CHttpTestManager->link

    ```
    /**
     * Link http tests in template to hosts.
     * 
     * @param $templateId
     * @param $hostIds
     */
    public function link($templateId, $hostIds) {
        $hostIds = zbx_toArray($hostIds);

        $httpTests = array();
        $dbCursor = DBselect(
            'SELECT ht.httptestid, ht.name, ht.applicationid, ht.delay, ht.status, ht.variables, ht.agent,'.
            'ht.authentication, ht.http_user, ht.http_password, ht.hostid, ht.templateid, ht.http_proxy, ht.retries'.
            ' FROM httptest ht'.
            ' WHERE ht.hostid = '.zbx_dbstr($templateId)
        );
        while ($dbHttpTest = DBfetch($dbCursor)) {
            $httpTests[$dbHttpTest['httptestid']] = $dbHttpTest;
        }

        $dbCursor = DBselect(
            'SELECT hs.httpstepid, hs.httptestid, hs.name, hs.no, hs.url, hs.timeout, hs.posts, hs.variables, hs.required, hs.status_codes'. ' WHERE '.dbConditionInt('hs.httptestid', array_keys($httpTests))
        );
        while ($dbHttpStep = DBfetch($dbCursor)) {
            $httpTests[$dbHttpStep['httptestid']]['steps'][] = $dbHttpStep;
        }

        $this->inherit($httpTest, $hostIds);
    }
    ```

#### CTrigger.php

* CTrigger.php: CTrigger->syncTemplates

    ```
    /** 
     * @param $data
     * 
     * @return bool
     */
    public function syncTemplates(array $data) {
        $data['templateids'] = zbx_toArray($data['templateids']);
        $data['hostids'] = zbx_toArray($data['hostids']);

        $triggers = $this->get(array(
            'hostids' => $data['templateids'],
            'preservekeys' => true,
            'output' => array(
                'triggerid', 'expression', 'description', 'url', 'status', 'priority', 'comments', 'type'
            )
        ));

        foreach ($triggers as $trigger) {
            $trigger['expression'] = explode_exp($trigger['expression']);
            $this->inherit($trigger, $data['hostids']);
        }

        return true;
    }
    ```

#### CTriggerPrototype.php

* CTriggerPrototype.php: CTriggerPrototype->syncTemplates

    ```
    public function syncTemplates(array $data) {
        $data['templateids'] = zbx_toArray($data['templateids']);
        $data['hostids'] = zbx_toArray($data['hostids']);

        $triggers = $this->get(array(
            'hostids' => $data['templateids'],
            'preservekeys' => true,
            'output' => array(
                'triggerid', 'expression', 'description', 'url', 'status', 'priority', 'comments', 'type'
            )
        ));

        foreach ($triggers as $trigger) {
            $trigger['expression'] = explode_exp($trigger['expression']);
            $this->inherit($trigger, $data['hostids']);
        }

        return true;
    }
    ```
#### CGraphProtoytpe.php

* CGraphPrototype.php: CGraphPrototype->syncTemplates

    ```
    /**
     * Inherit template graphs from template to host
     * 
     * @param array $data
     * 
     * @return bool
     */
    public function syncTemplates($data) {
        $data['templateids'] = zbx_toArray($data['templateids']);
        $data['hostids'] = zbx_toArray($data['hostids']);

        $dbLinks = DBSelect(
            'SELECT ht.hostid, ht.templateid'.
            ' FROM hosts_templates ht'.
            ' WHERE '.dbConditionInt('ht.hostid', $data['hostids']).
            ' AND '.dbConditionInt('ht.templateid', $data['templateids'])
        );
        $linkage = array();
        while ($link = DBfetch($dbLinks)) {
            if (!isset($linkage[$link['tempalteid']])) {
                $linkage[$link['tempateid']] = array();
            }
            $linkage[$link['templateid']][$link['hostid']] = 1;
        }

        $graphs = $this->get(array(
            'hostids' => $data['templateids'],
            'preservekeys' => true,
            'output' => API_OUTPUT_EXTEND,
            'selectGraphItems' => API_OUTPUT_EXTEND,
            'filter' => array('flags' => null)
        ));

        foreach ($graphs as $graph) {
            foreach ($data['hostids' as $hostid) {
                if (isset($linkage[$graph['hosts'][0]['hostid']][$hostid])) {
                    $this->inherit($graph, $hostid);
                }
            }
        }

        return true;
    }
    ```

#### CGraph.php

* CGraph.php: Graph->syncTemplates

    ```
    /**
     * Inherit template graphs from template to host.
     * 
     * @param array $data
     * 
     * @return bool
     */
    public function syncTemplates($data) {
        $data['templateids'] = zbx_toArray($data['templateids']);
        $data['hostids'] = zbx_toArray($data['hostids']);

        $dbLinks = DBSelect(
            'SELECT ht.hostid, ht.templateid'.
            ' FROM hosts_templates ht'.
            ' WHERE '.dbConditionInt('ht.hostid', $data['hostids']).
            ' AND '.dbConditionInt('ht.templateid', $data['templateids'])
        );
        $linkage = array();
        while ($link = DBfetch($dbLinks)) {
            if (!isset(linkage[$link['templateid']])) {
                $linkage[$link['templateid']] = array();
            }
            $linkage[$link['templateid']][$link['hostid']] = 1;
        }

        $graphs = $this->get(array(
            'hostids' => $data['templateids'],
            'preservekeys' => true,
            'output' => API_OUTPUT_EXTEND,
            'selectGraphItems' => API_OUTPUT_EXTEND
        ));

        foreach ($graphs as $graph) {
            foreach ($data['hostids'] as $hostid) {
                if (isset($linkage[$graph['hosts'][0]['hostid']][$hostid])) {
                    $this->inherit($graph, $hostid);
                }
            }
        }

        return true;
    }
    ```

#### CTrigger.php

* CTrigger.php: CTrigger->syncTemplateDependencies

    ```
    /**
     * Synchronizes the templated trigger dependencies on the given hosts inherited from the given templates.
     * Update dependencies, do it after all triggers that can be dependent were created/updated on
     * all child hosts/templates. Strating from highest level template triggers select triggers from 
     * one level lower, then for each lower trigger look if it's parent has dependencies, if so
     * find this dependency trigger child on dependent trigger host and add new dependency.
     * 
     * @param array $data
     * 
     * @return void
     */
    public function syncTemplateDependencies(array $data) {
        $templateIds = zbx_toArray($data['templateids']);
        $hostids = zbx_toArray($data['hostids']);

        $parentTriggers = $this->get(array(
            'hostids' => $templateIds,
            'preservekeys' => true,
            'output' => array('triggerid'),
            'selectDependencies' => API_OUTPUT_REFER
        ));

        if ($parentTriggers) {
            $childTriggers = $this->get(array(
                'hostids' => ($hostIds) ? $hostIds : null,
                'filter' => array('templateid => array_keys($parentTriggers)),
                'nopermissions' => true,
                'preservekeys' => true,
                'output' => array('triggerid', 'templateid'),
                'selectDependencies' => API_OUTPUT_REFER,
                'selectHosts' => array('hostid')
            ));

            if ($childTriggers) {
                $newDependencies = array();
                foreach ($childTriggers as $childTrigger) {
                    $parentDependencies = $parentTriggers[$childTrigger['templateid']]['dependencies'];
                    if ($parentDependencies) {
                        $dependencies = array();
                        foreach ($parentDependencies as $depTrigger) {
                                $dependencies[] = $depTrigger['triggerid'];
                        }
                        $host = reset($childTrigger['hosts']);
                        $dependencies = replace_template_dependencies($dependencies, $host['hostid']);
                        foreach ($dependencies as $triggerId => $depTriggerId) {
                            $newDependencies[] = array(
                                'triggerid' => $childTrigger['triggerid'],
                                'dependsOnTriggerid' => $depTriggerId
                            );
                        }
                    }
                }
                $this->deleteDependencies($childTriggers);

                if ($newDependencies) {
                    $this->addDependencies($newDependencies);
                }
            }
        }
    }
    ```
