---
title: vagrant box 制作
date: 2016-10-09 23:11:22
tags:
- vagrant
category:
- PHP
- 开发环境
---

## 安装基本系统 ##

### 系统安装 ###

#### 安装方式 ####
1. 安装方式的选取上，我们选择export安装，是为了在安装过程中能选择语言
相关的文件，和实现安装最小化包的


#### 用户，域及hostname ####
1. 语言选择,尽量将zh-cn相关的和en_us相关的语言包选择上.
2. 用户的设置上面除过root的密码设置为vagrant外(方便分发), 必须创建一个
普通用户名vagrant并设置密码为vagrant(这个是vagrant本身要求的)
3. hostname和domain可以设置为vagrantbox和vagrantbox
4. 分区
    4.1 选择由于virtualbox磁盘在不整理的情况下，一直处于一个增长的情形，
因此为了后续不会受磁盘不足的影响，初期尽量将磁盘空间预分配比较大的值，
如120g, 磁盘空间分配方式采用用多少分多少的方式。
    4.2 为了后续的管理方便，分区采用一个swap加上一个根分区的形式
5. 包选择，初期安装的时候，不要安装桌面环境，后续如果需要，再行安装


### 安装后 ###
#### 更新软件包缓存，并设置网络源 ####
```bash
apt-get clean && apt-get update
```
  注: 软件源可以参考[中科大的软件源使用帮助](https://lug.ustc.edu.cn/wiki/mirrors/help/debian), 也可以直接使用其[配置生成器工具](https://mirrors.ustc.edu.cn/repogen/)
PS: 本人之前使用过debian系的和redhat系的，最近一直在用arch,
对于各个版本，中科大的源的稳定性和包更新上都做的不错，如果有可能，请
[捐赠他们](https://lug.ustc.edu.cn/wiki/lug/donate),以支持其更好发展。

#### 安装必需软件 ####
2.1 安装sudo,并配置vagrant用户无密码授权: 
```bash
apt-get install -y sudo
echo "vagrant ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
```
2.2 安装ssh服务，以方便vagrant通过ssh登陆进box 进行box的维护
```bash
aptitude install -y openssh-server
sed -i "s/PermitRootLogin /PermitRootLogin/" /etc/ssh/sshd_config
```

#### 安装非必需软件 ####
3.1 进入box部分有编写文件的时候，vim是一个相当不错的文件编辑器,
如果不想进行繁琐的配置，可以直接使用[exvim项目](http://exvim.github. io),官方有详细的安装及使用文档，并且有[中文版](http://exvim.github.io/docs-zh/)的。
```bash
    aptitude install git vim -y 
    git clone  https://github.com/exvim/main
    cd main
    bash ./unix/install.sh  #安装vundle并使用其它安装插件
    bash ./unix/replace-my-vim.sh  #替换自已的配置
```
3.2 安装zsh及配置oh-my-zsh
[oh-my-zsh](http://ohmyz.sh)是一个在维护中的结构化的zsh
配置文件，bash虽然有bash-complete作为补全的功能上补偿，
但多少有些不尽人意，而oh-my-zsh则把补全做到了极致，除此而外，
它还有其它一些功能性的插件如git插件和各种主题供选择，是一个相对
不错的项目.
```bash
    aptitude install -y zsh curl 
    sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```
PS:
原bashrc中如果配置有一些环境变量，可以直接迁移至$HOME/.zshrc,
其中zsh主题，插件等的启用也在此文件中

#### 设置公钥 ####
使用vagrant登陆进入vagrant box的时候是通过ssh密钥进行认证的，
vagrant官方有在github释放出自己的公私钥对，我们需要使用其提供的
[公钥](https://raw.githubusercontent.com/mitchellh/vagrant/master/keys/vagrant.pub)用于首次的认证.
```bash
curl https://raw.githubusercontent.com/mitchellh/vagrant/master/keys/vagrant.pub -o ~/.ssh/authorized_keys
chmod  0600 ~/.ssh
chmod 0700 ~/.ssh/authorized_keys
```
注: ~/.ssh及其中的公私钥及认证文件出于安全考虑，权限要求必须特殊设置,
否则更改会无效。

#### 安装provider相关 ####
vagrant 同virtualbox沟通的方式是通过vb提供的一些接口进行交流，如
设置内存，设置ip, 设置文件共享,设置网络方式cpu等,其中部分高级操作
如私有网络设置，添加私有网络，管理usb等需要依赖于virtualbox提供的
额外组件，在用户通过菜单中的安装额外组件后，virtualbox 会将组件工具
以光盘的形式挂载到系统,我们只需要复制出其中的文件，解压并运行其中
的virtualboxAdditional.sh,之后根据步骤安装。

#### 环境清理 ####
为了后续打包一个干净的环境，这里选择清除软件包缓存.
```bash
    debian的apt-get clean
```


### 打包 ###
1. 在配置完成以后，我们需要将该虚拟机打包成一个vagrant的box,
便于携带和使用
```bash
vagrant package
```


### 后记 ###
1.  真实的使用的中，我们不可能将镜像制作的如此简单，但作为虚拟机，
我们可以将其导出为一个基础镜像，以后在其它环境中或者需要制作其它
类应用的vagrant box的时候，可以直接导入修改后进行再次打包。
2.  针对某个场景的box(如php开发环境)制作的时候，需要在打包前安装
该场景下需要使用到的应用，并进行初步的配置，然后再进行打包。
