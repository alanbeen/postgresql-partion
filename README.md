# postgresql-partion
postgresql高效分区

# postgresql-partion

本分区适用于zabbix建立分区表
创建下列函数:
#zabbix_partition_maintenance()

	CREATE OR REPLACE FUNCTION zabbix_partition_maintenance(VARCHAR, INTEGER DEFAULT NULL, INTEGER DEFAULT NULL, INTEGER 		DEFAULT 0, VARCHAR DEFAULT 'zabbix', BOOLEAN DEFAULT FALSE)
	RETURNS void AS
	$BODY$
	DECLARE
	  partition_range        VARCHAR := $1;
	  history_number_to_keep INTEGER := $2;
	  trends_number_to_keep  INTEGER := $3;
	  number_to_drop         INTEGER := $4;
	  object_owner           VARCHAR := $5;
	  dry_run                BOOLEAN := $6;
	BEGIN
	  PERFORM partition_maintenance('history',      partition_range, object_owner, history_number_to_keep, number_to_drop, dry_run);
	  PERFORM partition_maintenance('history_log',  partition_range, object_owner, history_number_to_keep, number_to_drop, dry_run);
	  PERFORM partition_maintenance('history_str',  partition_range, object_owner, history_number_to_keep, number_to_drop, dry_run);
	  PERFORM partition_maintenance('history_text', partition_range, object_owner, history_number_to_keep, number_to_drop, dry_run);
	  PERFORM partition_maintenance('history_uint', partition_range, object_owner, history_number_to_keep, number_to_drop, dry_run);
	  
	  PERFORM partition_maintenance('trends',      partition_range, object_owner, trends_number_to_keep, number_to_drop, dry_run);
	  PERFORM partition_maintenance('trends_uint', partition_range, object_owner, trends_number_to_keep, number_to_drop, dry_run);
	END
	$BODY$
	LANGUAGE plpgsql VOLATILE
	COST 100;

	ALTER FUNCTION zabbix_partition_maintenance(VARCHAR, INTEGER, INTEGER, INTEGER, VARCHAR, BOOLEAN)
	OWNER TO zabbix;

#partition_maintenance()

	CREATE OR REPLACE FUNCTION partition_maintenance(VARCHAR, VARCHAR, VARCHAR, INTEGER DEFAULT NULL, INTEGER DEFAULT 1, BOOLEAN DEFAULT FALSE)
	RETURNS void AS
	$BODY$
	DECLARE
	  master_table    VARCHAR := $1;
	  partition_range VARCHAR := $2;
	  object_owner    VARCHAR := $3;
	  number_to_keep  INTEGER := $4;
	  number_to_drop  INTEGER := $5;
	  dry_run         BOOLEAN := $6;
	  range_interval  INTEGER := EXTRACT(EPOCH FROM partition_range::INTERVAL);
	  now             INTEGER := EXTRACT(EPOCH FROM NOW())::INT;
	 
	  current_range     INTEGER := now / range_interval * range_interval;  
	  current_partition VARCHAR := FORMAT('%s_p%s', master_table, current_range);
	  current_check_min INTEGER := current_range;
	  current_check_max INTEGER := current_check_min + range_interval;
	 
	  previous_range     INTEGER := current_range - range_interval;
	  previous_partition VARCHAR := FORMAT('%s_p%s', master_table, previous_range);
	  previous_check_min INTEGER := previous_range;
	  previous_check_max INTEGER := previous_check_min + range_interval;
	 
	  next_range     INTEGER := current_range + range_interval;
	  next_partition VARCHAR := FORMAT('%s_p%s', master_table, next_range);
	  next_check_min INTEGER := next_range;
	  next_check_max INTEGER := next_check_min + range_interval;
	 
	  function_name VARCHAR := FORMAT('%s_part_trig_func()', master_table);
	 
	  create_function VARCHAR := FORMAT(
		'CREATE OR REPLACE FUNCTION %s
			 RETURNS trigger AS
		   $$ 
		   BEGIN
			 IF TG_OP = ''INSERT'' THEN 
			   IF NEW.clock >= %s AND NEW.clock < %s THEN 
				 INSERT INTO %I VALUES (NEW.*); 
			   ELSIF NEW.clock >= %s AND NEW.clock < %s THEN 
				 INSERT INTO %I VALUES (NEW.*);
			   ELSIF NEW.clock >= %s AND NEW.clock < %s THEN 
				 INSERT INTO %I VALUES (NEW.*);
			   ELSE
				 RETURN NEW;
			   END IF;
			 END IF; 
			 RETURN NULL; 
		   END $$
		   LANGUAGE plpgsql VOLATILE
		   COST 100',
	  function_name,
	  current_check_min,
	  current_check_max,
	  current_partition,
	  next_check_min,
	  next_check_max,
	  next_partition,
	  previous_check_min,
	  previous_check_max,
	  previous_partition);
	 
	  alter_function VARCHAR := FORMAT('ALTER FUNCTION %s OWNER TO %s;', function_name, object_owner);
	BEGIN
	  IF NOT EXISTS (SELECT 1 FROM pg_user WHERE usename = object_owner) THEN
		RAISE EXCEPTION 'User ''%'' does not exist.', object_owner;
	  ELSIF NOT EXISTS (SELECT 1 FROM pg_tables WHERE tablename = master_table) THEN
		RAISE EXCEPTION 'Master table ''%'' does not exist.', master_table;
	  ELSE
		PERFORM add_partition(master_table, previous_partition, previous_check_min, previous_check_max, object_owner, dry_run);
		PERFORM add_partition(master_table, current_partition,  current_check_min,  current_check_max,  object_owner, dry_run);
		PERFORM add_partition(master_table, next_partition,     next_check_min,     next_check_max,     object_owner, dry_run);
	 
		IF dry_run = FALSE THEN
		  EXECUTE create_function;
		  EXECUTE alter_function;
		ELSE
		  RAISE INFO '%', create_function;
		  RAISE INFO '%', alter_function;
		END IF;
	 
		IF number_to_keep IS NOT NULL THEN
		  PERFORM remove_partitions(master_table, current_range, range_interval, number_to_keep, number_to_drop, dry_run);
		END IF;
	  END IF;
	END
	$BODY$
	LANGUAGE plpgsql VOLATILE
	COST 100;

	ALTER FUNCTION partition_maintenance(VARCHAR, VARCHAR, VARCHAR, INTEGER, INTEGER, BOOLEAN)
	OWNER TO zabbix;
#add_partition()

	CREATE OR REPLACE FUNCTION add_partition(VARCHAR, VARCHAR, INTEGER, INTEGER, VARCHAR, BOOLEAN DEFAULT FALSE)
	RETURNS void AS
	$BODY$
	DECLARE
	  master_table VARCHAR := $1;
	  child_table  VARCHAR := $2;
	  check_min    INTEGER := $3;
	  check_max    INTEGER := $4;
	  table_owner  VARCHAR := $5;
	  dry_run      BOOLEAN := $6;
	  
	  create_table VARCHAR := FORMAT(
		'CREATE TABLE %I (
			 CHECK (clock >= %s AND clock < %s),
			 LIKE %I
			 INCLUDING DEFAULTS INCLUDING CONSTRAINTS INCLUDING INDEXES INCLUDING STORAGE INCLUDING COMMENTS)',
		child_table,
		check_min,
		check_max,
		master_table);
		
	  alter_table_owner   VARCHAR := FORMAT('ALTER TABLE %I OWNER TO %I', child_table, table_owner);
	  alter_table_inherit VARCHAR := FORMAT('ALTER TABLE %I INHERIT %I', child_table, master_table);
	BEGIN
	  IF EXISTS (SELECT 1 FROM pg_tables WHERE tablename = child_table ) THEN
		RAISE NOTICE 'Child table ''%'' already exists. Skipping.', child_table;
	  ELSE
		IF dry_run = false THEN
		  EXECUTE create_table;
		  EXECUTE alter_table_owner;
		  EXECUTE alter_table_inherit;
		ELSE
		  RAISE INFO '%', create_table;
		  RAISE INFO '%', alter_table_owner;
		  RAISE INFO '%', alter_table_inherit;
		END IF;
	  END IF;
	END
	$BODY$
	LANGUAGE plpgsql VOLATILE
	COST 100;

	ALTER FUNCTION add_partition(VARCHAR, VARCHAR, INTEGER, INTEGER, VARCHAR, BOOLEAN)
	OWNER TO zabbix;
#remove_partitions()

	CREATE OR REPLACE FUNCTION remove_partitions(VARCHAR, INTEGER, INTEGER, INTEGER, INTEGER, BOOLEAN DEFAULT FALSE)
	RETURNS void AS
	$BODY$
	DECLARE
	  master_table   VARCHAR := $1;
	  current_range  INTEGER := $2;
	  range_interval INTEGER := $3;
	  number_to_keep INTEGER := $4;
	  number_to_drop INTEGER := $5;
	  dry_run        BOOLEAN := $6;
	  max_range      INTEGER := current_range - range_interval * number_to_keep;
	  min_range      INTEGER := max_range - range_interval * (number_to_drop - 1);
	  child_table    VARCHAR;
	  drop_table     VARCHAR;
	BEGIN
	  FOR range_to_drop IN REVERSE max_range .. min_range BY range_interval LOOP
		child_table := FORMAT('%s_p%s', master_table, range_to_drop);

		IF EXISTS (SELECT 1 FROM pg_tables WHERE tablename = child_table) THEN
		  drop_table := FORMAT('DROP TABLE %I', child_table);
		  
		  IF dry_run = false THEN
			EXECUTE drop_table;
		  ELSE
			RAISE INFO '%', drop_table;
		  END IF;
		ELSE
		  RAISE INFO 'Child table ''%'' does not exist. Stopping.', child_table;
		  EXIT;
		END IF;
	  END LOOP;
	END
	$BODY$
	LANGUAGE plpgsql VOLATILE
	COST 100;

	ALTER FUNCTION remove_partitions(VARCHAR, INTEGER, INTEGER, INTEGER, INTEGER, BOOLEAN)
	OWNER TO zabbix;
	
创建触发器

CREATE TRIGGER history_part_trig      BEFORE INSERT ON history      FOR EACH ROW EXECUTE PROCEDURE history_part_trig_func();
CREATE TRIGGER history_log_part_trig  BEFORE INSERT ON history_log  FOR EACH ROW EXECUTE PROCEDURE history_log_part_trig_func();
CREATE TRIGGER history_str_part_trig  BEFORE INSERT ON history_str  FOR EACH ROW EXECUTE PROCEDURE history_str_part_trig_func();
CREATE TRIGGER history_text_part_trig BEFORE INSERT ON history_text FOR EACH ROW EXECUTE PROCEDURE history_text_part_trig_func();
CREATE TRIGGER history_uint_part_trig BEFORE INSERT ON history_uint FOR EACH ROW EXECUTE PROCEDURE history_uint_part_trig_func();
CREATE TRIGGER trends_part_trig       BEFORE INSERT ON trends       FOR EACH ROW EXECUTE PROCEDURE trends_part_trig_func();
CREATE TRIGGER trends_uint_part_trig  BEFORE INSERT ON trends_uint  FOR EACH ROW EXECUTE PROCEDURE trends_uint_part_trig_func();

开始分区

SELECT zabbix_partition_maintenance('1 week', 10, NULL, 1, 'zabbix', TRUE);
参数一:可以是month week day 等
参数二：history表保存的天数 默认为NULL
参数三：trends 表保存的天数 默认为NULL
参数四：删除表个数（需要在保存天数之外的才会删除）默认为1
参数五：所要操作的数据库
参数六：最后什么也不做，但显示DDL语句会执行


自动分区添加（linux crontab）
cd ~
vi .pgpass 
#地址:端口:数据库:用户名:密码
127.0.0.1:5432:zabbix:zabbix:zabbix

cat maintainZbxDbPartitions.sh

#!/bin/bash
echo `psql -h 127.0.0.1 -p 5432 -d zabbix -U zabbix << EOF
 \set VERBOSITY 'terse'
 SELECT zabbix_partition_maintenance('1 day', 10, 30, 1, 'zabbix', TRUE);
 \quit
EOF`

添加crontab
crontab -l
59  23  *  *  * /usr/etc/scripts/maintainZbxDbPartitions.sh runme &> /home/logs/pgsql/maintainZbxDbPartitions.log
