# Ansible_Environment

## 前提条件

何かしらキーペアを作成しておいてください。

* cli(CLIのセットアップが必要です)

    ```sh
    aws ec2 create-key-pair --key-name Ansible --query 'KeyMaterial' --output text > ~/.ssh/Ansible.pem
    ```

* Management Console

    [EC2 キーペアの作成](https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/WindowsGuide/ec2-key-pairs.html#prepare-key-pair)

## 構築手順

1. 東京リージョンのCloudFormationで./cloudformation/配下の01〜04までを使用し、スタックを作成 (05から07はOption)
2. ansible-serverにセッションマネージャで接続し、以下コマンドを実行

    ```sh
    sudo su

    #ansible実行用ユーザ
    useradd ansible

    passwd ansible

    su - ansible

    #ansibleのインストール
    pip3.8 install ansible --user

    #windowsに接続する際に必要
    pip3.8 install pywinrm --user

    #ansible Command not found となる場合
    exec $SHELL -l

    #インストール確認
    ansible --version

    #動作確認
    ansible localhost -m ping
    ```

3. target-server-Windowsにセッションマネージャで接続し、以下コマンドを実行 (WindowsServerで検証したい場合)

    ```powershell
    winrm set winrm/config/service/auth '@{Basic="true"}'

    winrm set winrm/config/service '@{AllowUnencrypted="true"}'
    ```

4. target-server-WindowsにRDPでログインし、下記参考をみながらボリュームの割り当てを行う。(Option)

    [Windows で Amazon EBS ボリュームを使用できるようにします。](https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/WindowsGuide/ebs-using-volumes.html)

5. ansibleをSSH接続用ユーザにする

    ```bash
    mkdir .ssh

    chmod 700 .ssh

    touch .ssh/authorized_keys
    #作成後、キーの内容を編集してください
    #vi ~/.ssh/authorized_keysで表示された内容をコピーすれば、EC2作成時に指定したPEMキーでログインできる

    chmod 600 .ssh/authorized_keys
    ```

    [キーの作成方法参考](https://aws.amazon.com/jp/premiumsupport/knowledge-center/new-user-accounts-linux-instance/)

6. ftpのセットアップ

    本当はftpuserを作った方がいい気もするが、めんどいのでansibleユーザに接続させる(chrootにansible入れないし大丈夫？)

    * 設定ファイルの作成

        ```bash
        #vsftpdの設定
        vi /etc/vsftpd/vsftpd.conf

        #アクセス許可ユーザ一覧
        vi /etc/vsftpd/user_list
        ```

    * サービスの起動

        ```bash
        #設定を読み込ませる
        service vsftpd restart

        #状態確認
        systemctl status vsftpd.service

        #自動起動設定(起動後、FileZillaなどのftpクライアントソフトで接続可能)
        systemctl enable vsftpd.service
        ```

7. ログ出力

    * ansibleのログ

        ```bash
        sudo su

        mkdir /etc/ansible

        vi /etc/ansible/ansible.cfg

        touch /var/log/ansible.log

        chmod 666 /var/log/ansible.log
        ```

    * ftpのログ

        ```bash
        #ログ出力の設定はftpの設定ファイルに記述済み
        touch /var/log/vsftpd.log
        ```

    * sftpのログ

        ```bash
        vi /etc/rsyslog.conf

        touch /var/log/sftp

        service rsyslog restart

        vi /etc/ssh/sshd_config

        service sshd restart
        ```

8. logrotate設定

    * logrotateの設定

        ```bash
        vi /etc/logrotate.d/ansible
        vi /etc/logrotate.d/syslog
        ```

    * 設定反映

        ```bash
        sudo su

        sudo /usr/sbin/logrotate /etc/logrotate.conf
        ```

9. CloudWatch Agentの設定（Option）

    * cloudWatchをとめる

        ```bash
        /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a stop
        ```

    * 設定ファイルの作成

        ```bash
        vi /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
        ```

    * 設定ファイルを読み込ませる

        ```bash
        /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s
        ```

## 環境削除手順

1. 構築手順の１で作成したCloudFormationスタックの削除
2. CloudWatch Logsグループの削除

## メモ

### collectdインストール(CloudWatch Agentのスタンダードモードで必要)

```bash
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm -y

sudo yum install -y collectd
```

### agentプロセスチェック

```bash
ps aux | grep amazon-cloudwatch-agent
ps aux | grep amazon-ssm-agent

sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status
```

### CloudFormationヘルパースクリプトインストール

```bash
curl -O https://s3.amazonaws.com/cloudformation-examples/aws-cfn-bootstrap-py3-latest.tar.gz
python3.8 /usr/lib/python3.8/site-packages/easy_install.py --script-dir /opt/aws/bin aws-cfn-bootstrap-py3-latest.tar.gz
```

### UserDataログ確認

```bash
cat /var/log/cloud-init-output.log
```

### CloudWatch Agentステータス確認

[CloudWatch エージェントのトラブルシューティング](https://docs.aws.amazon.com/ja_jp/AmazonCloudWatch/latest/monitoring/troubleshooting-CloudWatch-Agent.html)

### AWS CLI　インストール

```bash

su - ansible

pip3 install awscli --user
```

### vsftpd　セットアップ（Playbook送信用）

本当はftpuserを作った方がいい気もするが、めんどいのでansibleユーザに接続させる

[設定値の参考](https://nwengblog.com/rhel-ftp/)

```bash
sudo su

yum install vsftpd

service vsftpd restart

systemctl status vsftpd.service

touch /var/log/vsftpd.log

systemctl enable vsftpd.service
```

### SFTPのログを出力する

```bash
sudo su

vi /etc/rsyslog.conf

touch /var/log/sftp

service rsyslog restart

vi /etc/ssh/sshd_config

service sshd restart
```

### s3 bucket作成

```bash
aws s3 mb s3://ansibleplaybookyuma --profile ansible
```

aws　プロファイル情報の場所(~/.aws/credentials)

### Auditシステム

* 概要

    セキュリティーポリシーの違反を発見するために使用できる。
    1. イベントの日時、タイプ、結果
    2. サブジェクトとオブジェクトの機密性のラベル
    3. イベントを開始したユーザーのIDとイベントの関連性
    4. Audit設定の全修正及びAuditログファイルへのアクセス試行
    5. SSH、Kerberos、及びその他の認証メカニズムの全使用
    6. 信頼できるデータベース（etc/passwdなど）への変更
    7. システムからの情報のインポート、及びシステムへの情報のエクスポートの使用
    8. ユーザID、サブジェクト及びオブジェクトラベルなどの属性に基づくincludeまたはexcludeイベント

* どのように使用されるか

    1. ファイルアクセスの監視
    2. システムコールの監視
    3. ユーザが実行したコマンドの記録
    4. システムのパス名の実行の記録
    5. セキュリティーイベントの記録
    6. イベントの検索
    7. サマリーレポートの実行
    8. ネットワークアクセスの監視

    あまりにも監査しすぎると、システムパフォーマンスの低下にもつながる

* 基本コマンド
    |コマンド|説明|
    |-------|-----------------------|
    |auditctl|Auditの動作に関する設定、Auditルールの定義を行う|
    |ausearch|Auditのログファイルから監査結果を検索|
    |aureport|Auditのログファイルから監査結果のレポートを作成|

* Auditの設定(auditctl)
    Auditには、ルールを設定できる。
    1. 制御ルール

        Auditの動作に関する設定
    2. システムコールルール

        システムコールに関する設定
    3. ファイルシステムルール

        ファイルシステムに関するルールを設定  
        ファイルに対して書き込みや属性変更等が行わなれた場合はログに出力

* 開始及び制御コマンド

    ```bash
    service auditd start

    systemctl enable auditd

    service audit reload
    ```

* 設定コマンド（一時的）

    ```bash
    auditctl -a always,exit -F arch=x86_64 -S execve -F euid=0 
    ```

    再起動すると、このコマンドで設定したルールは消える

* 設定ファイル

    永続的にルールを適用するには、`/etc/audit/audit.rules`へルールを定義するか、`/etc/audit/rules.d`に定義する必要がある。
    ※ちなみに、デフォルトではrules.dを読み込むので、その設定を向こうしてあげないとダメ(普通にrules.d使った方が良さげ)

    rulesのサンプルファイルもある。（さまざまな監査規定に即したruleを事前に定義してくれている）

    ```bash
    # cd /usr/share/audit/sample-rules/
    # cp 10-base-config.rules 30-stig.rules 31-privileged.rules 99-finalize.rules /etc/audit/rules.d/
    # augenrules --load
    ```

* 無効手順

    ```bash
    cp -f /usr/lib/systemd/system/auditd.service /etc/systemd/system/

    vi /etc/systemd/system/auditd.service
    #augenrules を含む行をコメントアウトし、auditctl -R コマンドを含む行のコメント設定を解除する。

    # #ExecStartPost=-/sbin/augenrules --load
    # ExecStartPost=-/sbin/auditctl -R /etc/audit/audit.rules

    systemctl daemon-reload

    service audit restart
    ```

* ログ確認コマンド

    ```bash
    #直近のログイン履歴（時系列で表示）
    aulast
    #直近のログイン履歴（ユーザごとに表示）
    aulastlog
    #サマリレポート
    aureport
    #ログの条件検索（下記はユーザ追加のログ確認） 
    ausearch -m ADD_USER -ui 0
    ```

* [Red Hat Enterprise Linux 8 セキュリティーの強化](https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/8/pdf/security_hardening/Red_Hat_Enterprise_Linux-8-Security_hardening-ja-JP.pdf)
* [Linuxにおける監査システム Auditについて理解する](https://qiita.com/Brutus/items/7ec3d06adf6af6ca24b7)
* [RHELのAudit設定（ファイルアクセス監査）](https://qiita.com/ch7821/items/03bd936dd4cb070001b5)

### 負荷確認

* [Linuxパフォーマンス調査などで使うコマンドメモ](https://qiita.com/toshihirock/items/0e0b20064730469e93e6)
* [Linuxで手軽にCPU/メモリの負荷をかける方法](https://qiita.com/keita0322/items/8fba88debe66fa8d2b39)

### 語句

* デーモン  
    メモリ上で常駐している常駐プログラムのUNIX系OSにおける呼び名  
    WindowsやLinuxではサービスと呼ぶ  

* dnf  
    rhel8から実装されているyumのversion4のこと。  
    yumコマンドを実行すると、dnf.logにログがたまる  
