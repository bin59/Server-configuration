起因：

ALTER USER 'root'@'219.136.189.121' IDENTIFIED WITH mysql_native_password BY 'zbmlp01.';

报错

RROR 1396 (HY000>: Operation ALTER USER failedfor 'root'0'219.136.189.121’

你遇到的 ERROR 1396 错误，核心问题通常不在于命令语法本身，而是 MySQL 在系统中找不到 `'root'@'219.136.189.121'` 这个精确的用户组合。更具体地说，是**用户账户的主机名（host）不匹配**。

在 MySQL 5.7.6 及更高版本中，一个用户账户由 `'用户名'@'主机名'` 共同唯一确定[reference:0][reference:1]。这意味着 `'root'@'219.136.189.121'` 和 `'root'@'localhost'` 是两个不同的账户。

### 📋 诊断：查找准确的 root 账户信息

解决问题的第一步，是查明你的数据库里到底存在哪些 `root` 账户。请按以下步骤操作：

1.  使用一个有权限的账户（如 `root` 本身）连接到 MySQL。
2.  切换并查询 `mysql.user` 表，这个表里存储着所有用户的信息。
    ```sql
    USE mysql;
    SELECT User, Host, plugin FROM user WHERE User='root';
    ```

这个查询会列出所有 `root` 账户及其允许连接的主机。最可能的输出是 `'root'@'localhost'` 或 `'root'@'%'`，但不会是 `'root'@'219.136.189.121'` 这种具体的 IP 地址。

### 💡 解决方案：根据你的情况选择

根据查询结果，你可以选择最适合的一种方案来解决：

#### 方案一：修改现有 root 账户（推荐，且最安全）

这是最简单直接的方法，就是修改你查询到的那个**真实存在的** root 账户的密码。

- **如果查询结果是 `'root'@'%'`**：这意味着 root 可以从任何主机连接，密码修改命令应改为：
  ```sql
  ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'zbmlp01.';
  FLUSH PRIVILEGES;
  ```
- **如果查询结果是 `'root'@'localhost'`**：这意味着 root 仅限从数据库服务器本机连接，密码修改命令应改为：
  ```sql
  ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'zbmlp01.';
  FLUSH PRIVILEGES;
  ```

#### 方案二：为特定 IP 创建专用账户

如果你的业务确实需要 `root` 用户从一个固定的 IP 地址（`219.136.189.121`）进行远程连接，最佳实践是为它创建一个独立的专用账户，而不是强制修改默认的 `root`。

```sql
CREATE USER 'myuser'@'219.136.189.121' IDENTIFIED WITH mysql_native_password BY 'zbmlp01.';
GRANT ALL PRIVILEGES ON *.* TO 'myuser'@'219.136.189.121' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

创建成功后，就可以使用 `myuser` 这个账户和密码，从指定的 IP 地址连接到数据库了。

#### 方案三：强制修改现有 root 账户的 Host（不推荐，有安全风险）

此方法是将现有 `root` 账户允许连接的主机范围改为 `'%'`（所有主机）。这会严重降低安全性，请谨慎评估风险。

1.  更新 `root` 用户的 `host` 字段为 `'%'`。
    ```sql
    UPDATE mysql.user SET host = '%' WHERE user = 'root';
    FLUSH PRIVILEGES;
    ```
2.  然后再执行你最初想要修改密码的命令：
    ```sql
    ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'zbmlp01.';
    FLUSH PRIVILEGES;
    ```

#### 🛠️ 最后的手段：故障排查

如果以上方法都无效，可能还有其他潜在问题。

- **尝试刷新权限**：有时系统权限缓存没有更新，执行 `FLUSH PRIVILEGES;` 可以强制刷新[reference:2][reference:3]。
- **检查元数据一致性**：如果之前使用 `DELETE FROM mysql.user` 等语句手动修改过系统表，可能导致内存和磁盘数据不一致。此时，**再次执行 `FLUSH PRIVILEGES;`** 也可能解决问题[reference:4][reference:5]。

在成功修改账户信息后，**强烈建议执行 `FLUSH PRIVILEGES;`** 命令来刷新权限，确保更改立即生效[reference:6][reference:7]。

### 💎 总结

你遇到的 `ERROR 1396` 错误，最可能的原因就是**指定的用户和主机组合在数据库中并不存在**。建议采用**方案一**，将命令中的主机名改为 `'%'` 或 `'localhost'`，这通常能最快解决问题。
