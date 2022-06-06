# Oracle 设置用户密码永不过期

[TOC]

## Oracle 默认过期时间

Oracle 账号密码默认过期日期为 *180* 天，过期后用户登录会提示错误码 *ORA-28002*，出现此类错误需要使用DBA账号重置该账号密码。具体操作步骤如下：

## 1. 查看口令失效用户的`PROFILE`文件

```sql
SQL> select username,profile from dba_users;
```

这里以`DEFAULT` 概要文件为例。

## 2. 查看`PROFILE`文件口令有效期设置

```sql
SQL>SELECT * FROM dba_profiles s WHERE s.profile='DEFAULT' AND resource_name='PASSWORD_LIFE_TIME';
```

## 3. 将口令有效期默认值180天，设置为无限制

```sql
SQL>ALTER PROFILE DEFAULT LIMIT PASSWORD_LIFE_TIME UNLIMITED;
```

该参数修改后实时生效。

## 4.修改已失效用户密码

在修改PASSWORD_LIFE_TIME值之前已经失效的用户，还是需要重新修改一次密码才能使用。

```sql
SQL>ALTER USER <USER> INDENTIFIED BY <PASSWORD>
```









