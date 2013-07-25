:Date: 2011-04-05 09:12:09.812678638-09:00
:Author: Shane R. Spencer <shane@bogomip.com>

Django + Redis (master/slave) = Pseudo distributed event based caching
======================================================================
.. sectionauthor:: Shane R. Spencer <shane@bogomip.com>

This article focuses on the following topology to address usingRedis as the cache 
backend for Django where sets and deletes aresent to a master Redis server along with a 
local or dedicated Redisserver for a set of or single cluster member.

    - 'Cluster Slave 1' <- 'Cluster Master 1'
    
    - 'Cluster Slave 2' <- 'Cluster Master 1'

    - 'Cluster Slave 3' <- 'Cluster Master 1'

    - 'Django Node 1' uses 'Cluster Slave 1' as 'default' and Cluster Master 1 as 'master'

    - 'Django Node 2' uses 'Cluster Slave 2' as 'default' and Cluster Master 1 as 'master'

    - 'Django Node 3' uses 'Cluster Slave 3' as 'default' and Cluster Master 1 as 'master'

All updates from 'Cluster Master 1' are sent to all slaves as quickly as possible, not 
atomic, not verified. This requires you to expect slave cache keys to timeout 
appropriately as a backup.

You should note that using this method does not necessarilyaugment the standard of a 
memory cache.

    - Store/Fetch/Invalidate Keys

    - Keys can be deleted at any time

    - The application is ultimately responsible for understanding cache behaviour

    - Caches in a cluster may hold different values for a key

Number 4 is really only ever seen in a replicated or isolated environment where the 
backend is a database slave, a database in a master/master relationship, redis, or when 
using many different local caches per cluster. In the setup I am testing there is a 
single local slave at '10.10.0.41' and a single master at '10.10.0.40'. The redis 
servers are already configured, here is the Django cache configuraiton.

.. code-block:: python
    :linenos:

    CACHES = {
        'default': {'BACKEND': 'redis_cache.RedisCache', 'LOCATION': '10.10.0.41:6379',},
        'master': {'BACKEND': 'redis_cache.RedisCache', 'LOCATION': '10.10.0.40:6379',},
    }

And lets do a little testing:

.. code-block:: python
    :linenos:

    >>> from django.core.cache import get_cache
    >>> slave = get_cache('default')
    >>> master = get_cache('master')
    >>> master.set('hi','there')
    True
    >>> slave.get('hi')
    'there'
    Huzzah.. it replicated!
    >>> master.delete('hi')
    >>> slave.get('hi')
    Huzzah.. it deleted!
    >>> master.set('hi','there')
    True
    >>> slave.set('hi','pony')
    True
    >>> master.get('hi')
    'there'
    >>> slave.get('hi')
    'pony'
    Huzzah.. it can be different!
    >>> master.delete('hi')
    >>> master.get('hi')
    >>> slave.get('hi')
    Huzzah.. even though there were differences I can delete the key!

What this means to me is that I can use the master for all changes and isolate all 
lookups to the slave server and if I'm feeling frisky I can even update the slave 
server. Keep in mind that with Redis and the Django-Redis-Cache module you can specify a 
different namespace or rather database ID making the following possible.

.. code-block:: python
    :linenos:

    CACHES = {
        'default': { 'BACKEND': 'redis_cache.RedisCache', 'LOCATION': '10.10.0.41:6379', 'OPTIONS': { 'DB': 1, }, },
        'master': { 'BACKEND': 'redis_cache.RedisCache', 'LOCATION': '10.10.0.40:6379', 'OPTIONS': { 'DB': 1, }, },
        'local': { 'BACKEND': 'redis_cache.RedisCache', 'LOCATION': '10.10.0.41:6379', 'OPTIONS': { 'DB': 2, }, },
    }

Huzzah.

The idea behind all of this is to allow the master to fail while the slave continues 
operating. In a hash based distribution that is possible, however whenever a cache 
member leaves there is no persistance for what was once cached without hacking the 
distributor to deal with n+1 replication of events. Once a hash member is alive again 
all keys that were reallocated to a different member are invalidated simply because the 
distributor won't think to look there any more unless the same member fails again.. at 
which point you are seeing old data that you couldn't possible expire using 'delete' 
since you didn't know it existed. This makes deleting data in a hash based distribution 
very clumsy.

The above scenario works best if you set a key on the slave first and then send it to 
the master. This allows the Django cluster node to have immediate access to data that it 
is attempting to persist into memory between request which is assumed to be the most 
appropriate node (given an intelligent entry distribution for the request/requestor 
pairs). Now the master can fail and come back up without too much of a problem. You can 
take it one step further and store a local set of keys (command queues would be a lot of 
trouble, just use keys) for master updates that never made it for when it comes back 
online and needs to be updated. Redis can be set up using heartbeat and DRBD to mitigate 
the down time of a Redis master as well as supports master/master mode by way of slaving 
eachother. Redis can store the database to disk which is great. You can write a quick 
program to replay the master cache to a new slave if you're using dynamic nodes since 
redis supports key lookups. You can even have the slave store values only in memory 
while the master uses the disk as well. Lots of options.

Comments
--------

.. disqus:: techbitsgithub redispimping

