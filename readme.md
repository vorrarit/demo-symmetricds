# Setup a SymmetricDS Demo

## Create Environment

```
docker network create symmetricds-network
```

create db1 as a source database
```
docker run -d --name db1 \
    -e POSTGRES_PASSWORD=password \
    -e POSTGRES_DB=db1 \
    --net symmetricds-network \
    -p5433:5432 \
    postgres:14.1-alpine
```

create db2 as a destination database
```
docker run -d --name db2 \
    -e POSTGRES_PASSWORD=password \
    -e POSTGRES_DB=db2 \
    --net symmetricds-network \
    -p5434:5432 \
    postgres:14.1-alpine
```

create symmetric ds container and mount engines folder
```
docker run -it --rm --name symmetricds \
    --net symmetricds-network \
    -p 31415:31415 \
    -v "$(pwd)/symmetricds/engines:/opt/symmetric-ds/engines" \
    jumpmind/symmetricds
```

## Configurations
### db1-001.properties
specify the following configurations
- engine.name
- db.*
- sync.url
- group.id
- external.id
- registration.url **must be blank**


### db2-002.properties
specify the following configurations
- engine.name
- db.*
- sync.url
- group.id
- external.id
- registration.url

## Database Configuration

### Enabling Registration at DB1
** restart symmetric ds may required
~~~sql
insert into SYM_NODE_GROUP
        (node_group_id, description)
        values ('db2', 'DB2 node');
        
insert into SYM_NODE_GROUP_LINK
(source_node_group_id, target_node_group_id, data_event_action)
      values ('db1', 'db2', 'P');
~~~

### Create customers table for demonstration
~~~sql
create table customers (
	id bigserial primary key, // auto increment
	name character varying(20)
);
~~~

### Create Channel
~~~sql
insert into sym_channel (
    channel_id, processing_order, max_batch_size, enabled, description
) values (
    'channel1', 1, 100000, 1, 'channel1'
);
~~~

### Create Trigger (Source Table)
~~~sql
insert into sym_trigger (
    trigger_id, source_catalog_name, source_schema_name, source_table_name, 
    channel_id,last_update_time,create_time
) values(
    'customers_tg', null, 'app1', 'test_sym',
    'channel1', current_timestamp, current_timestamp
);
~~~

### Create Router (Destination Table)
~~~sql
insert into sym_router (
    router_id, target_catalog_name, target_schema_name, 
    source_node_group_id, target_node_group_id, 
    router_type, create_time, last_update_time
) values(
    'db1_to_db2', null, 'app2', 
    'db1', 'db2', 
    'default', current_timestamp, current_timestamp
);
~~~

### Create Trigger Router (Source / Destination Mapping)
~~~sql
insert into sym_trigger_router (
    trigger_id, router_id, initial_load_order, 
    last_update_time, create_time
) values (
    'customers_tg','db1_to_db2', 100, 
    current_timestamp, current_timestamp
);
~~~


### Create customers table at DB2
~~~sql
create table customers (
	id bigint primary key, //bigint
	name character varying(20)
);
~~~


## Testing
### Insert data at DB1 and check sync result at DB2
~~~sql
insert into customers (name) values ('name001');
insert into customers (name) values ('name002');
update customers set name='edit-name001' where id=1;
~~~
