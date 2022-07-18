# Audit Log

DBPack supports audit logs. Please follow below configuration example to enable it.

```yaml
data_source_cluster:
  - name: employees
    capacity: 10
    max_capacity: 20
    idle_timeout: 60s
    dsn: root:123456@tcp(dbpack-mysql:3306)/employees?timeout=60s&readTimeout=60s&writeTimeout=60s&parseTime=true&loc=Local&charset=utf8mb4,utf8
    ping_interval: 20s
    ping_times_for_change_status: 3
    filters:
      - auditLogFilter

filters:
  - name: auditLogFilter
    kind: AuditLogFilter
    conf:
      audit_log_dir: /var/log/dbpack/
      # unit MB
      max_size: 300
      # unit Day
      max_age: 28
      # maximum number of old log files to retain
      max_backups: 1
      # determines if the rotated log files should be compressed using gzip
      compress: true
      # define whether to log before or after sql execution
      record_before: true
```

To enable audit log, please add `audit_log_dir` to filters to configure log directory, as well as configuring audit log filter for datasource, so that DBPack will output audit logs to specified directories. You can set `max_size` to configure log rotate. In upper configuration example, the DBPack will create new audit log file after the first log file reached 300MB. You can also configure log retain time and whether to compress the rotated logs.

The audit log format will be:

```
[timestamp],[username],[ip address],[connection id],[command type],[command],[sql text],[args],[affected row]
```

example:

```
2022-06-14 07:15:44,dksl,172.18.0.1:60372,1,COM_QUERY,,SET NAMES utf8mb4,[],0
2022-06-14 07:15:45,dksl,172.18.0.1:60372,1,COM_STMT_EXECUTE,INSERT,INSERT INTO employees ( emp_no, birth_date, first_name, last_name, gender, hire_date ) VALUES (?, ?, ?, ?, ?, ?),['100000' '1992-01-07' 'scott' 'lewis' 'M' '2014-09-01'],1
2022-06-14 07:15:45,dksl,172.18.0.1:60372,1,COM_STMT_EXECUTE,DELETE,DELETE FROM employees WHERE emp_no = ?,['100000'],1
2022-06-14 07:15:45,dksl,172.18.0.1:60372,1,COM_STMT_EXECUTE,INSERT,INSERT INTO employees ( emp_no, birth_date, first_name, last_name, gender, hire_date ) VALUES (?, ?, ?, ?, ?, ?),['100001' '1992-01-07' 'scott' 'lewis' 'M' '2014-09-01'],1
2022-06-14 07:15:45,dksl,172.18.0.1:60372,1,COM_STMT_EXECUTE,SELECT,SELECT emp_no, birth_date, first_name, last_name, gender, hire_date FROM employees WHERE emp_no = ?,['100001'],0
2022-06-14 07:15:45,dksl,172.18.0.1:60384,2,COM_QUERY,,SET NAMES utf8mb4,[],0
2022-06-14 07:15:45,dksl,172.18.0.1:60384,2,COM_STMT_EXECUTE,UPDATE,UPDATE employees set last_name = ? where emp_no = ?,['louis' '100001'],1
2022-06-14 07:15:45,dksl,172.18.0.1:60384,2,COM_STMT_EXECUTE,DELETE,DELETE FROM employees WHERE emp_no = ?,['100001'],1
2022-06-14 07:15:45,dksl,172.18.0.1:60392,3,COM_QUERY,,SET NAMES utf8mb4,[],0
2022-06-14 07:15:45,dksl,172.18.0.1:60392,3,COM_STMT_EXECUTE,INSERT,INSERT INTO employees ( id, emp_no, birth_date, first_name, last_name, gender, hire_date ) VALUES (?, ?, ?, ?, ?, ?, ?),['1' '100001' '1992-06-04' 'jane' 'watson' 'F' '2013-06-01'],1
2022-06-14 07:15:45,dksl,172.18.0.1:60392,3,COM_STMT_EXECUTE,INSERT,INSERT INTO departments ( id, dept_no, dept_name ) VALUES (?, ?, ?),['1' '1001' 'sales'],1
2022-06-14 07:15:45,dksl,172.18.0.1:60392,3,COM_STMT_EXECUTE,INSERT,INSERT INTO dept_emp ( id, emp_no, dept_no, from_date, to_date ) VALUES (?, ?, ?, ?, ?),['1' '100001' '1001' '2020-01-01' '2022-01-01'],1
2022-06-14 07:15:45,dksl,172.18.0.1:60392,3,COM_STMT_EXECUTE,INSERT,INSERT INTO salaries ( id, emp_no, salary, from_date, to_date ) VALUES (?, ?, ?, ?, ?),['1' '100001' '8000' '2020-01-01' '2025-01-01'],1
2022-06-14 07:15:45,dksl,172.18.0.1:60394,4,COM_QUERY,,SET NAMES utf8mb4,[],0
2022-06-14 07:15:45,dksl,172.18.0.1:60394,4,COM_QUERY,INSERT,INSERT INTO dept_emp ( id, emp_no, dept_no, from_date, to_date ) VALUES (2, 100002, 1002, '2020-01-01', '2022-01-01'),[],1
2022-06-14 07:15:45,dksl,172.18.0.1:60394,4,COM_QUERY,INSERT,INSERT INTO salaries ( id, emp_no, salary, from_date, to_date ) VALUES (2, 100002, 8000, '2020-01-01', '2025-01-01'),[],1
```

