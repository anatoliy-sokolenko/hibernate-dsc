#Hibernate Dynamic SQL Cache Overview

Dynamic Sql Cache module for [Hibernate 4+](http://hibernate.org).

This project used by author in several projects some of them high loaded systems.
The reason to create it is inefficient Hibernate query caching mechanism:

1. HQL-cache including entity-collections cache, clearing on every UPDATE, INSERT or DELETE performed on any entity-table used in hql-query or entity-collection.
It nullifies the benefits of using HQL-cache and entity-collection cache in high loaded systems.
2. SQL-cache holds result of first query invocation only and holds it forever, there is no way to reset it.



Therefore idea to create dynamic updatable SQL-cache was born.
Query cache result updates on every INSERT or DELETE operation for entity-table used in SQL-query.
To take advantage of dynamic sql cache you should change your look towards sql-query creation, and use sql-queries instead of any entity-collections too:

1. Query should result only entity ID, because you can always load it from entity-cache by ID.
   (Two cache read operations much faster than database reading operation)
2. Use only immutable properties of entity as query parameters.               
3. Use sql-query with dynamic cache instead of any entity-collections.

Licensed under the Apache License 2.0.

### Features

* Compatible with any cache provider (EHCache, Infinispan, Hazelcast ...)
* Test cases included


#Usage example

This example uses "Spring Framework", but you can use project without it.

Somethere in hibernate.cfg.xml

<session-factory>

      ...

      <property name="hibernate.cache.query_cache_factory">com.corundumstudio.hibernate.dsc.DynamicQueryCacheFactory</property>

      <property name="hibernate.cache.region.factory_class">org.hibernate.cache.infinispan.InfinispanRegionFactory</property>
      <property name="hibernate.cache.use_second_level_cache">true</property>
      <property name="hibernate.cache.use_query_cache">true</property>

      ...

</session-factory>

Note: you can use any other cache factory not only org.hibernate.cache.infinispan.InfinispanRegionFactory


    @Configuration
    public class QueryCacheListenerConfig {

        @Bean
        public QueryCacheEntityListener createCacheListener() {
                return new QueryCacheEntityListener();		
        }
	
        @PostConstruct
        protected void init() {

                // register hibernate dynamic cache listener
                // QueryCacheEntityListener

                EventListenerRegistry registry = sessionFactory.getServiceRegistry().getService(EventListenerRegistry.class);
                registry.getEventListenerGroup(EventType.POST_UPDATE).appendListener(createCacheListener());
                registry.getEventListenerGroup(EventType.POST_INSERT).appendListener(createCacheListener());
                registry.getEventListenerGroup(EventType.POST_DELETE).appendListener(createCacheListener());
        }

    }


Entity DAO example:
 

    @Service
    public class SimpleEntityDao {

        private final String queryRegionName = "SimpleEntity_Query";
        private final String query = "SELECT id FROM SimpleEntity WHERE phone = :phone";

        @Autowired
        private QueryCacheEntityListener queryListener;
        @Autowired
        private SessionFactory sessionFactory;

        @PostConstruct
        protected void init() {

                // here is our cache callback
                // invokes on every "insert" or "delete" operation 
                // for SimpleEntity object and keeps query result up-to-date

        	CacheCallback<SimpleEntity> handler = new CacheCallback<SimpleEntity>() {

        		@Override
        		protected void onInsertOrDelete(InsertOrDeleteCommand command,
        				SimpleEntity object) {
        			command.setParameter("phone", object.getPhone());
        			command.setUniqueResult(object.getId());
        		}

        	};
        	queryCacheEntityListener.register(SimpleEntity.class, queryRegionName, handler);
        }
        
        @Transactional
        public SimpleEntity getEntityByPhone(String phone) {
                Session session = sessionFactory.getCurrentSession();
                SQLQuery sqlQuery = session.createSQLQuery(query);
                sqlQuery.addScalar("id", LongType.INSTANCE);
                sqlQuery.setCacheable(true);
                sqlQuery.setCacheRegion(cacheRegionName);
                sqlQuery.setParameter("phone", phone);
                Long idResult = (Long) sqlQuery.uniqueResult();      
                return session.get(SimpleEntity.class, idResult);
        }

        ...
        
        create, delete methods...

    }	




