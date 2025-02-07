# Encryption & Decryption

DBPack supports automatic encryption and decryption for sensitive data. DBPack uses AES algorithm to encrypt data, for example, when inserting and updating data, DBPack will automatically encrypt certain fields. If there are certain encrypted fields when querying from DB, DBPack will automatically decrypt them. Currently, encrypted fields are not allowed to be `WHERE` conditions.

You can enable encryption and decryption by adding `CryptoFilter` configuration. For example: 

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
```

Encrypt the specified table and column as needed. `aeskey` is the encryption key, and `cryptoType` is the supported encryption and decryption type. Currently, AES and SM4 algorithms are supported, and the following encryption and decryption types are formed by combining different encryption modes: `aesgcm`, `aescbc`, `aesecb`, `aescfb`, `sm4gcm`, `sm4ecb`, `sm4cbc`, `sm4cfb`, `sm4ofb`.

Above configuration aims to enable automatic encryption and decryption for `dept_name` column of `departments` table.

+ Insert operation:

```
INSERT INTO departments (id,dept_no,dept_name) VALUES (1,'1001','sunset')
```

will be overwritten to: 

```
INSERT INTO departments (id,dept_no,dept_name) VALUES (1,'1001','3d244141cb5b6f921923f7f88f073941')
```

+ Update operation:

```
UPDATE departments SET dept_name='moonlight' WHERE id=1
```

will be overwritten to:

```
UPDATE departments SET dept_name='5cdeb84b8fc3c22fd6c3e37ca6d837da' WHERE id=1
```

+ When selecting from DB, these encrypted data will be decrypted and return to user. 

Notice: After added encryption configuration to specified fields, in the physical DB, it stores the encrypted data instead of original data, to prevent data leakage when the database has been hacked. The columns' length to be encrypted should be set longer accordingly, since the encrypted column data length is longer than the original data.
