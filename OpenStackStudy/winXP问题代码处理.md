## 在客户端修改获取内存的代码
    ssh root@172.16.19.235
### 进入目录 
    cd /usr/local/nagios/libexec/hty_monitor_client
### vi get_vm_usage
``` python
def check_mem_usage(uuid):
    mem_used, mem_total,mem_rss = _get_mem_usage(uuid)
    if mem_used < 0:
        mem_used = 0 - mem_used
        mem_total = 0 - mem_total
        mem_rss = 0 - mem_rss

    result_dict = {'u': str(mem_used),
                   't': str(mem_total),
                   'r': str(mem_rss) }
    message = '%(u)s;%(t)s;%(r)s;' % result_dict
    print message
```
返回时多一个rss的量
```python
def _get_mem_usage(uuid):
    connection = _get_connection()
    domain = _get_domain_not_shut_off(connection, uuid)
    memory_stats = domain.memoryStats()
    memory_used = 0
    memory_avai = 0
    if (memory_stats and
            memory_stats.get('available') and
            memory_stats.get('unused') and
            memory_stats.get('rss')):
        memory_avai = memory_stats.get('available')
        memory_unused = memory_stats.get('unused')
        memory_used = memory_avai - memory_unused
        memory_rss = memory_stats.get('rss')
    elif (memory_stats and
            memory_stats.get('actual')):
        # domain.setMemory(1)
        # time.sleep(2)
        tree = etree.fromstring(domain.XMLDesc(0))

        # domain.setMemory(memory_avai)
        # actual = memory_stats.get('actual')
        rss = memory_stats.get('rss')
        # if rss < actual * 0.99:
        #     memory_used = rss
        #     memory_avai = 0 - memory_avai
        memory_used = 0 - rss
        memory_avai = 0 - int(tree.find('memory').text)
        # if memory_used == memory_avai:
        #     memory_used = domain.memoryStats().get('rss')
    _close_connection()
    return memory_used, memory_avai, memory_rss
```
### pdb后台调试命令
    ./get_vm_usage -t mem 67c3083e-5f5d-45ab-beb7-63e43eda6fe7
## 在服务端修改
    ssh root@172.16.19.238
### 进入目录
    cd /usr/local/nagios/libexec/hty_monitor_server/
### vi check_vm_mem(修改部分)
```python
 rss = float(data[2])
    mem_usage = int(float(used) / float(total) * 100)
    warning, critical = utils.get_w_and_c(hostname, 'mem')
    warning = float(warning)
    critical = float(critical)
    if mem_usage == 0:
       #windowsXP mem_total>=3G of operation
        result = {'usage': str(mem_usage),
                  'rss': str(rss), }
        message = 'Memory %(state)s: %(usage)s |mem_rss=%(rss)s;;;;'
    else:
        result = {'usage': str(mem_usage),
                  'used': str(used),
                  'total': str(total), }
        message = 'Memory %(state)s: %(usage)s %% used'\
            + '|mem_used=%(used)s;;;;mem_total=%(total)s;;;;'
    state = utils.check_state(mem_usage, warning, critical, message, result)
    utils.store_check_state(hostname, 'mem', state)
    sys.exit(state)

```
### pdb后台调试命令
    ./check_vm_mem 67c3083e-5f5d-45ab-beb7-63e43eda6fe7 172.16.19.236