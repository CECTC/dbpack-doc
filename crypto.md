# 数据加解密

DBPack 支持对敏感数据自动加密解密。DBPack 采用 AES 算法来加密数据。在插入和更新相应的数据时，DBPack 会对相关的列进行自动加密。当查询结果中存在相关的列时，DBPack 会自动解密。DBPack 不支持使用加密列作为 `WHERE` 条件。

开启加解密功能需通过在配置文件中增加 `CryptoFilter` 配置。例如：

```yaml
executors:
  - name: redirect
    mode: sdb
    config:
      data_source_ref: employees
    filters:
      - cryptoFilter
filters:
  - name: cryptoFilter
    kind: CryptoFilter
    conf:
      column_crypto_list:
      - table: departments
        columns: ["dept_name"]
        aeskey: 123456789abcdefg
        cryptoType: aesgcm
```

根据需要对指定表，指定列进行加密。`aeskey` 为加密密钥，`cryptoType` 为支持的加解密类型。目前支持AES和国密SM4算法，结合不同的加密模式组成以下加解密类型：`aesgcm`, `aescbc`, `aesecb`, `aescfb`, `sm4gcm`, `sm4ecb`, `sm4cbc`, `sm4cfb`, `sm4ofb`。

上面的配置表示需要对 `departments` 表的 `dept_name` 列进行自动加解密。

+ 插入操作：

```
INSERT INTO departments (id,dept_no,dept_name) VALUES (1,'1001','sunset')
```

会被重写为：

```
INSERT INTO departments (id,dept_no,dept_name) VALUES (1,'1001','3d244141cb5b6f921923f7f88f073941')
```

+ 更新操作：

```
UPDATE departments SET dept_name='moonlight' WHERE id=1
```

会被重写为：

```
UPDATE departments SET dept_name='5cdeb84b8fc3c22fd6c3e37ca6d837da' WHERE id=1
```

+ 查询时会自动解密返回给用户

注意：设置对指定的列加密后，数据库里只保存密文，不会保存明文，防止被拖库后数据泄漏。加密后的列数据比原本的数据要长，需设置好对应列的长度。