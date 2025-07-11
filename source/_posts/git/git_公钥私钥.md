---
title: git_公钥私钥
categories:
  - - Linux
  - - git
tags: 
date: 2022/10/24
update: 
comments: 
published:
---



```
usage: ssh-keygen [-q] [-a rounds] [-b bits] [-C comment] [-f output_keyfile]
                  [-m format] [-N new_passphrase] [-O option]
                  [-t dsa | ecdsa | ecdsa-sk | ed25519 | ed25519-sk | rsa]
                  [-w provider] [-Z cipher]
       ssh-keygen -p [-a rounds] [-f keyfile] [-m format] [-N new_passphrase]
                   [-P old_passphrase] [-Z cipher]
       ssh-keygen -i [-f input_keyfile] [-m key_format]
       ssh-keygen -e [-f input_keyfile] [-m key_format]
       ssh-keygen -y [-f input_keyfile]
       ssh-keygen -c [-a rounds] [-C comment] [-f keyfile] [-P passphrase]
       ssh-keygen -l [-v] [-E fingerprint_hash] [-f input_keyfile]
       ssh-keygen -B [-f input_keyfile]
       ssh-keygen -D pkcs11
       ssh-keygen -F hostname [-lv] [-f known_hosts_file]
       ssh-keygen -H [-f known_hosts_file]
       ssh-keygen -K [-a rounds] [-w provider]
       ssh-keygen -R hostname [-f known_hosts_file]
       ssh-keygen -r hostname [-g] [-f input_keyfile]
       ssh-keygen -M generate [-O option] output_file
       ssh-keygen -M screen [-f input_file] [-O option] output_file
       ssh-keygen -I certificate_identity -s ca_key [-hU] [-D pkcs11_provider]
                  [-n principals] [-O option] [-V validity_interval]
                  [-z serial_number] file ...
       ssh-keygen -L [-f input_keyfile]
       ssh-keygen -A [-a rounds] [-f prefix_path]
       ssh-keygen -k -f krl_file [-u] [-s ca_public] [-z version_number]
                  file ...
       ssh-keygen -Q [-l] -f krl_file [file ...]
       ssh-keygen -Y find-principals -s signature_file -f allowed_signers_file
       ssh-keygen -Y match-principals -I signer_identity -f allowed_signers_file
       ssh-keygen -Y check-novalidate -n namespace -s signature_file
       ssh-keygen -Y sign -f key_file -n namespace file [-O option] ...
       ssh-keygen -Y verify -f allowed_signers_file -I signer_identity
                  -n namespace -s signature_file [-r krl_file] [-O option]
```

# 生成SSH Keys

如果已经生存了ssh key，那就可以跳过这一步了。可以用以下命令查看：

```bash
ls -l ~/.ssh
```

如果出现`id_rsa`和`id_rsa_pub`那就说明已经生成。

如果没有，按以下步骤生成：

```
ssh-keygen -t rsa -C “your_email@example.com”
```

## 示例

`ssh-keygen -t rsa -C "xing-cg@qq.com"`

其中，`-t`选项代表的是生成哪种密钥，`[-t dsa | ecdsa | ecdsa-sk | ed25519 | ed25519-sk | rsa]`，选择rsa。

`-C`选项代表注释，一般填写邮箱地址。

# 拷贝现有的到新机器上

clone github的项目时出问题，报以下错误。

```
Permission denied (publickey).
  fatal: Could not read from remote repository.
  Please make sure you have the correct access rights
```

## 解决方法

1. 查看ssh-agent是否启用

   ```
   ssh-agent -s
   ```

   如果看到Agent pid xxxx说明已经启用。接下来把私钥添加到ssh-agent就可以了。

2. 将`.Key`添加到`ssh-agent`

   ```
   ssh-add ~/.ssh/id_rsa
   ```

   可能会提示，`It is required that your private key files are NOT accessible by others`。

   ```
   @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
   @         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
   @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
   Permissions 0664 for '/home/xcg/.ssh/id_rsa' are too open.
   It is required that your private key files are NOT accessible by others.
   This private key will be ignored.
   ```

   解决方法：`chmod 600 ~/.ssh/id_rsa ~/.ssh/id_rsa.pub`

3. 将ssh的公钥添加到git上

   1. 查找公钥，一般在`~/.ssh`中的`id_rsa.pub`中
   2. 打开github或gitlab，再依次进入settings、ssh keys、add ssh keys进行添加。
