# Linux Notes

## 应用软件

### git

发布 release 版本

```shell
git tag -a v0.1 -m "First release"
git push origin v0.1
```

分支

```shell
git branch develop # 建立分支
git checkout develop # 转到分支
git push origin develop:remote_develop # 推送分支
git checkout master
git cherry-pick 62ecb3 # 合并指定 commit
```



## 基本操作

#### 配置算法

##### GDB

[GDB实用插件peda, gef, pwndbg](https://www.jianshu.com/p/94a71af2022a)

[.gdbinit](https://www.cse.unsw.edu.au/~learn/debugging/modules/gdb_init_file/)

```
printf "\n"
printf "User commands:\n" 
printf "mode <num>: 1.peda 2.gef\n"
printf "\n"

define mode
    if $arg0 == 1
        source /home/ubuntu/Codes/peda/peda.py
    else
        if $arg0 == 2
            source /home/ubuntu/Codes/gef/gef.py
        else
            printf "Error! Please input the right number.\n"
        end
    end
end

document mode
    mode <num>
    <num>: 1.peda 2.gef
end
```



##### 递归下载 deb 包供离线使用

```shell
# 外网电脑准备：
sudo apt-get install dpkg-dev gnupg rng-tools
## 下载离线包
sudo rm -rf /var/cache/apt/archives/*  # 清空缓存目录
sudo apt-get -d install pkgs           # 仅下载 
mkdir /var/debs
cp -r /var/cache/apt/archives/* /var/debs/

## 对包签名
sudo rngd -r /dev/urandom              # 避免生成随机数时间过长
gpg --gen-key                          # 生成公私钥对
gpg --list-key                         # 查看密钥对
gpg -a --export-secret-keys username > username_sk.sec # 导出私钥
gpg -a --export username > username_pk.pub             # 导出公钥
apt-ftparchive packages debs > debs/Packages
#apt-ftparchive sources debs > debs/Sources
cd debs; gzip -c Packages > Packages.gz
        #gzip -c Sources > Sources.gz
apt-ftparchive release ./ > Release
gpg --clearsign -o InRelease Release
gpg -abs -o Release.gpg Release

# 内网电脑操作：
## 同样拷贝到 /var/debs
## 修改更新本地源：
sudo apt-key add username_pk.pub       # 导入公钥
sudo gedit /etc/apt/sources.list
deb file:/var debs/
#deb-src file:/var debs/
sudo apt-get update
sudo apt-get install pkgs
```

另一种递归下载依赖包的脚本：

```shell
#! /bin/bash
basepath=$(cd `dirname $0`; pwd)
pkg="$*"

function getDepends()
{
  ret=`apt-cache depends $1 | grep -i Depends | cut -d: -f2 | tr -d "<>"`
  if [[ -z $ret ]]; then
#    echo "$1 No depends"
     echo -n
  else
    for i in $ret
    do
       if [[ `echo $pkg | grep -e "$i "` ]]; then
#         echo "$i already in set"
         echo -n
       else
         pkg="$pkg $i"
         echo "Downloading $i"
         getDepends $i
       fi
    done
  fi
}

for j in $@
do
  getDepends $j
done

echo $pkg
#apt install $pkg --reinstall -d -y
for p in $pkg
{
  apt-get download $p -d -y
}
```

#### 美观和快捷配置

##### 修改更新源

```shell
cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo vim /etc/apt/sources.list
:%s/archive.ubuntu/mirrors.aliyun/g
sudo apt-get update
```

##### 安装 oh-my-zsh

```shell
sudo apt-get install zsh
chsh -s $(which zsh)
echo $SHELL # /usr/bin/zsh
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
# 设置 ys 主题
sudo vim ~/.zshrc
ZSH_THEME="ys"
source ~/.zshrc
# 安装语法高亮
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git
echo "source ${(q-)PWD}/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh" >> ${ZDOTDIR:-$HOME}/.zshrc
```

##### 添加桌面快捷方式

wine 好绿色版程序后，在 /usr/bin/ 下建一个新的 xxx 程序

``` shell
#!/bin/bash  
FILENAME="z:${1//\//\\}"
wine "/home/username/software/path/xxx.exe" $FILENAME
```

在 ~/.local/share/applications/ 目录下建一个新的 [Entry](https://developer.gnome.org/desktop-entry-spec/)

```
[Desktop Entry]
Name=xxx
Exec=~/bin/xxx %F
Terminal=false
Version=1.0
Type=Application
Icon=/home/username/software/path/xxx.exe.png
```

右键文件->属性，可以定义文件的默认打开方式，或者

```shell
/etc/gnome/defaults.list                    # 全局
~/.local/share/applications/mimeinfo.cache  # 个人
```

##### WSL 配色和修改 CapsLock

修改配色：[ColorTool](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fmicrosoft%2Fterminal%2Freleases%2Ftag%2F1708.14008) 

将 CapsLock 设置为 Ctrl

```install
Windows Registry Editor Version 5.00
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Keyboard Layout]
"Scancode Map"=hex:00,00,00,00,00,00,00,00,02,00,00,00,1D,00,3A,00,00,00,00,00
```

```uninstall
Windows Registry Editor Version 5.00
[HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Control\Keyboard Layout]
"Scancode Map"=-
```

## 使用技巧

#### 程序绑定到端口

```bash
nc -lvp 4000 | ./some_prog
socat tcp-listen:10001,reuseaddr,fork EXEC:./some_prog,pty,raw,echo=0
```

#### 取消 ssh 口令登录

```bash
vim /etc/ssh/sshd_config 
PermitEmptyPasswords no
PasswordAuthentication no
```

## 系统安全

### capabilities

Linux  2.2 增加了 capabilities 的概念，可以理解为水平权限的分离。以往如果需要某个程序的某个功能需要特权，我们就只能使用 root 来执行或者给其增加 SUID 权限，一旦这样，我们等于赋予了这个程序所有的特权，这是不满足权限最小化的要求的；在引入 capabilities 后，root 的权限被分隔成很多子权限，这就避免了滥用特权的问题，我们可以在 [capabilities(7) - Linux manual page](http://man7.org/linux/man-pages/man7/capabilities.7.html) 中看到这些特权的说明。

类似于 ping 和 nmap 这样的程序，他们其实只需要网络相关的特权即可。所以，如果你在 Kali 下查看 ping 命令的 capabilities，你会看到一个`cap_net_raw`：

```bash
$ ls -al /bin/ping
-rwxr-xr-x 1 root root 73496 Oct  5 22:34 /bin/ping
$ getcap /bin/ping
/bin/ping = cap_net_raw+ep
```

这就是为什么 kali 的 ping 命令无需设置 setuid 权限，却仍然可以以普通用户身份运行的原因。

同样，我们也可以给 nmap 增加类似的 capabilities：

```bash
sudo setcap cap_net_raw,cap_net_admin,cap_net_bind_service+eip /usr/bin/nmap
nmap --privileged -sS 192.168.1.1
```