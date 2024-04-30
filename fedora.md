# flathub
flatpakにflathubのレポジトリを追加する
https://flathub.org/setup/Fedora
```
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
```

# git client

Gittyupをインストール Linux Git Client
https://flathub.org/apps/com.github.Murmele.Gittyup
```
flatpak install flathub com.github.Murmele.Gittyup
```

githubのアカウントを紐付ける際は、パスワードの欄にトークンを入れる
https://qiita.com/shiro01/items/e886aa1e4beb404f9038
tokenはちゃんと控えておくこと

リポジトリへの接続方式はsshにする
httpだと毎回tokenの入力を求められる
うっかりhttpにした場合は、Repository > Configure Repository > Remotes の部分で、sshのURLに変更(github上の)
Upstreamもそこで設定できる


# nuphyの設定
1. ここからnuphyの定義jsonファイルを落とす
   https://nuphy.com/pages/json-files-for-nuphy-keyboards
2. https://usevia.app/ にアクセスし、DesignタブのLoad Draft DefinitionからDLしたjsonファイルをアップロード
3. Configureタブで、Authorize deviceでUSB接続したnuphyを選択 → Linuxの場合だとここでエラーが出る
4. Chromeで chrome://device-log/ を入力し、エラーログに以下のようなものが出てるのを確認する(Xは適当な数字が入る)
    ```
    [17:41:40] Failed to open '/dev/hidrawX': FILE_ERROR_ACCESS_DENIED
    [17:41:40] Access denied opening device read-write, trying read-only.
    ```
5. 端末を開いて、`sudo chmod a+rw /dev/hidrawX`として、再度3を実行するとOK

参考: https://bbs.archlinux.org/viewtopic.php?id=285709
