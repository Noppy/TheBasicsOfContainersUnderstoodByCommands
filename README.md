# コマンドで理解するコンテナの基礎

# はじめに

# 1.準備
- Amazon Linux2インスタンスを作成する(xlarge以上のサイズで作成)
- コンソールを２画面起動し、それぞれでSSH接続する。
```shell
ssh ec2-user@<Amazon Linux2のパブリックIPアドレス>
```
- コンソールはそれぞれ以下の通り利用する
  - Console1 : コンテナ環境の実行用
  - Console2 : ホストOS上からの観測用

# 2.ハンズオン
## Step.0 普通にプロセスを起動して確認する
### (1)Console1 でプロセスを起動し、psコマンドで状態を確認する
```shell
#Console1
sudo /usr/bin/bash
ps -fH
```
### (2)Console2 で親プロセス(ホスト)から見たプロセスの状態を確認する
Console1とConsole2でpsコマンドで確認した時に、プロセスツリーやbashプロセスのPIDが一致していることが確認できます。
```shell
#Console2
ps -u ec2-user -fH
```

- Console1実行例
  ```shell
  $ bash
  $ ps -u ec2-user -fH
  UID        PID  PPID  C STIME TTY          TIME CMD
  ec2-user  3877  3843  0 12:30 ?        00:00:00 sshd: ec2-user@pts/0
  ec2-user  3878  3877  0 12:30 pts/0    00:00:00   -bash
  ec2-user  3902  3878  0 12:30 pts/0    00:00:00     bash
  ec2-user  3921  3902  0 12:30 pts/0    00:00:00       ps -u ec2-user -fH
  ec2-user  3634  3600  0 12:23 ?        00:00:00 sshd: ec2-user@pts/1
  ec2-user  3635  3634  0 12:23 pts/1    00:00:00   -bash
  ```
- Console2実行例
  ```shell
  $ ps -u ec2-user -fH
  UID        PID  PPID  C STIME TTY          TIME CMD
  ec2-user  3877  3843  0 12:30 ?        00:00:00 sshd: ec2-user@pts/0
  ec2-user  3878  3877  0 12:30 pts/0    00:00:00   -bash
  ec2-user  3902  3878  0 12:30 pts/0    00:00:00     bash
  ec2-user  3634  3600  0 12:23 ?        00:00:00 sshd: ec2-user@pts/1
  ec2-user  3635  3634  0 12:23 pts/1    00:00:00   -bash
  ec2-user  3922  3635  0 12:30 pts/1    00:00:00     ps -u ec2-user -fH
  ```
### (3)Console1のbashをexitする
```shell
exit
```
## Step.1 ユーザ名前空間を分離する
### (1) unshareのコマンドを確認する
親プロセスから、新しい名前空間用意してコマンド実行(子プロセス起動)するためのコマンドとして、Amazon Linux2には`unshare`というコマンドが標準で実装されています。
まずはunshareコマンドでどのようなオプションがあるか確認してみます。
```shell
$ unshare -h

使い方:
 unshare [options] [<program> [<argument>...]]

Run a program with some namespaces unshared from the parent.

オプション:
 -m, --mount[=<file>]      unshare mounts namespace
 -u, --uts[=<file>]        unshare UTS namespace (hostname etc)
 -i, --ipc[=<file>]        unshare System V IPC namespace
 -n, --net[=<file>]        unshare network namespace
 -p, --pid[=<file>]        unshare pid namespace
 -U, --user[=<file>]       unshare user namespace
 -C, --cgroup[=<file>]     unshare cgroup namespace
 -f, --fork        fork してから <プログラム> を起動します
     --mount-proc[=<ディレクトリ>]
                   proc ファイルシステムを最初にマウントします
                     (これには --mount の意味を含みます)
 -r, --map-root-user       map current user to root (implies --user)
     --propagation slave|shared|private|unchanged
                           modify mount propagation in mount namespace
 -s, --setgroups allow|deny  control the setgroups syscall in user namespaces

 -h, --help     このヘルプを表示して終了します
 -V, --version  バージョン情報を表示して終了します

詳しくは unshare(1) をお読みください。
```
### (2) Console1 新しいユーザー名前空間でbashを起動する
unshareコマンドを利用して、新しいユーザ名前空間を作成しbashプロセスを起動してみます。
(この段階ではユーザマッピングは実施していないです)
実行したら、idコマンドとpsコマンドで実行ユーザやプロセスの状態を確認してみます。
```shell
#Console1
unshare --user /usr/bin/bash
id
ps -efH
```
### (3) Console2 プロセスの状態を確認する
Console２からpsコマンドを実行し、Console1のpsコマンドの結果と比較します。
```shell
#Console2
ps -efH
```
- Console1実行例
  ```shell
  $ id
  uid=65534(nfsnobody) gid=65534(nfsnobody) groups=65534(nfsnobody)

  $ ps -efH
  UID        PID  PPID  C STIME TTY          TIME CMD
  nfsnobo+     2     0  0 12:22 ?        00:00:00 [kthreadd]
  nfsnobo+     4     2  0 12:22 ?        00:00:00   [kworker/0:0H]
  中略
  nfsnobo+  1934  1900  0 15:13 ?        00:00:00       sshd: ec2-user@pts/2
  nfsnobo+  1935  1934  0 15:13 pts/2    00:00:00         -bash
  nfsnobo+  2261  1935  0 15:37 pts/2    00:00:00           /usr/bin/bash
  以下略
  ```
- Console2実行例
  ```shell
  $ ps -efH
  UID        PID  PPID  C STIME TTY          TIME CMD
  root         2     0  0 12:22 ?        00:00:00 [kthreadd]
  root         4     2  0 12:22 ?        00:00:00   [kworker/0:0H]
  中略
  root      1900  3469  0 15:13 ?        00:00:00     sshd: ec2-user [priv]
  ec2-user  1934  1900  0 15:13 ?        00:00:00       sshd: ec2-user@pts/2
  ec2-user  1935  1934  0 15:13 pts/2    00:00:00         -bash
  ec2-user  2261  1935  0 15:37 pts/2    00:00:00           /usr/bin/bash
  以下略
  ```
### (4) Console2 新しいユーザー名前空間で実行ユーザをrootにマッピングする
Console1のidコマンドの通り、ユーザー名前空間の作成だけでは、`UID`/`GID`とも`65534`(nfsnobody)にしかならない。
そこで親(ホスト)のユーザ名前空間と新しいユーザー名前空間の間でのユーザマッピングを行います。ここではプロセスを起動したec2-uerユーザー(UID=1000)を、rootユーザ(UID=0)にマッピングします。
```shell
#Console２
ChildPID=<unshareから起動した/usr/bin/bashのPIDを設定>
ExeUID=$(id -u)
ExeGID=$(id -g)
echo ${ChildPID} ${ExeUID} ${ExeGID}

#マッピング情報の変更
echo deny > /proc/${ChildPID}/setgroups
echo "0 ${ExeUID} 1" > /proc/${ChildPID}/uid_map
echo "0 ${ExeGID} 1" > /proc/${ChildPID}/gid_map
```
### (5) Console1/2 idコマンド/psコマンドでマッピング後のユーザ状態を確認する
- Console1実行例
  ```shell
  $ id
  uid=0(root) gid=0(root) groups=0(root),65534(nfsnobody)
  
  $ ps -efH
  UID        PID  PPID  C STIME TTY          TIME CMD
  nfsnobo+     2     0  0 12:22 ?        00:00:00 [kthreadd]
  nfsnobo+     4     2  0 12:22 ?        00:00:00   [kworker/0:0H]
  中略
  nfsnobo+  1900  3469  0 15:13 ?        00:00:00     sshd: ec2-user [priv]
  root      1934  1900  0 15:13 ?        00:00:00       sshd: ec2-user@pts/2
  root      1935  1934  0 15:13 pts/2    00:00:00         -bash
  root      2261  1935  0 15:37 pts/2    00:00:00           /usr/bin/bash
  以下略
  ```
- Console2実行例
  ```shell
  $ ps -efH
  UID        PID  PPID  C STIME TTY          TIME CMD
  root         2     0  0 12:22 ?        00:00:00 [kthreadd]
  root         4     2  0 12:22 ?        00:00:00   [kworker/0:0H]
  中略
  root      1900  3469  0 15:13 ?        00:00:00     sshd: ec2-user [priv]
  ec2-user  1934  1900  0 15:13 ?        00:00:00       sshd: ec2-user@pts/2
  ec2-user  1935  1934  0 15:13 pts/2    00:00:00         -bash
  ec2-user  2261  1935  0 15:37 pts/2    00:00:00           /usr/bin/bash
  以下略
  ```
### (5) Console1 unshareコマンドでrootへのユーザマッピングを試す
(4)〜(5)のユーザマッピングの処理と同じことを、`unshareコマンド`の`--map-root-user`オプションで実現することができます。
```shell
#次の作業のために一旦bashを終了します。
exit
id   #ec2-userに戻っていることを確認します。

#unshareコマンドでrootユーザのマッピングもまとめて実施
unshare --user --map-root-user /usr/bin/bash

id
ps -efH
```
### (5) Console1 プロセスを終了する
次のハンズオンのためにbashを終了します。
```shell
#次の作業のために一旦bashを終了します。
exit
id   #ec2-userに戻っていることを確認します。
```

## Step.2 ホスト名の名前空間を分離する
### (1) Console1 新しいホスト名の名前空間でbashを起動しホスト名を変更する
`unshareコマンド`の`--uts`オプションで、ホスト名やNISドメイン名を管理するUTS名前空間を分離することができます。
```shell
#Console1
unshare --user --map-root-user --uts /usr/bin/bash

#hostnameを確認する
hostname

#hostnameを変更する
hostname HOGE

#hostnameを確認する
hostname
```
### (2) Console2 親プロセス(ホスト)のホスト名が変更されていないことを確認
```shell
#Console2
hostname
```
- Console1実行例
  ```shell
  # hostname
  HOGE
  ```
- Console2実行例
  ```shell
  $ hostname
  ip-172-31-20-86.ap-northeast-1.compute.internal
  ```
### (3) Console1 プロセスを終了する
次のハンズオンのためにbashを終了します。
```shell
#次の作業のために一旦bashを終了します。
exit
id   #ec2-userに戻っていることを確認します。
```
### (4) (Option) Console1 UTS名前空間を分離しない状況でhostnameを変更してみる
`--uts`オプションを付けずに親プロセスと同じUTS名前空間でホスト名を変更実行した場合は、親プロセスの空間ではrootユーザーでないことからホスト名変更はエラーとなります。
```shell
#Console1
unshare --user --map-root-user /usr/bin/bash

hostname HOGE
```
- Console1実行結果例
  ```shell
  # unshare --user --map-root-user /usr/bin/bash
  # hostname HOGE
  hostname: you must be root to change the host name
  ```


## Step.3 プロセスの名前空間を分離する
### (1) Console1 新しいプロセスの名前空間でbashを起動する
```shell
unshare --user --map-root-user --uts --pid --mount-proc --fork /usr/bin/bash

ps -efH
```
### (2)Console2 親プロセス(ホスト)でプロセスの状態を確認する
```shell
ps -efH
```
- Console1実行例
  ```shell
  # ps -efH
  UID        PID  PPID  C STIME TTY          TIME CMD
  root         1     0  0 15:52 pts/4    00:00:00 /usr/bin/bash
  root        16     1  0 15:52 pts/4    00:00:00   ps -efH
  ```
- Console2実行例
  ```shell
  $ ps -efH
  UID        PID  PPID  C STIME TTY          TIME CMD
  root         2     0  0 12:22 ?        00:00:00 [kthreadd]
  root         4     2  0 12:22 ?        00:00:00   [kworker/0:0H]
  中略
  root      2377  3469  0 15:45 ?        00:00:00     sshd: ec2-user [priv]
  ec2-user  2411  2377  0 15:45 ?        00:00:00       sshd: ec2-user@pts/4
  ec2-user  2412  2411  0 15:45 pts/4    00:00:00         -bash
  ec2-user  2529  2412  0 15:52 pts/4    00:00:00           unshare --user --map-root-user --uts --p
  ec2-user  2530  2529  0 15:52 pts/4    00:00:00             /usr/bin/bash
  以下略
  ```


## Step.4 マウントポイントを分離する


fallocate -l 100M rootfs.dat
mkfs -t xfs rootfs.dat
mkdir container_mnt
sudo mount -t xfs -o loop ./rootfs.dat ./container_mnt/

sudo  cp -aR /usr/ /lib /lib64 /etc /var ./container_mnt/




https://www.youtube.com/watch?v=8fi7uSYlOdc
https://tech.retrieva.jp/entry/2019/06/04/130134
