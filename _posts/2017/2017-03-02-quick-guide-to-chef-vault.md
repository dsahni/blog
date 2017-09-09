---
title: Quick Guide to Chef-Vault
description: A quick and dirty guide to using chef-vault
header: Quick Guide to Chef-Vault
tags: [chef, dev]
---
**1) Create a cleartext JSON file to be encrypted**

`vi secrets.json`

**2) Create a vault from the file**

`knife vault create myvault mysecrets -J /path/to/secrets.json`

**Note**: This only creates the vault _locally_. You'll still need to upload the databag items (the encrypted content and the corresponding keys) to the Chef Server - outlined in the the last step.

**3) Specify who gets to access the vault (can be users _or_ nodes)**

`knife vault update myvault mysecrets -A "mynode.fqdn.com,mynode2.fqdn.com,user2,user3"`

**4) Upload the newly created vault**
```
cd /to/chef-repo/dir
knife upload data_bags/myvault
```
