## **自作システムコール実装**



システムコールとは**ユーザが実行したい操作をユーザの代わりに実行するように、OSのカーネルに対して要求する手段**である。<br>


システムコールは通常の関数の動作とは異なり、CPU割り込みを発生させCPUに対してカーネルモードに切り替えることを要求し、カーネル内であらかじめ定義された場所に移動し、要求された処理をsyscallやsysenterの命令により実現する。<br>
<br>

自作のシステムコール実装は、テーブルから特定のシステムコール番号を見つけ、呼び出すべき適切なカーネル関数のアドレスが存在すれば可能である。
<br>

Linuxソースコードを取得し、改造した上でビルド、コンパイルする。<br>
そのため、仮想マシン上VM(VirtualBox)に仮想ハードドライブ(ArchLinuxのVDIファイル) :[201608 CLI 64bit](http://www.osboxes.org/archlinux/)を試す。
<br>


VM起動後、Linuxビルド時にbcコマンドが必要になる為、root権限( su 等  )で入れる
<br>`$pacman -S bc`
※ この時、`pacman -Syu`コマンドでVMをアップデートしてしまうと、過去バージョンが削除されたために発生するエラーがでるのでしないように注意



Gitからカーネルソースをcloneすると全体の把握が困難になる等の理由から、自分のカーネルバージョン(`$uname -r`:自分のバージョン確認)に対応したソースを[kernel.org](https://cdn.kernel.org/pub/linux/kernel/v4.x/)からtarballでダウンロードする方針にする。私の場合は、以下のようにした。
<br>`$ curl -O -J https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.6.7.tar. `

<br>
tarballを展開し、
<br>` tar xvf linux-4.6.7.tar.xz `
<br>` cd linux-4.6.7 `
<br>

カーネルの設定をする。ビルドパラメータを利用中のカーネルの設定をコピーする。
<br>` $ zcat /proc/config.gz > .config `
<br>` $ make oldconfig `

<br>
既存のカーネル名と競合しないために、カーネル名の設定を変更する
<br>` $ vi .config `
<br>` $ CONFIG_LOCALVERSION=“-ARCHl" `
<br>↓
<br>` $ CONFIG_LOCALVERSION="-xxxhal" `
<br>

Linuxのコードは割り込みやシステムコール等の処理の為、アーキテクチャ、プロセッサに依存する為、<br>
今回システムコールを自分で書く時x86_64を意識する。<br>
64bitのCPUにシステムコール名とシステムコールを実装する関数の名前をテーブルに書く。<br>
これはスクリプトに読まれボイラープレートコードの一部として生成される。<br>

<br>` $ vi arch/x86/entry/syscalls/syscall_64.tbl `
<br>` $ 329 common    xxxhal     sys_xxxhal `
<br>

printk( )を使い一個の文字列を引数に取り、カーネルログに書き出す自分のシステムコール関数になる。<br>
<br>` $ vi kernel/sys.c `
<br>
``` C:
$ SYSCALL_DEFINE1(xxxhal,char *,msg)
{
    char buf[256];
    long copied =     strncpy_from_user(buf,msg,sizeof(buf));
    if(copied < 0 || copied == sizeof(buf))
        return -EFAULT;
    printk(KERN_INFO "xxxhal syscall called with \"%s\"\n",buf);
    return 0;
}
```
<br>
<br>
カーネルとカーネルのモジュールをmakeコマンドで実行し、arch/x86_64/boot/bzImageファイルを生成後、<br>
make modules_installで/lib/modules/KERNEL_VERSIONにコンパイルし、コピーする。<br>

```
$ vi linux-4.6.7/deploy.sh
```

``` bash:
$#!/usr/bin/bash
$# Compile and "deploy" a new custom kernel from source on Arch Linux
$
$# Change this if you'd like. It has no relation
$# to the suffix set in the kernel config.
$SUFFIX="-xxxhal"

$# This causes the script to exit if an error occurs
$set -e

$# Compile the kernel
$make
$# Compile and install modules
$make modules_install

$# Install kernel image
$cp arch/x86_64/boot/bzImage /boot/vmlinuz-linux$SUFFIX

$# Create preset and build initramfs
$sed s/linux/linux$SUFFIX/g \
$   </etc/mkinitcpio.d/linux.preset \
$   >/etc/mkinitcpio.d/linux$SUFFIX.preset
$mkinitcpio -p linux$SUFFIX

$# Update bootloader entries with new kernels.
$grub-mkconfig -o /boot/grub/grub.cfg
```

<br>
`
$chmod u+x deploy.sh
`
<br>
`
$./deploy.sh
`
<br>
`
$reboot
`
<br>
<br>
reboot後、[Advanced Option for Arch Linux]で自分でカスタムしたカーネルを選択。
<br>
<br>
![screen shot 2017-01-30 at 16 42 05](https://cloud.githubusercontent.com/assets/17031124/22594559/e3d5cdc6-ea66-11e6-8635-50571c1406bb.png)
<br>
<br>
これでシステムコール実装は終わるが、自分の自作システムコールのテストも行う。<br>
今回はGNU Cライブラリのsyscall( )関数を使い、自作システムコールを呼び出すテストプログラムを作る。
<br>
`
$vi test.c
`

```C:
$/*
$ * Test the stephen syscall (#329)
$ */
$#define _GNU_SOURCE
$#include <unistd.h>
$#include <sys/syscall.h>
$#include <stdio.h>
$/*
$ * Put your syscall number here.
$ */
$#define SYS_stephen 329
$int main(int argc, char **argv)
${
$  if (argc <= 1) {
$    printf("Must provide a string to give to system call.\n");
$    return -1;
$  }
$  char *arg = argv[1];
$  printf("Making system call with \"%s\".\n", arg);
$  long res = syscall(SYS_stephen, arg);
$  printf("System call returned %ld.\n", res);
$  return res;
$}
```
<br>
`
$gcc -o test test.c
`
<br>
`
$ ./test 'Hello World!'
`
<br>
<br>

実行結果と、端末に出力される情報を取得するdmesgを使いシステムコールのログを抽出したもの。
<br>
<br>
![screen shot 2017-01-30 at 17 10 29](https://cloud.githubusercontent.com/assets/17031124/22594568/ef097d64-ea66-11e6-88f6-1c8268436f0b.png)


