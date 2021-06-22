# JPiereをdocker-composeで試してみる

## 急いでいる人
JPiereサーバのDockerコンテナをビルドして、``docker-compose``で起動。

### Dockerコンテナのビルド

```bash:
docker build ./ -f DockerJpiere -t jpiere:8.21
```

ビルドに失敗したときにコンテナに入って原因追及。

```bash:
docker run -it --user root --name jpiere {コンテナID} bash -p
```

### docker-composeで起動　
Corei5 8MB Windows10 WSL2のdocker環境で起動までに10分程度かかる。

```bash:
docker-compose up -d
```

### 起動状況をログ確認

```bash:
docker logs -f jpiere_idempiere_1
```
次のようなメッセージが最後に出ていたら起動完了。

```bash:
略
11:59:41.010 PackInApplicationActivator.processFilePath: *** Creating list from folder migration/i4.1z [93]
11:59:41.011 PackInApplicationActivator.processFilePath: *** Creating list from folder migration/i7.1z [93]
11:59:41.016 AbstractActivator.setSummary: No zip files to process [93]
```

### ログに次のようなエラーが出ていても問題なし

#### バージョンエラーメッセージの例

```
19:02:52.425-----------> DB.isBuildOK: Build Version Error

The program assumes build version 8.2.0.202101020630, but database has build version ${env.ADEMPIERE_VERSION} 20080428-1232.
This is likely to cause hard to fix errors.
Please contact administrator. [1]
```

次のURLを参考。[【iDempiere Lab】サーバ起動時のDBビルドバージョンエラーについて](
https://www.compiere-distribution-lab.net/2016/06/28/idempiere-lab-%E3%82%B5%E3%83%BC%E3%83%90%E8%B5%B7%E5%8B%95%E6%99%82%E3%81%AEdb%E3%83%93%E3%83%AB%E3%83%89%E3%83%90%E3%83%BC%E3%82%B8%E3%83%A7%E3%83%B3%E3%82%A8%E3%83%A9%E3%83%BC%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6/)

```
このDB.javaのisBuildOK()メソッド内では、文字列情報を比較("Build DB"の値と"Build Cl"の値を比較)して異なっていたらこのメッセージが出力されるようにしているだけで、実際にデータベースのスキーマ情報を精査しているわけではありません。
```


#### バリデーションエラーメッセージの例

```
java.lang.Exception: Failed to load model validator class jpiere.plugin.groupware.base.JPierePluginGroupwareModelValidator
        at org.adempiere.base.DefaultModelValidatorFactory.newModelValidatorInstance

java.lang.Exception: Failed to load model validator class jpiere.base.plugin.org.adempiere.base.JPiereBankStatementLineModelValidator
        at org.adempiere.base.DefaultModelValidatorFactory.newModelValidatorInstance(DefaultModelValidatorFactory.java:64)

以下数件。
```

次のURLを参考。[JPiereコンフィギュレーションズの使用上の注意](https://www.compiere-distribution-lab.net/jpiere-lab/about-jpiere/jpiere-configurations/#no3)

```
JPiere ベースプラグインではモデルバリデータを使用しています。モデルバリデータに設定されているクラスが存在しないとエラーになりますので、JPiereベースプラグインを使用しない場合には、使用しないモデルバリデータを非アクティブにする必要があります。AD_ModelValidatorテーブルの下記の名称(Name)のデータをSQLで非アクティブ(IsActive='N')にして下さい。
```

## ログイン

次のURLでログインします。

http://localhost:8080/webui

## ユーザとパスワード

参考URL：[JPiereへのログイン](https://www.compiere-distribution-lab.net/jpiere-lab/install/jpiere7-1/#no5)


# Dockerコマンドメモ

## 実行中のコンテナに入る

```bash:
docker exec -it --user root {コンテナ名} bash -p

```

## docker-compose 起動、ログ確認、停止、コンテナ終了
```
docker-compose up -d
docker log -f {コンテナ名}
docker-compose stop
docker-compose down
```

## 止まってしまったコンテナに入って原因追及

```bash:
docker commit {問題のコンテナID} {新たなコンテナ名}
docker run --rm -it {新たなコンテナ名} sh
```

# はまったところ
docker-composeで試す前に、手動でインストールを試そうとしてはまったこと。

## shellの改行文字がCRLF

[OSDN > Find Software > Office/Business > CRM > JPiere > Download File List > Package JPiere Install Package > Release JPiere82 - 20210101](https://osdn.net/projects/jpiere/releases/74175)からインストール用のzipとPostgreSQLに登録するdmpファイルをダウンロードして、手順に従って、console-setup.shを動かしたのですが、

```bash:
$ ./console-setup.sh
-bash: ./console-setup.sh: /bin/sh^M: bad interpreter: No such file or directory
```

となってしまいました。shellファイルの改行文字がCRLFになっている。
気になってgrepしたら軒並み改行文字がCRLF。手順では、``sh console-setup.sh``となっているので、改行文字がCRLFでもよいのだけれど、気持ち悪いので、LFに変える。

```bash:
$ find . -type f -name '*.sh' | xargs file | grep CRLF | awk -F: '{print $1}' | xargs dos2unix
```

## 実行可能ファイルの実行権限がついていない

```bash:
$ ./console-setup.sh
Setup idempiere Server
: not foundup.sh: 4:
console-setup.sh: 6: ./idempiere: Permission denied
略
```

改行文字同様、他の実行可能ファイル、shellに実行権限がついていない。

```bash:
$ chmod u+x idempiere
$ find . -type f -name '*.sh' | xargs chmod u+x
```

# JPiereサーバとPostgreSQLサーバをそれぞれを起動してみる
## 前提
上記DockerJpiereファイルでDockerコンテナを作成済のこと。

## JPiereとPostgresqlをつなぐdocker networkを作成

```bash:
docker network create jpiere_net
```

## PostgreSQLサーバを起動

```bash:
docker run -d --name postgres --net jpiere_net -p 5432:5432 -e POSTGRES_PASSWORD=postgres postgres:12
```

## JPiereサーバを起動

```bash:
docker run -it --name jpiere -p 8080:8080 --net jpiere_net \
  -e DB_HOST=postgres \
  -e DB_PORT=5432 \
  -e DB_NAME=idempiere \
  -e DB_USER=adempiere \
  -e DB_PASS=adempiere \
  -e DB_ADMIN_PASS=postgres \
  jpiere:8.21
```

## JPiereサーバとPostgreSQLサーバがネットワークにつながっているか確認
```bash:
docker network inspect jpiere_net
```

## docker buildで次のエラーが起きたとき

```
Err:1 http://deb.debian.org/debian buster InRelease
  Temporary failure resolving 'deb.debian.org'
```

コンテナホスト側のDNSの問題。``/etc/resolv.conf``に次を記入。

```
nameserver 8.8.8.8
nameserver 8.8.4.4
```

wsl2の場合は、``/etc/wsl.conf``に次を記入し、

```
[network]
generateResolvConf = false
```

``/etc.resolv.conf``に次を記入。

```
nameserver 8.8.8.8
nameserver 8.8.4.4
```

# 参考URL

[https://hub.docker.com/r/idempiereofficial/idempiere](https://hub.docker.com/r/idempiereofficial/idempiere)

[https://github.com/idempiere/idempiere-docker](https://github.com/idempiere/idempiere-docker)

[https://mebee.info/2020/11/29/post-17806/](https://mebee.info/2020/11/29/post-17806/)

[https://www.compiere-distribution-lab.net/jpiere-lab/install/jpiere7-1/](https://www.compiere-distribution-lab.net/jpiere-lab/install/jpiere7-1/)


