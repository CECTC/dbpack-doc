# 审计日志

DBPack 支持记录审计日志。可通过配置开启，如下：

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

在 filter 中通过 `audit_log_dir` 指定审计日志目录，并在数据源上配置审计日志 filter，DBPack 会将审计日志输出到指定的目录。还可以通过 `max_size` 配置审计日志轮转，上面的例子中，当审计日志文件大小达到 300 兆时会开启新的文件记录审计日志。还可配置日志保留的时间以及对轮转的日志压缩。

审计日志的格式为：

```
[timestamp],[username],[ip address],[connection id],[command type],[command],[sql text],[args],[affected row]
```

例如：

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

