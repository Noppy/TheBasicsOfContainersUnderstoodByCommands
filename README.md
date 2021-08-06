# （ハンズオン）コマンドで理解するコンテナの基礎

# はじめに
このハンズオンは、シェルコマンドを利用しコンテナもどきを作りながら、コンテナの基礎である「独立した空間でプロセスを動かす」ということがどういうことかを体感し理解していただくことを目的としています。

## コンテナの基礎技術の確認
こちらを参照　--> [コンテナの作り方「Dockerは裏方で何をしているのか？](https://www.slideshare.net/zembutsu/what-isdockerdoing/27)
<img src="https://image.slidesharecdn.com/what-is-docker-doing-191213003129/95/docker-27-638.jpg?cb=1576197252">

# ハンズオンの概要
このハンズオンでは、以下の５ステップで独立させる名前空間(対象リソース)を増やしながらコンテナもどきを作っていきます。
- Step.0 単純にbashを起動し確認する
- Step.1 ユーザ名前空間を分離する
- Step.2 ホスト名の名前空間を分離する
- Step.3 プロセスの名前空間を分離する
- Step.4 マウントポイントを分離する(Dockerみたいなoverlay実装あり)


# 1.準備
- Amazon Linux2インスタンスを１つ作成する(インスタンスサイズ不問)
- コンソールを２画面起動し、それぞれでSSH接続する。(Session Managerでも可)
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
/usr/bin/bash
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
## Step.4 マウントポイントを分離する(Dockerみたいなoverlay実装あり)
dockerでは、CoW(Copy-On-Write)新しくファイルシステムを

### (1)Console1 ファイルシステムの作成とデータコピー
まず、overlayfsのベースとなるReadOnly用のファイルシステムと中身を作ります。
```shell
TMP_ROOT="./tmp_mnt/"
ROOTFS_FILE="./rootfs.dat"
echo ${NEW_ROOT} ${ROOTFS_FILE}

#実行ユーザとカレントディレクトリの確認
cd ~
id    #UID=1000(ec2-user), GID=1000(ec2-user)であることを確認
pwd   #/home/ec2-userであることを確認

#マウントするファイルシステムを作成しループバックマウントする
fallocate -l 2G ${ROOTFS_FILE}
mkfs -t xfs ${ROOTFS_FILE}
mkdir ${TMP_ROOT}
sudo mount -t xfs -o loop ${ROOTFS_FILE} ${TMP_ROOT}
sudo chown -R ec2-user:ec2-user ${TMP_ROOT}

#マウントした新しいファイルシステムに、シェルの実行に必要なデータをコピーする
#(一部Permission deniedが出るが今回は無視する)
mkdir ${TMP_ROOT}/{usr,etc,proc,dev}
cp -a /usr/bin /usr/sbin /usr/lib /usr/lib64 /usr/libexec /usr/share ${TMP_ROOT}/usr
cp -a /lib /lib64 /bin ${TMP_ROOT}/
cp /etc/{passwd,group,filesystems} ${TMP_ROOT}/etc/

sudo cp -a /root ${TMP_ROOT}/root
sudo chown -R ec2-user:ec2-user ${TMP_ROOT}/root

sudo mknod -m 666 ${TMP_ROOT}/dev/null c 1 3
sudo mknod -m 666 ${TMP_ROOT}/dev/zero c 1 5
sudo chown -R ec2-user:ec2-user ${TMP_ROOT}/dev/

#ファイルシステムをumountする
sudo umount ${TMP_ROOT}
```
### (2)Console1 overlayfsのマウントテスト
まずは名前空間を分離しない状態でoverlayfsの動作を確認します。
マウントしてファイルを追加して、追加したファイルがupperレイヤーに保管されることを確認します。
```shell
#実行ユーザとカレントディレクトリの確認
cd ~
id    #UID=1000(ec2-user), GID=1000(ec2-user)であることを確認
pwd   #/home/ec2-userであることを確認

CONTAINER_ROOT="./overlay_mnt/"
ROOTFS_FILE="./rootfs.dat"
echo ${CONTAINER_ROOT} ${ROOTFS_FILE}

#overlayでのマウント
mkdir -p ${CONTAINER_ROOT}/{lower,upper,work,merged}                 #Overlay対象ディレクトリとマウントポイントの作成
sudo mount -t xfs -o loop,ro ${ROOTFS_FILE} ${CONTAINER_ROOT}/lower  #先ほど作成したファイルシステムをReadOnlyでlowerにマウント
sudo mount -t overlay -olowerdir=${CONTAINER_ROOT}/lower,upperdir=${CONTAINER_ROOT}/upper,workdir=${CONTAINER_ROOT}/work overlay ${CONTAINER_ROOT}/merged

#マウント状況の確認
mount | grep -e '^overlay'

#オーバーレイを試してみる
cd ${CONTAINER_ROOT}/merged
pwd; ll

#マウントポイント直下にファイルシステムを作って状況を見てみる
touch hoge  
ll 
ll ../lower #lowerレイヤーに作成したファイルがあるか確認(存在しないはず)
ll ../upper #upperレイヤーに作成したファイルがあるか確認(作成したhogeファイルを確認できる)

#オーバーレイの解除
#次のchrootに向けて、一度overlayを解除します。
cd ~
sudo umount ${CONTAINER_ROOT}/merged
sudo umount ${CONTAINER_ROOT}/lower
```

### (3)Console1 新しいマウント名前空間でのプロセス起動とルートファイルシステムの変更
```shell
#Console1
#実行ユーザとカレントディレクトリの確認
cd ~
id    #UID=1000(ec2-user), GID=1000(ec2-user)であることを確認
pwd   #/home/ec2-userであることを確認

#overlayfsのマウント作業
OVERRAYDIR="./overlay_mnt/"
ROOTFS_FILE="./rootfs.dat"
echo ${OVERRAYDIR} ${ROOTFS_FILE}

#overlayでのマウント
mkdir -p ${OVERRAYDIR}/{lower,upper,work,merged}                 #Overlay対象ディレクトリとマウントポイントの作成
sudo mount -t xfs -o loop,ro ${ROOTFS_FILE} ${OVERRAYDIR}/lower  #先ほど作成したファイルシステムをReadOnlyでlowerにマウント
sudo mount -t overlay -olowerdir=${OVERRAYDIR}/lower,upperdir=${OVERRAYDIR}/upper,workdir=${OVERRAYDIR}/work overlay ${OVERRAYDIR}/merged


#新しいマウント名前空間でプロセスを起動する
unshare --user --map-root-user --uts --pid --fork --mount /usr/bin/bash

#ルートファイルシステムの変更(pivot_rootによる変更)
ROOTDIR="$(pwd)/overlay_mnt/merged"
mount --make-private /               #マウントポイントを共有しないようにする
mount --bind ${ROOTDIR} ${ROOTDIR}   #pivot_rootのためのbind処理(これを実行しないとpivot_rootがエラーになる)
cd ${ROOTDIR}                        #処理の簡素化のため新しいrootに移動
mkdir -p .old                        #既存のルートファイルシステムの待避先のマウントポイント
pivot_root . .old                    #ルートファイルシステムの変更(ここが肝)
cd /

mount -t proc proc ./proc  #/procを利用できるようにマウント
umount --lazy .old         #既存のルートファイルシステムを解除
rmdir .old                 #不要になったディレクトリの削除
```
### (4)新しいマウント名前空間でのプロセスを動かしてみる
- Console1 新しいプロセス
  ```shell
  #Console1
  ps -efH
  pwd
  mount

  #
  touch hogehage
  ls -l
  ```
- Console2 親プロセス(ホスト)
  ```shell
  mount

  cd overlay_mnt/
  ls -la merged/
  ls -la upper
  ls -la lower
  ```

# 参考資料
- https://www.youtube.com/watch?v=8fi7uSYlOdc
- https://tech.retrieva.jp/entry/2019/06/04/130134
- https://blog.framinal.life/entry/2020/04/09/183208
- https://www.slideshare.net/zembutsu/what-isdockerdoing
- https://udzura.hatenablog.jp/entry/2018/08/29/210145

overlayfs
- https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html
