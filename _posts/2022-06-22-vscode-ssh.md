---
layout: post
title: VScode ssh to Windows server 遠端寫扣！
date: 2022-06-22 21:42 +0800
tags: update ssh config vscode
---

雖然我已經是 Vim 的信徒ㄌ，但總不可能用 Vim 寫 Jupyter Notebook 吧！開好 Jupyter Notebook Server 之後寫了大概一個月，真的覺得預設的框框有夠窄有夠難寫，改成用 Jupyter Lab 寫了一個禮拜之後又覺得那個編輯器的設定真的莫名其妙（舉例：在輸入搜尋欄後按 enter，cursor 會直接 focus 在第二個結果，也就是說當你想再按一次 enter 跳到下一個結果時，第一個結果的字元直接變成換行）。於是！大概查了一下，發現 VScode 有個酷外掛叫做 Remote - SSH名稱: [Remote - SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)，讓我們可以直接連到遠端的電腦做開發！不過最大的問題是要開啟遠端主機的 SSH server，也因為有觀眾敲碗所以想說來記錄一下好ㄌ！

## Windows SSH Server Configuration

首先要感謝大學時的研究所學長嗡嗡，這段基本上是照著他的網誌做設定ㄉ🥺

### Step 1 安裝 SSH server

首先要進到設定 > 應用程式與功能 > 選用功能，檢查有沒有安裝 SSH server，如果沒有的話可以點選第二張圖上方的「新增功能」，搜尋並安裝「OpenSSH 伺服器」

![](/assets/img/ssh1.png)
![](/assets/img/ssh2.png)

### Step 2 執行 SSH server

接下來要以**系統管理員**開啟 Windows PowerShell，並輸入以下指令來開啟 server：

```
Start-Service sshd  # 開啟 SSH server
Set-Service -Name sshd -StartupType 'Automatic'  # 開機時自動啟用，擔心資安問題也可以把 'Automatic' 改成 'Manual'
```
設定好遠端主機後，可以回到現在用的電腦上，先開啟 VScode 並安裝官方的 Remote 外掛

## VScode plugin: Remote - SSH

安裝完遠端主機的 SSH server 之後，回到現在使用的電腦上，安裝 VScode 的 Remote - SSH 外掛

![](/assets/img/ssh-vscode.png)

安裝完後會發現左下角有一個藍色的框框可以點
![](/assets/img/ssh-btn.png)

點選之後會跳出一個對話筐，輸入你的遠端主機 hostname/ip
![](/assets/img/ssh-host.png)

第一次使用時需要先新增主機，按照對話筐的提示（`ssh user@hostname`）輸入、並輸入該使用者的密碼後，就可以連線到遠端主機了！
> 註：這邊的 user 是指遠端主機的使用者，密碼則是登入該位使用者的密碼

![](/assets/img/ssh-new.png)


<font color="grey" style="font-size: 24px">Reference</font>
- 大神學長的網誌：[【Windows】windows 開啟 ssh 的方式｜嗡嗡的隨手筆記](https://www.wongwonggoods.com/draft_notes/windows/windows-ssh/)
- Remote - SSH plugin 本人的連結：[Remote - SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)
