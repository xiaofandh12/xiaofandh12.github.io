---
layout: post
title: Zabbix Server源码分析之main_dbconfig_loop()
description: Zabbix Server, main_dbconfig_loop(), 柔性数组
category: blog
---

## 文档

* [Zabbix2.2源码](http://www.zabbix.com/download.php)
* [Zabbix2.2 Doc](https://www.zabbix.com/documentation/2.2/)

## 相关数据结构

进程```main_dbconfig_loop()```的作用是将某些数据库中数据导入内存中，以便后面可以更快速、方面的使用这些数据；具体来说就是将数据库中的数据查询出来，并存入结构体```ZBX_DC_CONFIG```中，```ZBX_DC_CONFIG```的定义如下所示：

* ZBX_DC_CONFIG

    ```
    typedef struct
    {
        zbx_hashset_t   items;
        zbx_hashset_t   items_hk;
        zbx_hashset_t   numitems;
        zbx_hashset_t   snmpitems;
        zbx_hashset_t   ipmiitems;
        zbx_hashset_t   flexitems;
        zbx_hashset_t   trapitems;
        zbx_hashset_t   logitems;
        zbx_hashset_t   dbitems;
        zbx_hashset_t   sshitems;
        zbx_hashset_t   telnetitems;
        zbx_hashset_t   simpleitems;
        zbx_hashset_t   jmxitems;
        zbx_hashset_t   calcitems;
        zbx_hashset_t   deltaitems;
        zbx_hashset_t   functions;
        zbx_hashset_t   triggers;
        zbx_hashset_t   trigdeps;
        zbx_vector_ptr_t    *time_triggers;
        zbx_hashset_t   hosts;
        zbx_hashset_t   hosts_h;
        zbx_hashset_t   proxies;
        zbx_hashset_t   host_inventories;
        zbx_hashset_t   ipmihosts;
        zbx_hashset_t   htmpls;
        zbx_hashset_t   gmacros;
        zbx_hashset_t   gmacros_m;
        zbx_hashset_t   hmacros;
        zbx_hashset_t   hmacros_hm;
        zbx_hashset_t   interfaces;
        zbx_hashset_t   interfaces_ht;
        zbx_hashset_t   interfaces_snmpaddrs;
        zbx_hashset_t   interfaces_snmpitems;
        zbx_hashset_t   regexps;
        zbx_hashset_t   expressions;
        zbx_binary_heap_t   queues[ZBX_POLLER_TYPE_COUNT]
        zbx_binary_heap_t pqueue;
        ZBX_DC_CONFIG_TABLE *config;
    }
    ZBX_DC_CONFIG;

    static ZBX_DC_CONFIG *config = NULL;
    ```

在定义完结构体```ZBX_DC_CONFIG```后，初始化了一个类型为```ZBX_DC_CONFIG```的全局变量```config```。

在```ZBX_DC_CONFIG```中，很多变量都定义为```zbx_hashset_t```，```zbx_hashset_t```是一个结构体，其中包括用于存储数据的HASH链表的入口```ZBX_HASHSET_ENTRY_T **slots```、存储数据时需要用到的函数、int常量，它的定义如下：

* zbx_hashset_t

    ```
    typedef struct
    {
        ZBX_HASHSET_ENTRY_T **slots;
        int num_slots;
        int num_data;
        zbx_hash_func_t hash_func;
        zbx_compare_func_t compare_func;
        zbx_clean_func_t clean_func;
        zbx_mem_malloc_func_t mem_malloc_func;
        zbx_mem_realloc_func_t mem_realloc_func;
        zbx_mem_free_func_t mem_free_func;
    }
    zbx_hashset_t;
    ```

```zbx_hashset_t```中，除有Hash表相关的一些函数和变量外，还有```ZBX_HASHSET_ENTRY_T **slots```，即用来保存数据的Hash链表，数据就是存储在```ZBX_HASHSET_ENTRY_T```中的data字段，```ZBX_HASHSET_ENTRY_T```的定义如下：

* ZBX_HASHSET_ENTRY_T

    ```
    #define ZBX_HASHSET_ENTRY_T struct zbx_hashset_entry_s

    ZBX_HASHSET_ENTRY_T
    {
        ZBX_HASHSET_ENTRY_T *next;
        zbx_hash_t hash;
    #if SIZEOF_VOID_P > 4
        char padding[sizeof(void *) - sizeof(zbx_hash_t)];
    #endif
        char data[1];
    };
    ```

```ZBX_HASHSET_ENTRY_T```中的data字段为一个柔性数组，关于柔性数组我会在本文的最后做相关介绍。

本文主要通过讲述如何把数据库hosts表格中的数据查询出来，并存入```config-->hosts```中的过程，来说明进程```main_dbconfig_loop()```的工作原理。hosts表格中的数据是存于一Hash链表中的，在此Hash链表中存储的数据类型为```ZBX_DC_HOST```，```ZBX_DC_HOST```的定义如下：                                                                                                                             

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
        int snmp_disable_until;
        int ipmi_errors_from;
        int ipmi_disable_until;
        int jmx_errors_from;
        int jmx_disable_until;
        unsigned char maintenance_status;
        unsigned char maintenance_type;
        unsigned char abailable;
        unsigned char snmp_available;
        unsigned char ipmi_available;
        unsigned char jmx_abailable;
        unsigned char status;
    }
    ZBX_DC_HOST;
    ```

## MAIN_ZABBIX_ENTRY()

```MAIN_ZABBIX_ENTRY()```函数首先做一些初始化工作，然后创建Zabbix Server端各进程，当然包括```main_dbconfig_loop()```。

* MAIN_ZABBIX_ENTRY()

    ```
    ...
    init_configuration_cache();
    ...
    DCsync_configuration();
    ...
    else if (server_num <= (server_conut += CONFIG_CONFSYNCER_FORKS))
    {
        INIT_SERVER(ZBX_PROCESS_TYPE_CONFSYNCER, CONFIG_CONFSYNCER_FORKS);
        main_dbconfig_loop();
    }
    ...
    ```

## init_configuration_cache()

在```init_configuration_cache()```函数中，通过调用```CREATE_HASHSET(config->hosts)```，来初始化```config->hosts```中的相关的Hash函数。```init_configuration_cache()```函数的定义大致如下：

* init_configuration_cache()

    ```
    void init_configuration_cache()
    {
        ...
        #define INIT_HASHSET_SIZE 1000
        #define CREATE_HASHSET(hashset) CREATE_HASHSET_EXT(hashset, ZBX_DEFAULT_UINT64_HASH_FUNC, ZBX_DEFAULT_UINT64_COMPARE_FUNC)
        #define CREATE_HASHSET_EXT(hashset, hash_func, compare_func) zbx_hashset_create_ext(&hashset, INIT_HASHSET_SIZE, hash_func, compare_func, NULL, __config_mem_malloc_func, __config_mem_realloc_func, __config_mem_free_func)
        ...
        CREATE_HASHSET(config->hosts);
        ...
    }
    ```

```CREATE_HASHSET(config->hosts)```是通过调用函数```zbx_hashset_create_ext()```来初始化```config->hosts```，函数```zbx_hashset_create_ext()```的代码如下所示：

* zbx_hashset_create_ext()

    ```
    void zbx_hashset_create_ext(zbx_hashset_t *hs, size_t init_size,
                zbx_hash_func_t hash_func,
                zbx_compare_func_t compare_func,
                zbx_clean_func_t clean_func,
                zbx_mem_malloc_func_t mem_malloc_func,
                zbx_mem_realloc_func_t mem_realloc_func,
                zbx_mem_free_func_t mem_free_func)
    {
        int nslots = next_prime(init_size);

        if (NULL == (hs->slots = mem_malloc_func(NULL, nslots * sizeof(ZBX_HASHSET_ENTRY_T *))))
            return;

        hs->num_data = 0;
        hs->num_slots = nslots;

        memset(hs->slots, 0, hs->num_slots * sizeof(ZBX_HASHSET_ENTRY_T *));

        hs->hash_func = hash_func;
        hs->compare_func = compare_func;
        hs->clean_func = clean_func;
        hs->mem_malloc_func = mem_malloc_func;
        hs->mem_realloc_func = mem_realloc_func;
        hs->mem_free_func = mem_free_func;
    }
    ```

## main_dbconfig_loop()

进程```main_dbconfig_loop()```每隔```CONFIG_CONFSYNCER_FREQUENCY```秒，就调用函数```DCsync_configuration()```将数据库中的数据同步到内存中，```main_dbconfig_loop()```的代码如下所示：

* main_dbconfig_loop()

    ```
    void main_dbconfig_loop(void)
    {
        double sec = 0.0;

        zbx_setproctitle("%s [waiting %d sed for processes]",getprocess_type_string(process_type), CONFIG_CONFSYNCER_FREQUENCY);

        zbx_sleep_loop(CONFIG_CONFSYNCER_FREQUENCY);

        zbx_setproctitle("%s [connecting to the database]", get_process_type_string(process_type));

        DBconnect(ZBX_DB_CONNECT_NORMAL);

        for (;;)
        {
            zbx_setproctitle("%s [synced configuration in " ZBX_FS_DBL "sec, syncing configuration]", get_process_type_string(process_type), sec);
            
            sec = zbx_time();
            DCsync_configuration();
            sec = zbx_time() - sec;

            zbx_setproctitle("%s [synced configuration in " ZBX_FS_DBL "sec, idle %d sec]", get_process_type_string(process_type), sec, CONFIG_CONFSYNCER_FREQUENCY);

            zbx_sleep_loop(CONFIG_CONFSYNCER_FREQUENCY);
        }
    }
    ```

## DCsync_configuration()

```MAIN_ZABBIX_ENTRY```的初始化工作中和进程```main_dbconfig_loop()```的循环中，都调用了函数DCsync_configuration()，函数DCsync_configuration会从相应的表格中查询出结果，并将结果传入相应的函数做处理，如从hosts表格中查询得到```host_result```，再将```host_result```传入函数```DCsync_hosts()```做进一步处理。函数```DCsync_configuration()```的部分源码如下所示：

* DCsync_configuration()

    ```
    void DCsync_configuration(void)
    {
        ...
        DB_RESULT host_result = NULL;
        ...

        sec = zbx_time();
        if (NULL == (host_result = DBselect(
                "select hostid,proxy_hostid,host,ipmi_authtype,ipmi_privilege,ipmi_username,"
                    "ipmi_password,maintenance_status,maintenance_type,maintenance_from,"
                    "errors_from,available,disable_until,snmp_errors_from,"
                    "snmp_available,snmp_disable_until,ipmi_errors_from,ipmi_available,"
                    "ipmi_disable_until,jmx_errors_from,jmx_available,jmx_disable_until,"
                    "status,name"
                "from hosts"
                "where status in (%d,%d,%d,%d)"
                    " and flags<>%d"
                    ZBX_SQL_NODE,
                HOST_STATUS_MONITORED, HOST_STATUS_NOT_MONITORED,
                HOST_STATUS_PROXY_ACTIVE, HOST_STATUS_PROXY_PASSIVE,
                ZBX_FLAG_DISCOVERY_PROTOTYPE,
                DBand_node_local("hostid"))))
        {
            goto out;
        }
        hsec = zbx_time() - sec;

        ...

        START_SYNC;

        ...

        sec = zbx_time();
        DCsync_hosts(host_result);
        hsec2 = zbx_time() - sec;

        ...

        FINISH_SYNC;
    out:
        ...
        DBfree_result(host_result);
        ...
    }
    ```

## DCsync_hosts()

在函数```DCsync_hosts()```中，首先定义```host```为一指向```ZBX_DC_HOST```的指针，然后调用函数```DCfind_id()```获取一指向```ZBX_HASHSET_ENTRY_T->data```的指针并赋值给```host```，最后调用函数```DBfetch(host_result)```获得hosts表格中的一行，并将数据赋值给```host```中的各个字段（如```host->proxy_hostid,host->host,host->name,host->maintenance_status,host->maintenance_type,host->maintenance_from```等）。
函数```DCsync_hosts()```的部分源码如下所示：

* DCsync_hosts()
    
    ```
    static void DCsync_hosts(DB_RESULT result)
    {
        const char  *__function_name = "DCsync_hosts";

        DB_ROW  row;

        ZBX_DC_HOST *host;
        ZBX_DC_IPMIHOST *ipmihost;
        ZBX_DC_PROXY *proxy;
        ZBX_DC_HOST_H   *host_h,host_h_local;

        int found;
        int update_index;
        zbx_uint64_t    hostid,proxy_hostid;
        zbx_vector_uint64_t ids;
        zbx_hashset_iter_t  iter;
        unsigned char   status;
        time_t  now;
        signed char ipmi_authtype;
        unsigned char   ipmi_privilege;

        zabbix_log(LOG_LEVEL_DEBUG, "In %s()", __function_name);

        zbx_vector_uint64_create(&ids);
        zbx_vector_uint64_reserve(&ids, config->hosts.num_data + 32);

        now = time(NULL);

        while (NULL != (row = DBfetch(result)))
        {
            ZBX_STR2UINT64(hostid, row[0]);
            ZBX_DBROW@UINT64(proxy_hostid, row[1]);
            ZBX_STR2UCHAR(status, row[22]);

            zbx_vector_uint64_append(&ids, hostid);

            host = DCfind_id(&config->hosts, hostid, sizeof(ZBX_DC_HOST), &found);

            update_index = 0;

            if ((HOST_STATUS_MONITORED == status || HOST_STATUS_NOT_MONITORED == status) &&
                    (0 == found || 0 != strcmp(host->host, row[2])))
            {
                if (1 == found)
                {
                    host_h_local.host = host->host;
                    host_h = zbx_hashset_search(&config->hosts_h,&host_h_local);

                    if (NULL != host_h && host == host_h->host_ptr)
                    {
                        zbx_strpool_release(host_h->host);
                        zbx_hashset_remove(&config->hosts_h, &host_h_local);
                    }
                }
                host_h_local.host = row[2];
                host_h = zbx_hashset_search(&config->hosts_h, &host_h_local);

                if (NULL != host_h)
                    host_h->host_ptr = host;
                else
                    update_index = 1;
            }

            host->proxy_hostid = proxy_hostid;
            DCstrpool_replace(found, &host->host, row[2]);
            DCstrpool_replace(found, &host->name, row[23]);

            if (0 == found)
            {
                host->maintenance_status = (unsigned char)atoi(row[7]);
                host->maintenance_type = (unsigned char)atoi(row[8]);
                host->maintenance_from = atoi(row[9]);
                host->data_expected_from = now;

                host->errors_from = atoi(row[10]);
                host->available = (unsigned char)atoi(row[11]);
                host->disable_until = atoi(row[12]);
                host->snmp_errors_from  = atoi(row[13]);
                host->snmp_available = (unsigned char)atoi(row[14]);
                host->snmp_disable_until = atoi(row[15]);
                host->ipmi_errors_from = atoi(row[16]);
                host->ipmi_available = (unsigned char)atoi(row[17]);
                host->ipmi_disable_until = atoi(row[18]);
                host->jmx_errors_from = atoi(row[18]);
                host->jmx_available = (unsigned char)atoi(row[20]);
                host->jmx_disable_until = atoi(row[21]);
            }
            else
            {
                if (HOST_STATUS_MONITORED == status && HOST_STATUS_MONITORED != host->status)
                    host->data_expected_from = now;
            }

            if (1 == update_index)
            {
                host_h_local.host = zbx_strpool_acquire(host->host);
                host_h_local.host_ptr = host;
                zbx_hashset_insert(&config->hosts_h, &host_h_local, sizeof(ZBX_DC_HOST_H));
            }

            ...
        }
    }

    ```

问题：```host = DCfind_id(&config->hosts, hostid, sizeof(ZBX_DB_HOST), &found)```,```host```是一个指向```ZBX_DC_HOST```的指针，而函数```DCfind_id()```定义时返回的是指向```void```的指针，```DCfind_id```中```ptr```定义为指向```void```的指针，而```zbx_hashset_search()```或```zbx_hashset_insert()```中返回的```entry->data```的定义为```char data[1]```。
关于这个问题在后面__柔性数组__这个章节有所解释。

## DCfind_id()

以上面的函数调用形式（```host = DCfind_id(&config->host, hostid, sizeof(ZBX_DB_HOST),
&found)```）进行分析，函数```DCfind_id()```调用函数```zbx_hashset_search()```在```&config->host```中搜索```hostid```，若找到就将```*found```置为1，并返回```zbx_hashset_search()```的返回值；若未找到就将```*found```置为0，并调用函数```zbx_hashset_insert()```在```&config->host```中插入```hostid```，返回```zbx_hashset_insert()```的返回值。
函数```DCfind_id()```的源码如下所示：

* DCfind_id()

    ```
    static void *DCfind_id(zbx_hashset_t *hashset, zbx_uint64_t id, size_t size, int *found)
    {
        void *ptr;
        zbx_uint64_t buffer[1024];

        if (NULL == (ptr = zbx_hashset_search(hashset, &id)))
        {
            *found = 0;

            buffer[0] = id;
            ptr = zbx_hashset_insert(hashset, &buffer[0], size);
        }
        else
            *found = 1;

        return ptr;
    }
    ```

函数```zbx_hashset_search()```和```zbx_hashset_insert()```的源码列于下方:

### zbx_hashset_search()

* zbx_hashset_search()
    
    ```
    void *zbx_hashset_search(zbx_hashset_t *hs, const void *data)
    {
        int slot;
        zbx_hash_t hash;
        ZBX_HASHSET_ENTRY_T *entry;

        hash = hs->hash_func(data);

        slot = hash % hs->num_slots;
        entry = hs->slots[slot];

        while (NULL != entry)
        {
            if (entry->hash == hash && hs->compare_func(entry->data, data) == 0)
                break;

            entry = entry->next;
        }

        return (NULL != entry ? entry->data : NULL);
    }
    ```

### zbx_hashset_insert()

* zbx_hashset_insert()

    ```
    void *zbx_hashset_insert(zbx_hashset_t *hs, const void *data, size_t size)
    {
        return zbx_hashset_insert_ext(hs, data, size, 0);
    }

    void *zbx_hashset_insert_ext(zbx_hashset_t *hs, const void *data, size_t size, size_t offset)
    {
        int slot;
        zbx_hash_t hash;
        ZBX_HASHSET_ENTRY_T *entry;

        hash = hs->hash_func(data);

        slot = hash % hs->num_slots;
        entry = hs->slots[slot];

        while (NULL != entry)
        {
            if (entry->hash == hash && hs->compare_func(entry->data, data) == 0)
                break;

            entry = entry->next;
        }

        if (NULL == entry)
        {
            if (hs->num_data + 1 >= hs->num_slots * CRIT_LOAD_FACTOR)
            {
                int inc_slots, new_slot;
                void *slots;
                ZBX_HASHSET_ENTRY_T **prev_next, *curr_entry, *tmp;

                inc_slots = next_prime(MAX(hs->num_slots + 1, hs->num_slots * SLOT_GROWTH_FACTOR));

                if (NULL == (slots = hs->mem_realloc_func(hs->slots, inc_slots * sizeof(ZBX_HASHSET_ENTRY_T *))))
                    return NULL;

                hs->slots = slots;

                memset(hs->slots + hs->num_slots, 0, (inc_slots - hs->num_slots) * sizeof(ZBX_HASHSET_ENTRY_T *));

                for (slot = 0; slot < hs->num_slots; slot++)
                {
                    prev_next = &hs->slots[slot];
                    curr_entry = hs->slots[slot];

                    while (NULL != curr_entry)
                    {
                        if (slot != (new_slot = curr_entry->hash % inc_slots))
                        {
                            tmp = curr_entry->next;
                            curr_entry->next = hs->slots[new_slot];
                            hs->slots[new_slot] = curr_entry;

                            *prev_next = tmp;
                            curr_entry = tmp;
                        }
                        else
                        {
                            prev_next = &curr_entry->next;
                            curr_entry = curr_entry->next;
                        }
                    }
                }

                hs->num_slots = inc_slots;

                slot = hash % hs->num_slots;
            }

            if (NULL == (entry = hs->mem_malloc_func(NULL, offsetof(ZBX_HASHSET_ENTRY_T, data) + size)))
                return NULL;

            memcpy((char *)entry->data + offset, (const char *)data + offset, size - offset);
            entry->hash = hash;
            entry->next = hs->slots[slot];
            hs->slots[slot] = entry;
            hs->num_data++;
        }

        return entry->data;
    }
    ```

## 柔性数组

在函数```DCsync_hosts()```中，重要的一步就是获得```host```指针，如下所示：

    ```
    ZBX_DC_HOST *host;
    host = DCfind_id(&config->hosts, hostid, sizeof(ZBX_DC_HOST), &found);
    ```

得到```host```指针后，再对```host```赋值，对此有两点疑问：

1，```host```为指向```ZBX_DC_HOST```的指针，函数```DCfind_id```返回的是指向```void```的指针，那么函数```DCfind_id```的值怎么可以直接赋给```host```呢？

2，跟踪函数```DCfind_id```，可以发现真正的返回值是由函数```zbx_hashset_search```或```zbx_hashset_insert```返回的，这两个函数的返回值又为```entry->data```，其中```entry```为一个指向```ZBX_HASHSET_ENTRY_T```的指针，而在```ZBX_HASHSET_ENTRY_T```中```data```的定义为```char data[1]```，```host```中这么多的字段在```entry->data```中怎么存储得下呢？

关于这两个疑问可以由C语言的两个特性来回答：

>> 1. ”标准表示一个void *类型的指针可以转换为其他任何类型的指针。但是，有些编译器，尤其是那些老式的编译器，可能要求你在转换时使用强制类型转换。“--《C和指针》p222 

>> 2. 柔性数组

下面用一个示例对柔性数组做进一步的解释，如示例中所示```EntryStruct```中的```c```即为柔性数组，尽管它的定义为```char c[1]```，后面我们却可以用它来存储类型为```TestStruct```的数据；
需要注意的是在为```entryStruct```分配内存时，需要分配```sizeof(EntryStruct) + sizeof(TestStruct) + 1```字节的内存，因为```entryStruct->c```将会用来存储类型为```TestStruct```的数据。

* 柔性数组示例

    ```
    //柔性数组示例
    #include <iostream>
    #include <malloc.h>
    using namespace std;

    /* 结构体EntryStruct的定义
     * EntryStruct中的c就是所谓的柔性数组
     */
    typedef struct EntryStruct
    {
        int i;
        double j;
        char c[1];
    }*pEntryStruct;

    //结构体TestStruct的定义
    typedef struct TestStruct
    {
        int i;
        char c;
    }*pTestStruct;
    
    int main()
    {
        /*
         *使用函数malloc为entryStruct分配内存，这里需要对函数malloc的返回值做强制类型转化；
         *《C和指针》第222页写道，”标准表示一个void *类型的指针可以转化为其它任何类型的指针“，
         * malloc函数返回的正是一个void *类型的指针，但在这里若不做强制类型转化将报错。
         * 另外注意，为entryStruct分配的内存的长度为sizeof(EntryStruct) + sizeof(TestStruct) + 1,
         * 因为在entryStruct中c将会存储一个TestStruct的数据。
         */
        pEntryStruct entryStruct = (pEntryStruct)malloc(sizeof(EntryStruct) + sizeof(TestStruct) + 1);

        //为entryStruct->i和entryStruct->j赋值
        if (NULL != entryStruct)
        {
            entryStruct->i = 1;
            entryStruct->j = 1.1;
        }

        //为entryStruct->c赋值
        pTestStruct testStruct;
        testStruct = (pTestStruct)entryStruct->c;
        testStruct->i = 2;
        testStruct->c = 'a';

        //打印结构体EntryStruct的定义
        cout << "Definition of EntryStruct:"
             << "\n    typedef struct EntryStruct"
             << "\n    {"
             << "\n        int i;"
             << "\n        double j;"
             << "\n        char c[1];"
             << "\n     };\n"
             << endl;

        //打印结构体TestStruct的定义
        cout << "Definition of TestStruct:"
             << "\n     typedef struct TestStruct"
             << "\n     {"
             << "\n         int i;"
             << "\n         char c;"
             << "\n     };\n"
             << endl;
        
        //打印entryStruct的值
        cout << "Value of entryStruct:"
             << "\n    entryStruct:"
             << "\n        i(int) = " << entryStruct->i
             << "\n        j(double) = " << entryStruct->j
             << "\n        c(pTestStruct) = {"
             << "\n            i(int) = " << ((pTestStruct)entryStruct->c)->i
             << "\n            c(char) = " << ((pTestStruct)entryStruct->c)->c
             << "\n          }"
             << endl;

        //释放entryStruct占用的内存空间
        free(entryStruct);
        return 0;
    }
    ```
