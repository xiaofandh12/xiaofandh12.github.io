---
layout: post
title: Zabbix源码分析: Zabbix Server的main_poller_loop()分析
description: Zabbix Server, main_poller_loop()
category: blog
---

## 相关数据结构

对于进程```main_poller_loop()```来说，最重要的变量为```cache```，```cache```是一个类型为```ZBX_DC_CACHE```的结构体；进程```main_poller_loop()```的主要作用就是为```cache```赋值。

在```cache```中，可以保存各```item```的历史数据，各```item```的趋势数据，各类型```item```历史数据的数量，以及相关的其它数据。

结构体```ZBX_DC_CACHE```的具体内容如下所示：

* ZBX_DC_CACHE

    ```
    typedef struct
    {
        zbx_hashset_t trends; /* 各item的趋势数据 */
        ZBX_DC_STATS stats; /* 各类型item历史数据的数量 */
        ZBX_DC_HISTORY *history; /* 各item的历史数据 */
        char *text;
        zbx_uint64_t *itemids;
        char *last_text;
        int history_first;
        int history_num;
        int history_gap_num;
        int text_free;
        int trends_num;
        int itemids_alloc;
        int itemids_num;
        zbx_timespec_t last_ts;
    }
    ZBX_DC_CACHE;

    static ZBX_DC_CACHE *cache = NULL;
    ```

在ZBX_DC_CACHE中，用到了```zbx_hashset_t```、```ZBX_DC_STATS```、```ZBX_DC_HISTORY```等结构体，关于```zbx_hashset_t```在[Zabbix Server源码分析之main_dbconfig_loop()](http://xiaofandh12.github.io/Zabbix-Server-dbconfig/)已做了分析，此处不再赘述；接下来只对```ZBX_DC_CACHE```,```ZBX_DC_HISTORY```做相关介绍。

结构体```ZBX_DC_HISTORY```存储某一个```item```某一次的历史数据，其具体内容如下所示：

* ZBX_DC_HISTORY

    ```
    typedef struct
    {
        zbx_uint64_t itemid;
        history_value_t value_orig;
        history_value_t value; 
        zbx_uint64_t lastlogsize;
        zbx_timespec_t ts;
        int timestamp;
        int severity;
        int logeventid;
        int mtime;
        int num;
        unsigned char value_type;
        unsigned char value_null;
        unsigned char keep_history;
        unsigned char keep_trends;
        unsigned char state;
    }
    ZBX_DC_HISTORY;
    ```

结构体```ZBX_DC_STATS```保存了各类型（总的item、类型为float的item、类型为uint的item、类型为str的item、类型为log的item、类型为text的item、暂不支持的item）item历史数据的数量，其具体内容如下所示：

* ZBX_DC_STATS

    ```
    typedef struct
    {
        zbx_uint64_t history_counter; /* the total number of processed values */
        zbx_uint64_t history_float_counter; /* the number of processed float values */
        zbx_uint64_t history_uint_counter; /* the number of processed uint values */
        zbx_uint64_t history_str_counter; /* the number of processed str values */
        zbx_uint64_t history_log_counter; /* the number of processed log values */
        zbx_uint64_t history_text_counter; /* the number of processed text values */
        zbx_uint64_t notsupported_counter; /* the number of processed not supported items */
    }
    ZBX_DC_STATS;
    ```

在结构体```ZBX_DC_ITEM```中,存储有```item```公有的一些信息，根据```item->value_type```的不同，额外的信息会存入```ZBX_DC_NUMITEM```、```ZBX_DC_LOGITEM```中；
根据```item->type```的不同，额外的信息会存入```ZBX_DC_SNMPITEM```、```ZBX_DC_IPMIITEM```、```ZBX_DC_FLEXITEM```、```ZBX_DC_TRAPITEM```、```ZBX_DC_DBITEM```、```ZBX_DC_SSHITEM```、```ZBX_DC_TELNETITEM```、```ZBX_DC_SIMPLEITEM```、```ZBX_DC_JMXITEM```、```ZBX_DC_CALCITEM```、```ZBX_DC_INTERFACE_ITEM```中；```ZBX_DC_ITEM```具体内容如下所示：

* ZBX_DC_ITEM

    ```
    typedef struct
    {
        zbx_uint64_t itemid;
        zbx_uint64_t hostid;
        zbx_uint64_t interfaceid;
        zbx_uint64_t valuemapid;
        const char *key;
        const char *port;
        const char *units;
        const char *db_error;
        ZBX_DC_TRIGGER **triggers;
        int delay;
        int nextcheck;
        int lastclock;
        int mtime;
        int data_expected_from;
        int history;
        unsigned char type;
        unsigned char data_type;
        unsigned char value_type;
        unsigned char poller_type;
        unsigned char state;
        unsigned char db_state;
        unsigned char inventory_link;
        unsigned char location;
        unsigned char flags;
        unsigned char status;
        unsigned char unreachable;
    }
    ZBX_DC_ITEM;
    ```

如前所述，```item```的信息是存储于不同变量中的，为应用方便又定义了一个结构体```DC_ITEM```来存储```item```的所有信息，```DC_ITEM```具体内容如下所示：

* DC_ITEM

    ```
    typedef struct
    {
        DC_HOST host;
        DC_INTERFACE interface;
        zbx_uint64_t itemid;
        zbx_uint64_t lastlogsize;
        zbx_uint64_t valuemapid;
        unsigned char type;
        unsigned char data_type;
        unsigned char value_type;
        unsigned char delta;
        unsigned char multiplier;
        unsigned char state;
        unsigned char db_state;
        unsigned char snmpv3_securitylevel;
        unsigned char authtype;
        unsigned char flags;
        unsigned char snmpv3_authprotocol;
        unsigned char snmpv3_privprotocol;
        unsigned char inventory_link;
        unsigned char status;
        unsigned char unreachable;
        char key_orig[ITEM_KEY_LEN * 4 + 1], *key;
        char *formula;
        char *units;
        int delay;
        int nextcheck;
        int lastclock;
        int mtime;
        int history;
        int trends;
        char trapper_hosts[ITEM_TRAPPER_HOSTS_LEN_MAX];
        char logtimefmt[ITEM_LOGTIMEFMT_LEN_MAX];
        char snmp_community_orig[ITEM_SNMP_COMMUNITY_LEN_MAX],*snmp_community;
        char snmp_oid_orig[ITEM_SNMP_OID_LEN_MAX],*snmp_oid;
        char snmpv3_securityname_orig[ITEM_SNMPV3_SECURITYNAME_LEN_MAX],*snmpv3_securityname;
        char snmpv3_authpassphrase_orig[ITEM_SNMPV3_AUTHPASSPHRASE_LEN_MAX], *snmpv3_authpassphrase;
        char snmpv3_privpassprhrase_orig[ITEM_SNMPV3_PRIVPASSPHRASE_LEN_MAX], *snmpv3_privpassphrase;
        char ipmi_sensor[ITEM_IPMI_SENSOR_LEN_MAX];
        char *params;
        char delay_flex[ITEM_DELAY_FLEX_LEN_MAX];
        char username_orig[ITEM_USERNAME_LEN_MAX], *username;
        char publickey_orig[ITEM_PUBLICKEY_LEN_MAX], *publickey;
        char privatekey_orig[ITEM_PRIVATEKEY_LEN_MAX], *privatekey;
        char password_orig[ITEM_PASSWORD_LEN_MAX], *password;
        char snmpv3_contextname_orig[ITEM_SNMPV3_CONTEXTNAME_LEN_MAX], *snmpv3_contextname;
        char *db_error;
    }
    DC_ITEM;
    ```

一个```item```的值是有不同类型的，在Zabbix中```item```的值的类型包括```float```、```str```、```log```、```uint```、```text```，这些信息存储于枚举类型```zbx_item_value_type_t```中，```zbx_item_value_type_t```具体内容如下所示：

* zbx_item_value_type_t

    ```
    /* item value types */
    typedef enum
    {
        ITEM_VALUE_TYPE_FLOAT = 0,
        ITEM_VALUE_TYPE_STR,
        ITEM_VALUE_TYPE_LOG,
        ITEM_VALUE_TYPE_UINT64,
        ITEM_VALUE_TYPE_TEXT,
        /* the number of defined value types */
        ITEM_VALUE_TYPE_MAX
    }
    zbx_item_value_type_t;
    ```

一个```item```的值可以通过多种方式获取，根据获取方式的不同，一个```item```有不同的类型，在Zabbix中一个```item```的类型包括```ITEM_TYPE_ZABBIX```、```ITEM_TYPE_SNMPv1```、```ITEM_TYPE_TRAPPER```等，这些信息存储于枚举类型```zbx_item_type_t```中，```zbx_item_type_t```具体内容如下所示：

* zbx_item_type_t

    ```
    /* item types */
    typedef enum
    {
        ITEM_TYPE_ZABBIX = 0,
        ITEM_TYPE_SNMPv1,
        ITEM_TYPE_TRAPPER,
        ITEM_TYPE_SIMPLE,
        ITEM_TYPE_SNMPv2c,
        ITEM_TYPE_INTERNAL,
        ITEM_TYPE_SNMPv3,
        ITEM_TYPE_ZABBIX_ACTIVE,
        ITEM_TYPE_AGGREGATE,
        ITEM_TYPE_HTTPTEST,
        ITEM_TYPE_EXTERNAL,
        ITEM_TYPE_DB_MONITOR,
        ITEM_TYPE_IPMI,
        ITEM_TYPE_SSH,
        ITEM_TYPE_TELNET,
        ITEM_TYPE_CALCULATED,
        ITEM_TYPE_JMX,
        ITEM_TYPE_SNMPTRAP /* 17 */
    }
    zbx_item_type_t;
    ```

进程```main_poller_loop```一次会获取一个或多个```item```的数据，这些数据不会直接一个一个的存入```cache```中，而是会先存入```item_values```中，最后再将```item_values```中的数据一次刷入```cache```中，```item_values```的类型为```dc_item_value_t```，```dc_item_value_t```为一结构体，其具体内容如下所示：

* dc_item_value_t

    ```
    typedef struct
    {
        zbx_uint64_t itemid;
        dc_value_t value;
        zbx_timespec_t ts;
        dc_value_str_t source;
        zbx_uint64_t lastlogsize;
        int timestamp;
        int severity;
        int logeventid;
        int mtime;
        unsigned char value_type;
        unsigned char state;
        unsigned char flags;
    }
    dc_item_value_t;

    static dc_item_value_t *item_values = NULL;
    staic size_t item_values_alloc = 0, item_values_num = 0;
    ```

```config->hosts```中存储的```host```的类型为```ZBX_DC_HOST```，```ZBX_DC_HOST```的具体内容如下所示：

* ZBX_DC_HOST

    ```
    typedef struct
    {
        zbx_uint64_t hostid;
        zbx_uint64_t proxy_hostid;
        const char *host;
        const char *name;
        int maintenance_from;
        int data_expected_from;
        int errors_from;
        int disable_until;
        int snmp_errors_from;
        int snmp_disable;
        int ipmi_errors_from;
        int ipmi_disable_until;
        int jmx_errors_from;
        int jmx_disable_until;
        unsigned char maintenance_status;
        unsigned char maintenance_type;
        unsigned char available;
        unsigned char snmp_available;
        unsigned char ipmi_available;
        unsigned char jmx_available;
        unsigned char status;
    }
    ZBX_DC_HOST;
    ```

```DC_ITEM```中有个host字段的类型为```DC_HOST```，```DC_HOST```相比于```ZBX_DC_HOST```有着更全面的信息，```DC_HOST```的具体内容如下所示：

* DC_HOST

    ```
    typedef struct
    {
        zbx_uint64_t hostid;
        zbx_uint64_t proxy_hostid;
        char host[HOST_HOST_LEN_MAX];
        char name[HOST_HOST_LEN_MAX];
        unsigned char maintenance_staus;
        unsigned char maintenance_type;
        int maintenance_from;
        int errors_from;
        unsigned char available;
        int disable_until;
        int snmp_errors_from;
        unsigned char snmp_available;
        int snmp_disable_until;
        int ipmi_errors_from;
        unsigned char ipmi_available;
        int ipmi_disable_until;
        signed char ipmi_authtype;
        unsigned char ipmi_privilege;
        char ipmi_username[HOST_IPMI_username_LEN_MAX];
        char ipmi_password[HOST_IPMI_PASSWORD_LEN_MAX];
        int jmx_errors_from;
        unsigned char jmx_available;
        int jmx_disable_until;
        char inventory_mode;
        unsigned char status;
    }
    DC_HOST;
    ```

## 进程main_poller_loop()的主要函数

```main_poller_loop```主要作用是通过调用函数```get_values(unsigned char poller_type)```来获取监控数据，传入函数```get_values```的参数为```poller_type```,目前```poller_type```共有5种：```ZBX_POLLER_TYPE_NORMAL```、```ZBX_POLLER_TYPE_UNREACHABLE```、```ZBX_POLLER_TYPE_IPMI```、```ZBX_POLLER_TYPE_PINGER```、```ZBX_POLLER_TYPE_JAVA```。

* Poller.c ==> void main__poller_loop(unsigned char poller_type)

    ```
    void main_poller_loop(unsigned char poller_type)
    {
        int nextcheck, sleeptime = -1, processed = 0, old_processed = 0;
        double sec, total_sec = 0.0, old_total_sec = 0.0;
        time_t last_stat_time;

    #define STAT_INTERVAL 5 /* if a process is busy and does not sleep then update status not faster than once in STAT_INTERVAL seconds */

        zbx_setproctitle("%s #%d [connecting to the database]", get_process_type_string(process_type), process_num);
        last_stat_time = time(NULL);

        DBconnect(ZBX_DB_CONNECT_NORMAL);

        for (;;)
        {
            if (0 != sleeptime)
            {
                zbx_setproctitle("%s #%d [got %d values in " ZBX_FS_DBL " sec, getting values]",
                        get_process_type_string(process_type), process_num, old_processed,
                        old_total_sec);
            }

            sec = zbx_time();
            processed += get_values(poller_type);
            total_sec += zbx_time() - sec;

            nextcheck = DCconfig_get_poller_nextcheck(poller_type);
            /*
             *通过函数calculate_sleeptime来计算for循环睡眠时间，其中当nextcheck小于当前时间时sleeptime为0，
             *在这种情况下for循环没有睡眠时间，会持续调用函数get_values。
             */
            sleeptime = calculate_sleeptime(nextcheck, POLLER_DELAY);

            if (0 != sleeptime || STAT_INTERVAL <= time(NULL) - last_stat_time)
            {
                if (0 == sleeptime)
                {
                    zbx_setproctitle("%s #%d [got %d values in " ZBX_FS_DBL " sec, getting values]",
                        get_process_type_string(process_type), process_num, processed, total_sec);
                }
                else
                {
                    zbx_setproctitle("%s #%d [got %d values in " ZBX_FS_DBL " sec, idle %d sec]",
                        get_process_type_string(process_type), process_num, processed, total_sec,
                        sleeptime);
                    old_processed = processed;
                    old_total_sec = total_sec;
                }
                processed = 0;
                total_sec = 0.0;
                last_stat_time = time(NULL);
            }

            zbx_sleep_loop(sleeptime);
        }

    #undef STAT_INTERVAL
    }
    ```

函数```get_values```主要做三件事：1，准备```items```(prepare items)，```items```指定了需要获取哪些```item```的数据；2，获取```items```中各```item```的数据(retrive item values)；3，处理获取到的各```item```的数据(process item values)，就是要把获取到的数据存入```cache```中。

* Poller.c ==> static int get_values(unsigned char poller_type)
    
    ```
    /*
     *Function Purpose: retrive values of metrics from monitored hosts
     *Function Return value: number of items processed
     */
    static int get_values(unsigned char poller_type)
    {
        const char *__function_name = "get_values";
        DC_ITEM items[MAX_POLLER_ITEMS];
        AGENT_RESULT results[MAX_POLLER_ITEMS];
        zbx_uint64_t lastlogsizes[MAX_POLLER_ITEMS];
        int errcodes[MAX_POLLER_ITEMS];
        zbx_timespec_t timespec;
        int i,num;
        char *port = NULL, error[ITEM_ERROR_LEN_MAX];
        int last_available = HOST_AVAILABLE_UNKNOWN;

        zabbix_log(LOG_LEVEL_DEBUG, "In %s()", __function_name);
    
        /* prepare items */
        /*   
         *调用函数DCconfig_get_poller_items准备items，这个函数并不是直接返回item的集合，
         *而是把空的items传入，在函数中把这个items填好即可，注意这里传入的为一指针；
         *而函数DCconfig_get_poller_items的返回值为items中item的数量，当poller_type为ZBX_POLLER_TYPE_NORMAL时，num为1。
         *item的数据是从config->queues中获取的，关于这个过程会在函数DCconfig_get_poller_items函数中分析。
         */
        num = DCconfig_get_poller_items(poller_type, items);
        
        if (0 == num)
            goto exit;
        
        //接下来的for循环，会对刚才获得的items中的每个item做额外处理，就是将item中某些字段再进行额外赋值。
        for (i = 0; i < num; i++)
        {
            init_result(&results[i]);
            errcodes[i] = SUCCEED;
            lastlogsizes[i] = 0;

            ZBX_STRDUP(items[i].key, items[i].key_orig);
            if (SUCCEED != substitute_key_macros(&items[i].key, NULL, &items[i], NULL,
                    MACRO_TYPE_ITEM_KEY, error, sizeof(error)))
            {
                SET_MSG_RESULT(&results[i], zbx_strdup(NULL, error));
                errcodes[i] = CONFIG_ERROR;
                continue;
            }

            switch (items[i].type)
            {
                case ITEM_TYPE_ZABBIX:
                case ITEM_TYPE_SNMPv1:
                case ITEM_TYPE_SNMPv2c:
                case ITEM_TYPE_SNMPv3:
                case ITEM_TYPE_IPMI:
                case ITEM_TYPE_JMX:
                    ZBX_STRDUP(port, items[i].interface.port_orig);
                    substitute_simple_macros(NULL, NULL, NULL, NULL, &items[i].host.hostid, NULL, NULL,
                        &port, MACRO_TYPE_COMMON, NULL, 0);
                    if (FAIL == is_ushort(port, &items[i].interface.port))
                    {
                        SET_MSG_RESULT(&results[i], zbx_dsprintf(NULL, "Invalid port number [%s]",
                                items[i].interface.port_orig));
                        errcodes[i] = CONFIG_ERROR;
                        continue;
                    }
                    break;
            }

            switch (items[i].type)
            {
                case ITEM_TYPE_SNMPv3:
                    ZBX_STRDUP(items[i].snmpv3_securityname, items[i].snmpv3_securityname_orig);
                    ZBX_STRDUP(items[i].snmpv3_authpassphrase, items[i].snmpv3_authpassphrase_orig);
                    ZBX_STRDUP(items[i].snmpv3_privpassphrase, items[i].snmpv3_privpassphrase_orig);
                    ZBX_STRDUP(items[i].snmpv3_contextname, items[i].snmpv3_contextname_orig);

                    substitute_simple_macros(NULL,NULL,NULL,NULL,&items[i].host.hostid,NULL,NULL,
                        &items[i].snmpv3_securityname,MACRO_TYPE_COMMON,NULL,0);
                    substitute_simple_macros(NULL,NULL,NULL,NULL,&items[i].host.hostid,NULL,NULL,
                        &items[i].snmpv3_authpassphrase,MACRO_TYPE_COMMON,NULL,0);
                    substitute_simple_macros(NULL,NULL,NULL,NULL,&items[i].host.hostid,NULL,NULL,
                        &items[i].snmpv3_privpassphrase,MACRO_TYPE_COMMON,NULL,0);
                    substitute_simple_macros(NULL,NULL,NULL,NULL,&items[i].host.hostid,NULL,NULL,
                        &items[i].snmpv3_contextname,MACRO_TYPE_COMMON,NULL,0);
                case ITEM_TYPE_SNMPv1:
                case ITEM_TYPE_SNMPv2c:
                    ZBX_STRDUP(items[i].snmp_community, items[i].snmp_community_orig);
                    ZBX_STRDUP(items[i].snmp_oid, items[i].snmp_oid_orig);

                    substitute_simple_macros(NULL,NULL,NULL,NULL,&items[i].host.hostid,NULL,NULL,
                        &items[i].snmp_community,MACRO_TYPE_COMMON,NULL,0);
                    if (SUCCEED != substitute_key_macros(&items[i].snmp_oid, &items[i].host.hostid,NULL,
                            NULL,MACRO_TYPE_SNMP_OID,error,sizeof(error)))
                    {
                        SET_MSG_RESULT(&results[i], zbx_strdup(NULL,error));
                        errcodes[i] = CONFIG_ERROR;
                        continue;
                    }
                    break;
                case ITEM_TYPE_SSH:
                    ZBX_STRDUP(items[i].publickey,items[i].publickey_orig);
                    ZBX_STRDUP(items[i].privatekey,items[i].privatekey_orig);

                    substitute_simple_macros(NULL,NULL,NULL,NULL,&items[i].host.hostid,NULL,NULL,
                        &items[i].publickey,MACRO_TYPE_COMMON,NULL,0);
                    substitute_simple_macros(NULL,NULL,NULL,NULL,&items[i].host.hostid,NULL,NULL,
                        &items[i].privatekey,MACRO_TYPE_COMMON,NULL,0);
                case ITEM_TYPE_TELNET:
                case ITEM_TYPE_DB_MONITOR:
                    substitute_simple_macros(NULL,NULL,NULL,NULL,NULL,NULL,&items[i],
                        &items[i].params,MACRO_TYPE_PARAMS_FIELD,NULL,0);
                case ITEM_TYPE_SIMPLE:
                case ITEM_TYPE_JMX:
                    itmes[i].username = zbx_strdup(items[i].username, items[i].username_orig);
                    items[i].password = zbx_strdup(items[i].password, items[i].password_orig);

                    substitute_simple_macros(NULL,NULL,NULL,NULL,&items[i].host.hostid,NULL,NULL,
                        &items[i].username,MACRO_TYPE_COMMON,NULL,0);
                    substitute_simple_macros(NULL,NULL,NULL,NULL,&items[i].host.hostid,NULL,NULL,
                        &items[i].password,MACRO_TYPE_COMMON,NULL,0);
                    break;
            }
        }

        zbx_free(port);

        /* retrieve item value */
        if (SUCCEED == is_snmp_type(items[0].type))
        {
    #ifdef HAVE_NETSNMP
            /*SNMP checks use their own timeouts */
            get_values_snmp(items, results, errcodes, num);
    #else
            for (i = 0;i < num; i++)
            {
                if (SUCCEED != errcodes[i])
                    continue;

                SET_MSG_RESULT(&results[i], zbx_strdup(NULL, "Support for SNMP checks was not compiled in."));
                errcodes[i] = CONFIG_ERROR;
            }
    #endif
        }
        else if (ITEM_TYPE_JMX == items[0].type)
        {
            alarm(CONFIG_TIMEOUT);
            get_values_java(ZBX_JAVA_GATEWAY_REQUEST_JMX,items,results,errcodes,num);
            alarm(0);
        }
        //当poller_type为ZBX_POLLER_TYPE_NORMAL时，num为1，此时就会调用函数get_value(&items[0], &results[0])来获取数据。
        else if (1 == num)
        {
            if (SUCCEED == errcodes[0])
                errcodes[0] = get_value(&items[0], &results[0]);
        }
        else
            THIS_SHOULD_NEVER_HAPPEN;

        zbx_timespec(&timespec);

        /* process item values */
        for (i = 0; i < num; i++)
        {
            switch (errcodes[i])
            {
                case SUCCEED:
                case NOTSUPPORTED:
                case AGENT_ERROR:
                case TIMEOUT_ERROR:
                    if (HOST_AVAILABLE_TRUE != last_available)
                        activate_host(&items[i],&timespec,&last_available);
                    break;
                case NETWORK_ERROR:
                case GATEWAY_ERROR:
                    if (HOST_AVAILABLE_FALSE != last_available)
                        deactivate_host(&items[i], &timespec, &last_available, result[i].msg);
                    break;
                case CONFIG_ERROR:
                    break;
                default:
                    zbx_error("unknown response code returned: %d", errcodes[i]);
                    assert(0);
            }

            if (SUCCEED == errcodes[i])
            {
                /* remove formatting symbols from the end of the result */
                /* so it could be checked by "is_uint64" and "is_double" functions */
                /* when we try to get "int" or "float" values from "string" result */
                if (ISSET_STR(&results[i]))
                    zbx_rtrim(results[i].str, ZBX_WHITESPACE);
                if (ISSET_TEXT(&results[i]))
                    zbx_rtrim(results[i].text, ZBX_WHITESPACE);

                items[i].state = ITEM_STATE_NORMAL;
                //通过调用函数dc_add_history将item的数据，存入item_values中，item_values已在相关数据结构章节中有了相关介绍。
                dc_add_history(items[i].itemid, items[i].value_type, items[i].flags, &results[i], &timespec,
                    items[i].state, NULL);

                if (0 != ISSET_LOG(&results[i]))
                {
                    if (NULL != results[i].logs[0])
                    {
                        size_t j;

                        for (j = 1; NULL != results[i].logs[j]; j++)
                            ;
                        lastlogsizes[i] = results[i].log[j - 1]->lastlogsize;
                    }
                    else
                        lastlogsizes[i] = items[i].lastlogsize;
                }
            }
            else if (HOST_AVAILABLE_FALSE != last_available)
            {
                items[i].state = ITEM_STATE_NOTSUPPORTED;
                dc_add_hisroty(items[i].itemid, items[i].value_type, items[i].flags,NULL,&timespec,
                        items[i].state,resutls[i].msg);
            }
            //items中item是从config->queues中获取的，在获取的时候是将获得的item从queue中移除掉的，所以在这里需要把item重新加入queue中。
            DCrequeue_items(&items[i].itemid, &items[i].state, &timespec, &lastlogsizes[i],NULL,
                    &errcodes[i],1);

            zbx_free(items[i].key);

            switch (items[i].type)
            {
                case ITEM_TYPE_SNMPv3:
                    zbx_free(items[i].snmpv3_securityname);
                    zbx_free(items[i].snmpv3_authpassphrase);
                    zbx_free(items[i].snmpv3_privpassphrase);
                    zbx_free(items[i].snmpv3_contextname);
                    /* break; is not missing here */
                case ITEM_TYPE_SNMPv1:
                case ITEM_TYPE_SNMPv2c:
                    zbx_free(items[i].snmp_community);
                    zbx_free(items[i].snmp_oid);
                    break;
                case ITEM_TYPE_SSH:
                    zbx_free(items[i].publickey);
                    zbx_free(items[i].privatekey);
                    /* break; is not missing here */
                case ITEM_TYPE_TELNET:
                case ITEM_TYPE_DB_MONITOR:
                case ITEM_TYUPE_SIMPLE:
                case ITEM_TYPE_JMX:
                    zbx_free(items[i].username);
                    zbx_free(items[i].password);
                    break;
            }

            free_result(&results[i]);
        }

        DCconfig_clean_items(items, NULL, num);
        //将item_values中的数据刷入cache中
        dc_flush_history();
    exit:
        zabbix_log(LOG_LEVEL_DEBUG, "End of %s():%d", __function_name, num);

        return num;
    }
    ```

如前所述，在函数```get_values```中共有3个步骤：prepare items, retrive item values, process item values；前面已对每个步骤做了大略的介绍，下面会对每个步骤做更为详细的讲解。

### prepare items

调用函数```DCconfig_get_poller_items```时，向其传入了指向类型```DC_ITEM```的名为```items```的指针，在函数```DCconfig_get_poller_items```中会从```config->queues```中获取数据，然后存入```items```中；函数```DCconfig_get_poller_items```的返回值为```items```中```item```的数量。


* Dbconfig.c ==> int DCconfig_get_poller_items(unsigned char poller_type, DC_ITEM *items)

    ```
    /*
     * Purpose: Get array of items for selected poller
     * Return value: number of items in items array
     * Comments: Items leave the queue only through this function. Pollers must
     *           always return the items they have taken using DCrequeue_items().
     */
    int DCconfig_get_poller_items(unsigned char poller_type, DC_ITEM *items)
    {
        const char *__function_name = "DCconfig_get_poller_items";

        int now, num = 0, max_items;
        zbx_binary_heap_t *queue;

        zabbix_log(LOG_LEVEL_DEBUG, "In %s() poller_type:%d", __function_name, (int)poller_type);

        now = time(NULL);

        //根据poller_type从config->queues中获取queue，该queue就是保存了item的二叉堆。
        queue = &config->queues[poller_type];

        /*
         *根据poller_type的不同，设置不同的max_items，max_items是指items中最多可存入的item的数量；
         *可以看到当poller_type为ZBX_POLLER_TYPE_NORMAL时，max_items中为1，也就是说此时items中最多只会存入一个item。
         */
        switch (poller_type)
        {
            case ZBX_POLLER_TYPE_JAVA:
                max_items = MAX_JAVA_ITEMS;
                break;
            case ZBX_POLLER_TYPE_PINGER:
                max_items = MAX_PINGER_ITEMS;
                break;
            default:
                max_items = 1;
        }

        LOCK_CACHE;

        //num小于max_items，并且queue不为空时，执行while循环。
        while (num < max_items && FAIL == zbx_binary_heap_empty(queue))
        {
            ...
            ZBX_DC_HOST *dc_host;
            //从queue中可以获取到一个类型为ZBX_DC_ITEM的信息
            ZBX_DC_ITEM *dc_item;
            ...

            //使用函数zbx_binary_heap_find_min取出queue中的第一个元素，将该元素赋值给min，该元素中包含一个dc_item的信息。
            min = zbx_binary_heap_find_min(queue);
            //min->data包含了一个dc_item的信息，但在赋值时需要做一下强制类型转换。
            dc_item = (ZBX_DC_ITEM *)min->data;

            ...

            //从queue中去除掉取出来的元素
            zbx_binary_heap_remove_min(queue);
            
            ...

            //使用函数zbx_hashset_search，根据dc_item中的hostid从&config->hosts中获得dc_host(指向ZBX_DC_HOST的指针)
            if (NULL == (dc_host = zbx_hashset_search(&config->hosts, &dc_item->hostid)))
                continue;

            ...

            //&items[num].host类型为DC_HOST，dc_host类型为ZBX_DC_HOST；使用函数DCget_host，用dc_host为&items[num].host赋值。
            DCget_host(&items[num].host, dc_host);
            //&items[num]类型为DC_ITEM，dc_item类型为ZBX_DC_ITEM；使用函数DCget_item，用dc_item为&items[num]赋值。
            DCget_item(&items[num], dc_item);
            num++;

            ...
        }

        UNLOCK_CACHE;

        zabbix_log(LOG_LEVEL_DEBUG, "End of %s():%d", __function_name, num);
    
        return num;
    }
    ```

### retrive item values

函数```get_value```有两个参数```item```(类型为DC_ITEM)，```result```(类型为AGENT_RESULT)，参数```item```指明此次要获取哪一个```item```的数据，参数```result```用来存储获取到的item的数据。

* Poller.c ==> static int get_value(DC_ITEM *item, AGENT_RESULT *result)

    ```
    static int get_value(DC_ITEM *item, AGENT_RESULT *result)
    {
        const char *__function_name = "get_value";
        int res = FAIL;

        zabbix_log(LOG_LEVEL_DEBUG, "In %s() key:'%s'", __function_name, item->key_orig);

        //根据item->type值的不同，调用不同的函数获取某item的数据
        switch (item->type)
        {
            /*
             * 当item->type的值为ITEM_TYPE_ZABBIX时，调用函数get_value_agent来获取该item的值，并将结果存入result。
             * 当item->type的值为其它时，会调用相应的函数获取该item的值，要想了解该item的值是具体是怎么获取的可以仔细研究相应函数。
             */
            case ITEM_TYPE_ZABBIX:
                alarm(CONFIG_TIMEOUT);
                res = get_value_agent(item, result);
                alarm(0);
                break;
            case ITEM_TYPE_IPMI:
    #ifdef HAVE_OPENIPMI
                res = get_value_ipmi(item, result);
    #else
                SET_MSG_RESULT(result, zbx_strdup(NULL, "Support for IPMI checks was not compiled in."));
                res = CONFIG_ERROR;
    #endif
                break;
            case ITEM_TYPE_SIMPLE:
                /* simple checks use their own timeouts */
                res = get_value_simple(item, result);
                break;
            case ITEM_TYPE_INTERNAL:
                res = get_value_internal(item, result);
                break;
            case ITEM_TYPE_DB_MONITOR:
    #ifdef HAVE_UNIXODBC
                alarm(CONFIG_TIMEOUT);
                res = get_value_db(item, result);
                alarm(0);
    #else
                SET_MSG_RESULT(result,
                        zbx_strdup(NULL, "Support for Database monitor checks was not compiled in."));
                res = CONFIG_ERROR;
    #endif
                break;
            case ITEM_TYPE_AGGREGATE:
                res = get_value_aggregate(item, result);
                break;
            case ITEM_TYPE_EXTERNAL:
                /* external checks use their own timeouts */
                res = get_value_external(item, result);
                break;
            case ITEM_TYPE_SSH:
    #ifdef HAVE_SSH2
                alarm(CONFIG_TIMEOUT);
                res = get_value_ssh(item, result);
                alarm(0);
    #else
                SET_MSG_RESULT(result, zbx_strdup(NULL, "Support for SSH checks was not compiled in."))
                res = CONFIG_ERROR;
    #endif
                break;
            case ITEM_TYPE_TELNET:
                alarm(CONFIG_TIMEOUT);
                res = get_value_telnet(item, result);
                alarm(0);
                break;
            case ITEM_TYPE_CALCUATED:
                res = get_value_calculated(item, result);
                break;
            default:
                SET_MSG_RESULT(result, zbx_dsprintf(NULL, "Not supported item type:%d", item->type));
                res = CONFIG_ERROR;
        }
        
        if (SUCCEED != res)
        {
            if (!ISSET_MSG(result))
                SET_MSG_RESULT(result, zbx_strdup(NULL, ZBX_NOTSUPPORTED));

            zabbix_log(LOG_LEVEL_DEBUG, "Item [%s:%s] error: %s", item->host.host, item->key_orig, result->msg);
        }

        zabbix_log(LOG_LEVEL_DEBUG, "End of %s():%s", __function_name, zbx_result_string(res));

        return res;
    }
    ```

函数```get_value_agent```会通过Zabbix agent获取```item```的值，并将结果存入```result```中。

* Checks_agent.c ==> int get_value_agent(DC_ITEM *item, AGENT_RESULT *result)

    ```
    int get_value_agent(DC_ITEM *item, AGENT_RESULT *result)
    {
        const char *__function_name = "get_value_agent";
        zbx_sock_t s;
        char *buf, buffer[MAX_STRING_LEN];
        int ret = SUCCEED;
        ssize_t received_len;

        zabbix_log(LOG_LEVEL_DEBUG, "In %s() host:'%s' addr:'%s' key:'%s'",
                __function_name, item->host.host, item->interface.addr, item->key);

        //使用函数zbx_tcp_connect连接两个ip: CONFIG_SOURCE_IP与item->interface.addr:item->interface.port。
        if (SUCCEED == (ret = zbx_tcp_connect(&s, CONFIG_SOURCE_IP, item->interface.addr, item->interface.port, 0)))
        {
            zbx_snpritf(buffer, sizeof(buffer), "%s\n", item->key);
            zabbix_log(LOG_LEVEL_DEBUG, "Sending [%s]", buffer);

            /* send requests using old protocol */
            //使用函数zbx_tcp_send_raw向被监控设备发送数据
            if (SUCCEED != zbx_tcp_send_raw(&s, buffer))
                ret = NETWORK_ERROR;
            //使用函数zbx_tcp_recv_ext从被监控设备获取数据
            else if (FAIL != (received_len = zbx_tcp_recv_ext(&s, &buf, ZBX_TCP_READ_UNTIL_CLOSE,0)))
                ret = SUCCEED;
            else
                ret = TIMEOUT_ERROR;
        }
        else
            ret = NETWORK_ERROR;

        ...

        zbx_tcp_close(&s);

        zabbix_log(LOG_LEVEL_DEBUG, "End of %s():%s", __function_name, zbx_result_string(ret));

        return ret;
    }
    ```

### process item values

process item values又可以分为两个步骤：1，将获取到的数据存入```item_values```中；2，将```item_values```中的数据存入```cache```中。

函数```dc_add_history```会根据```item```的```value_type```的不同，调用不同的函数向```item_values```中存入数据。

* Dbcache.c ==> void dc_add_history(zbx_uint64_t itemid, unsigned char value_type, unsigned char flags, AGENT_RESULT *value, zbx_timespec_t *ts, unsigned char state, const char *error)

    ```
    void dc_add_history(zbx_uint64_t itemid, unsigned char value_type, unsigned char flags, AGENT_RESULT *value, zbx_timespec_t *ts, unsigned char state, const char *error)
    {
        ...

        switch (value_type)
        {
            //根据item的value_type的不同，调用不同的函数向item_values中存入数据。
            //当item的value_type为ITEM_VALUE_TYPE_FLOAT时，调用函数dc_local_add_history向item_values中存入数据。
            case ITEM_VALUE_TYPE_FLOAT:
                if (GET_DBL_RESULT(value))
                    dc_local_add_history_dbl(itemid, ts, value->dbl);
                break;
            case ITEM_VALUE_TYPE_UINT64:
                if (GET_UI64_RESULT(value))
                    dc_local_add_history_uint(itemid, ts, value->ui64);
                break;
            case ITEM_VALUE_TYPE_STR:
                if (GET_STR_RESULT(value)
                    dc_local_add_history_text(itemid, ts, value->str);
                break;
            case ITEM_VALUE_TYPE_TEXT:
                if (GET_TEXT_RESULT(value))
                    dc_local_add_history_text(itemid, ts, value->text);
                break;
            case ITEM_VALUE_TYPE_LOG:
                if (GET_LOG_RESULT(value))
                {
                    size_t i;
                    zbx_log_t *log;

                    for (i = 0; NULL != value->logs[i]; i++)
                    {
                        log = value->logs[i];

                        dc_local_add_history_log(itemid, ts, log->value, log->timestamp, log->source,
                                log->severity, log->logeventid, log->lastlogsize, log->mtime);
                    }
                }
                break;
            default:
                zabbix_log(LOG_LEVEL_ERR, "unknown value type [%d] for itemid [" ZBX_FS_UI64 "]",
                        value_type, itemid);
                return;
        }
    }
    ```

函数```dc_local_add_history_dbl```先从```item_values```中获取```item_value```，然后填写```item_value```中的某些字段。

* static void dc_local_add_history_dbl(zbx_uint64_t itemid, zbx_timespec_t *ts, double value_orig)

    ```
    static void dc_local_add_history_dbl(zbx_uint64_t itemid, zbx_timespec_t *ts, double value_orig)
    {
        dc_item_value_t *item_value;
        
        //从item_values中获取item_value
        item_value = dc_local_get_history_slot();

        item_value->itemid = itemid;
        item_value->ts = *ts;
        item_value->value_type = ITEM_VALUE_TYPE_FLOAT;
        item_value->flags = 0;
        item_value->value.value_dbl = value_orig;
    }
    ```

函数```dc_local_get_history_slot```具体实现从```item_values```中获取```item_value```。

* static dc_item_value_t *dc_local_get_history_slot()

    ```
    static dc_item_value_t *dc_local_get_history_slot()
    {
        if (ZBX_MAX_VALUES_LOCAL == item_values_num)
            dc_flush_history();

        while (item_values_alloc == item_values_num)
        {
            item_values_alloc += ZBX_STRUCT_REALLOC_STEP;
            item_values = zbx_realloc(item_values, item_values_alloc *sizeof(dc_item_value_t));
        }

        return &item_values[item_values_num++];
    }
    ```


函数```dc_flush_history```会将```item_values```中的数据存入```cache```中。

* Dbcache.c ==> void dc_flush_history()

    ```
    void dc_flush_history()
    {
        size_t i;
        dc_item_value_t *item_value;

        if (0 == item_values_num)
            return;

        LOCK_CACHE;

        for (i = 0; i < item_values_num; i++)
        {
            //从item_values中获取item_value
            item_value = &item_values[i];

            if (ITEM_STATE_NOTSUPPORTED == item_value->state)
            {
                DCadd_history_notsupported(item_value);
            }
            else if (0 != (ZBX_FLAG_DISCOVERY_RULE & item_value->flags))
            {
                DCadd_history_lld(item_value);
            }
            else
            {
                switch (item_value->value_type)
                {
                    //根据item_value->value_type的不同调用不同的函数，将item_value中的数据存入cache。
                    //当item_value->value_type为ITEM_VALUE_TYPE_FLOAT时，调用函数DCadd_history_dbl。
                    case ITEM_VALUE_TYPE_FLOAT:
                        DCadd_history_dbl(item_value);
                        break;
                    case ITEM_VALUE_TYPE_UINT64:
                        DCadd_history_uint(item_value);
                        break;
                    case ITEM_VALUE_TYPE_STR:
                        DCadd_history_str(item_value);
                    case ITEM_VALUE_TYPE_TEXT:
                        DCadd_history_text(item_value);
                        break;
                    case ITEM_VALUE_TYPE_LOG:
                        DCadd_history_log(item_value);
                        break;
                }
            }
        }

        UNLOCK_CACHE;

        item_values_num = string_values_offset = 0;
    }
    ```

当```item_value->value_type```为```ITEM_VALUE_TYPE_FLAOT```时，会调用函数```DCadd_history_dbl```将某```item_value```中的数据存入```cache```中。

* Dbcache.c ==> static void DCadd_history_dbl(dc_item_value_t *value)

    ```
    static void DCadd_history_dbl(dc_item_value_t * value)
    {
        ZBX_DC_HISTORY *history;

        DCcheck_ns(&value->ts);

        //调用函数DCget_history_ptr获取cache中history的一个指针。
        history = DCget_history_ptr(0);

        history->itemid = value->itemid;
        history->ts = value->ts;
        history->state = ITEM_STATE_NORMAL;
        history->value_type = ITEM_VALUE_TYPE_FLOAT;
        history->value_orig.dbl = value->value.value_dbl;
        history->value.dbl = 0;
        history->value_null = 0;

        cache->stats.history_counter++;
        cache->stats.history_float_counter++;
    }
    ```


函数```DCget_history_ptr```从```cache```中获取```history```的一个指针。

* Dbcache.c ==> static ZBX_DC_HISTORY *DCget_history_ptr(size_t text_len)

    ```
    static ZBX_DC_HISTORY * DCget_history_ptr(size_t text_len)
    {
        ZBX_DC_HISTORY *history;
        int f;
        size_t free_len;
    retry:
        if (cache->history_num == ZBX_HISTORY_SIZE)
        {
            //没明白？
            DCvacuum_history();

            if (cache->history_num == ZBX_HISTORY_SIZE)
            {
                UNLOCK_CACHE;

                zabbix_log(LOG_LEVEL_DEBUG, "History buffer is full. Sleeping for 1 second.");
                sleep(1);

                LOCK_CACHE;

                goto retry;
            }
        }

        if (0 != text_len)
        {
            if (text_len > CONFIG_TEXT_CACHE_SIZE)
            {
                zabbix_log(LOG_LEVEL_ERR, "insufficient shard memory for text cache");
                exit(-1);
            }

            free_len = CONFIG_TEXT_CACHE_SIZE - (cache->last_text - cache->text);

            if (text_len > free_len)
            {
                DCvacuum_text();

                free_len = CONFIG_TEXT_CACHE_SIZE - (cache->last_text - cache->text);
                if (text_len > free_len)
                {
                    UNLOCK_CACHE;

                    zabbix_log(LOG_LEVEL_DEBUG, "History text buffer is full. Sleeping for 1 second.");
                    sleep(1);

                    LOCK_CACHE;

                    goto retry;
                }
            }
        }

        if (ZBX_HISTORY_SIZE <= (f = (cache->history_first + cache->history_num)))
            f -= ZBX_HISTORY_SIZE;
        history = &cache->history[f];
        history->num = 1;
        history->keep_history = 0;
        history->keep_trends = 0;

        cache->history_num++;

        return history;
    }
    ```

函数DCvacuum_history的作用未理清。

* Dbcache.c ==> static void DCvacuum_history()

    ```
    static void DCvacuum_history()
    {
        const char *__function_name = "DCvacuum_history";
        int n, f, n_gap = 0, n_data = 0;

        zabbix_log(LOG_LEVEL_DEBUG, "In %s() history_gap_num:%d/%d",
                __function_name, cache->history_gap_num, ZBX_HISTORY_SIZE);

        if (ZBX_HISTORY_SIZE / 100 >= cache->history_gap_num)
            goto exit;

        if (ZBX_HISTORY_SIZE <= (f = cache->history_first + cache->history_num))
            f -= ZBX_HISTORY_SIZE;

        for (n = cache->history_num; 0 < n; n--)
        {
            if (0 == f)
                f = ZBX_HISTORY_SIZE;
            f--;

            if (0 == cache->history[f].itemid)
            {
                if (0 != n_data)
                {
                    DCmove_history(f + 1, n_data, n_gap);
                    n_data = 0;
                }
                n_gap++;
            }
            else if (0 != n_gap)
            {
                n_data++;

                if (0 == f)
                {
                    DCmove_history(f, n_data, n_gap);
                    n_data = 0;
                }
            }
        }

        if (0 != n_data)
            DCmove_history(f, n_data, n_gap);

        cache->history_num -= n_gap;
        cache->history_gap_num -= n_gap;
        if (ZBX_HISTORY_SIZE <= (cache->history_first += n_gap))
            cache->history_first -= ZBX_HISTORY_SIZE;
    exit:
        zabbix_log(LOG_LEVEL_DEBUG, "End of %s()", __function_name);
    }
    ```
