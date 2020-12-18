# Mac折腾

### 安装低版本的 node

```bash
brew install node@10
brew link node@10
brew link --overwrite --force node@10
```

### 切换Python到3.x版本

安装：

```bash
brew install python@3.7
```

在 .bash_profile 设置 python 别名

```bash
sudo unlink /usr/bin/python
sudo ln -s /usr/local/Cellar/python@3.7/3.7.9_2/bin/python3.7 /usr/bin/python
python -V
```

### 清除Mac DNS 缓存

```bash
sudo killall -HUP mDNSResponder
sudo killall mDNSResponderHelper
sudo dscacheutil -flushcache
```

