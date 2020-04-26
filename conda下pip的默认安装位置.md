<!-- TOC -->

- [使用方式](#%E4%BD%BF%E7%94%A8%E6%96%B9%E5%BC%8F)
- [问题描述](#%E9%97%AE%E9%A2%98%E6%8F%8F%E8%BF%B0)
- [探寻与困惑](#%E6%8E%A2%E5%AF%BB%E4%B8%8E%E5%9B%B0%E6%83%91)
- [TODO 待解决](#todo-%E5%BE%85%E8%A7%A3%E5%86%B3)

<!-- /TOC -->

## 使用方式

一般而言, 我在安装包的时候偏向于使用 pip 安装, 偶尔在 pip 安装包需要编译的时候会更换为 conda 安装.

conda 通常用于建立和管理环境, 而不是管理依赖. conda 的另一个好处是预装了很多环境, 尤其是科学计算类的包(严重依赖编译).

因为自用比较多, 所以没怎么关注安装路径的问题. 直到今天遇到了多用户共享依赖的问题.

## 问题描述

依据[文档](https://docs.anaconda.com/anaconda/install/multi-user/), 配置了多人共享型的 conda.

主要过程是创建了公用的 conda 组, 在 conda 安装完之后, 将 conda 所在的目录的权限更改为这个组.
以后, 添加用户的时候, 将这些用户添加到 conda 组下, 就可以使用 conda 安装, 创建环境等了.

新建了一个 conda 环境,

```bash
conda create -p /home/anaconda2/envs/py3 python=3.8
```

然后切换到这个环境下, 使用 pip 安装包的时候, 安装路径居然是 `~/.local/`.

```bash
pip show requests
```

使用 `pip show` 可以看到包的安装路径, 因为安装到了用户目录上, 导致了虽然创建了环境, 却不能共享依赖.

## 探寻与困惑

这不太符合我的认知, 在我印象中, 安装的时候使用 `--user` 选项才会安装到用户目录中, 这使我很困惑.

```bash
$ python -m site
sys.path = [
    '/home/tt',
    '/home/anaconda2/lib/python27.zip',
    '/home/anaconda2/lib/python2.7',
    '/home/anaconda2/lib/python2.7/plat-linux2',
    '/home/anaconda2/lib/python2.7/lib-tk',
    '/home/anaconda2/lib/python2.7/lib-old',
    '/home/anaconda2/lib/python2.7/lib-dynload',
    '/home/tt/.local/lib/python2.7/site-packages',
    '/home/anaconda2/lib/python2.7/site-packages',
]
USER_BASE: '/home/tt/.local' (exists)
USER_SITE: '/home/tt/.local/lib/python2.7/site-packages' (exists)
ENABLE_USER_SITE: True
```

使用 `python -m site` 可以看到一些信息, 比如 `~/.local/` 会被包含在 `sys.path` 中, 以及用户目录的位置等.

这依赖没有减少我心中的疑惑, 同时, 我又在 pip 中找到一个选项 `-t`.

```bash
-t, --target <dir>          Install packages into <dir>. By default this will not replace existing files/folders in <dir>. Use
                            --upgrade to replace existing packages in <dir> with new versions.
```

使用 `-t` 参数可以手动指定安装位置. 似乎可以, 但依然有点麻烦, 我想看看它的默认位置是在哪.

## TODO 待解决

可能的原因:

- 权限问题:
  默认创建的新环境的权限是 751, 导致了其他用户切换到这个环境下没有 w 权限.
  无法安装包到新环境所在的目录上, 只能安装在本地目录中.

对于权限问题, 可以试着在创建环境之前使用 `umask 0002`.
