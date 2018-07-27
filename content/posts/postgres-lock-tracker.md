---
title: "[EN] Lock inspections"
date: 2018-07-27T22:26:12+02:00
draft: true
---

coucou  les loulous !
<!--more-->
# more

SELECT l.mode, l.pid, a.datname, c.relname, a.usename, a.application_name,  a.query_start, age(clock_timestamp(), a.query_start), query
			FROM pg_catalog.pg_locks l JOIN pg_class c ON l.relation = c.oid
			JOIN pg_catalog.pg_stat_activity a ON l.pid = a.pid
			WHERE granted = true AND mode = 'AccessExclusiveLock';


SELECT l.mode, l.pid, a.datname, c.relname, a.usename, a.application_name,  a.query_start, age(clock_timestamp(), a.query_start), query
			FROM pg_catalog.pg_locks l JOIN pg_class c ON l.relation = c.oid
			JOIN pg_catalog.pg_stat_activity a ON l.pid = a.pid
			WHERE granted = true AND mode = 'AccessExclusiveLock';

select * from pg_catalog.pg_stat_activity where pid = 10649;

DROP TABLE IF EXISTS lockTracking;
CREATE TABLE IF NOT EXISTS lockTracking (
	mode TEXT,
	pid INTEGER, 
	db TEXT,
	relation TEXT, 
	username TEXT,
	application TEXT, 
	startedAt TIMESTAMP WITH TIME ZONE,
	age INTERVAL, 
	query TEXT
);

CREATE OR REPLACE FUNCTION  trackLocks(duration INTERVAL, loopInterval INTERVAL) RETURNS void AS $$
	DECLARE i INTEGER;
	DECLARE stopAt TIMESTAMP;
BEGIN
	DELETE FROM lockTracking;
	stopAt = clock_timestamp() + duration;
	raise notice 'will stop at %', stopAt;
	WHILE clock_timestamp() < stopAt LOOP

	    INSERT INTO locktracking(mode, pid, db, relation, username, application, startedAt, age, query) 
			SELECT l.mode, l.pid, a.datname, c.relname, a.usename, a.application_name,  a.query_start, age(clock_timestamp(), a.query_start), query
			FROM pg_catalog.pg_locks l JOIN pg_class c ON l.relation = c.oid
			JOIN pg_catalog.pg_stat_activity a ON l.pid = a.pid
			WHERE granted = true AND mode = 'AccessExclusiveLock' ;
		
		PERFORM pg_sleep(EXTRACT('seconds' FROM loopInterval));
		
	END LOOP;
END;
$$ LANGUAGE plpgsql;

SELECT trackLocks('20 seconds'::interval, '0.5 second');
SELECT pid, db, relation, startedAt, query, MAX(age) as duration FROM locktracking
GROUP by pid, db, relation, startedAt, query
ORDER BY startedAt;

select * from locktracking
select  clock_timestamp()
