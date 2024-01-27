---
title: "ScreenとSystemdでMinecraftサーバーを管理し、GitHubにバックアップを取る"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Minecraft"]
published: false
---

# 環境
- サーバー機
  - Raspberry Pi 4 Model B / 8GB
- OS

# マイクラサーバーの構築

```zsh
mkdir ~/minecraftServer
cd ~/minecraftServer
```

buildtools.jarのダウンロード  
```zsh
wget https://hub.spigotmc.org/jenkins/job/BuildTools/lastSuccessfulBuild/artifact/target/BuildTools.jar
```

最新版をビルド
```zsh
java -jar BuildTools.jar --rev latest
```


参考 [BuildTools | SpigotMC](https://www.spigotmc.org/wiki/buildtools/)

# バックアップ用のリポジトリ作成
gitを有効化
```zsh
git init ~/minecraftServer
```

.gitignoreを作成
```zsh
nano .gitignore
```

```.gitignore
/apache-*
/BuildData
/Bukkit
/bundler
/CraftBukkit
/Spigot
/work

BuildTools.jar
BuildTools.log.txt

spigot-*.jar
```

GitHubにPush
```zsh
git branch main
git add -A
git commit -m "1st"
gh repo create minecraft-backup --private --source=. --remote=upstream
```

# サーバー操作用スクリプトの作成
スクリプトを置くためのディレクトリ作成
```zsh
mkdir ~/script
mkdir ~/script/mcsv
```

## コミットしてGitHubにPushするするスクリプト
```zsh
nano mcsv-backup_github.sh
```

```bash
#!/bin/bash

DATE=`date '+%Y-%m-%d %H:%M:%S %z'`

cd ~/minecraftServer

git add -A
git commit -m "$DATE"
git pull origin main --rebase
git push origin main
```

## 開始・停止・セーブ用スクリプト
```zsh
nano mcsv-boot.sh
```

```bash
#!/bin/bash

SCREEN_NAME="minecraftServer"

case $1 in
    save-all)
        screen -x -S "$SCREEN_NAME" -p 0 -X stuff "save-all\n"
        sleep 30;;
    start)
        /usr/bin/screen -DmS "$SCREEN_NAME" /usr/bin/java -Xmx5G -jar spigot-1.20.4.jar nogui;;
    stop)
        screen -x -S "$SCREEN_NAME" -p 0 -X stuff "say お知らせ: 30秒後にサーバーを再起動します\n"
        screen -x -S "$SCREEN_NAME" -p 0 -X stuff "save-all\n"
        sleep 30
        screen -x -S "$SCREEN_NAME" -p 0 -X stuff "stop\n"
        sleep 120;;
    *)
        echo "start | stop | save-all"
esac
```

# systemdに登録
```zsh
sudo nano /lib/systemd/system/minecraftServer.service
```

```service
[Unit]
#ユニットの説明
Description=Minecraft Server
#ユニットが開始する順序を定義
#ネットワークが立ち上がった後に開始
After=network.target

[Service]
#作業ディレクトリを指定
WorkingDirectory=/home/user/minecraftServer

#実行ユーザーを指定
User=user
Group=user

#プロセスが落ちたとき自動的に再起動
Restart=always
RestartSec=60

#ユニット開始時のコマンド
#Javaを起動
ExecStart=/usr/bin/bash /home/user/script/mcsv/mcsv-boot.sh start

#ユニット終了時のコマンド
ExecStop=/usr/bin/bash /home/user/script/mcsv/mcsv-boot.sh stop

[Install]
#マルチユーザモード（コンソールログイン）
WantedBy=multi-user.target
```

# cronの登録

## 再起動用のスクリプト
```zsh
nano mcsv-restart.sh
```

```bash
#!/bin/bash

sudo systemctl stop minecraftServer
bash ~/script/mcsv/mcsv-backup_github.sh
sudo systemctl start minecraftServer
```

## セーブ用のスクリプト
```zsh
nano mcsv-saveall.sh
```

```bash
#!/bin/bash

bash ~/script/mcsv/mcsv-boot.sh save-all
bash ~/script/mcsv/mcsv-backup_github.sh
```

## cronに登録
```zsh
crontab -e
```

```cron
0 */1 * * * /usr/bin/bash /home/user/script/mcsv/mcsv-saveall.sh > /dev/null 2>&1
0 3-7 * * wed /usr/bin/screen -x -S "minecraftServer" -p 0 -X stuff "say お知らせ: 本日7:30(AM)に再起動を行います\n"
30 7 * * wed /usr/bin/bash /home/user/script/mcsv/mcsv-restart.sh
```