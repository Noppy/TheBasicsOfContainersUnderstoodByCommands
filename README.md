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
sudo /usr/bin/bash
ps -fH
```
### (2)Console2 でホストから見たプロセスの状態を確認する
Console1とConsole2でpsコマンドで確認した時に、プロセスツリーやbashプロセスのPIDが一致していることが確認できます。
```shell
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
## Step.1 ユーザ空間を分離する
親プロセスから、新しい名前空間用意してコマンド実行(子プロセス起動)するためのコマンドとして、Amazon Linux2には`unshare`というコマンドが標準で実装されています。
まずはunshareコマンドでどのようなオプションがあるか確認してみます。
### (1) unshareのコマンドを確認する
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
## Step.2 ユーザー空間を分離する

## Step.3 マウントポイントを分離する




https://www.youtube.com/watch?v=8fi7uSYlOdc
