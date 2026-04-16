# 你现在该做什么

先把两件事补上：

## 1. 安装 Git

```
sudo apt update
sudo apt install -y git
```

确认一下：

```
git --version
```

------

## 2. 给公司这台机器生成自己的 SSH key

你家里那台配过，不代表公司这台也有。两台机器要分别配。

执行：

```
ssh-keygen -t ed25519 -C "你的GitHub邮箱"
```

它问你保存位置时，直接回车。
 问 passphrase 时，你现在可以直接回车留空。

然后看公钥：

```
cat ~/.ssh/id_ed25519.pub
```

复制整行输出。

------

# 3. 把这把公钥加到 GitHub

去 GitHub：

- 右上角头像
- Settings
- SSH and GPG keys
- New SSH key

然后：

- Title 写 `company-vm` 或类似名字
- Key type 选 Authentication Key
- 把刚才那整行公钥粘进去
- 保存

------

# 4. 再测试 SSH

```
ssh -T git@github.com
```

如果成功，你会看到类似：

```
Hi Yunbaby8! You've successfully authenticated, but GitHub does not provide shell access.
```