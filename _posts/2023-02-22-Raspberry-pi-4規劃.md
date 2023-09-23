---
title:  "Raspberry pi 4規劃"
toc: true
toc_label: "Raspberry pi 4規劃"
tags: 
    - Raspberry pi
---

## 作業系統

- 使用Raspberry Pi Imager燒錄作業系統至sd卡或隨身碟
- 選擇Raspberry Pi OS(64-bit)
- 安裝新酷音(中文輸入法) `sudo apt-get install scim-chewing`
    - 要重新啟動才會生效
- 開啟VNC
- 設定環境變數 [參考](https://pimylifeup.com/environment-variables-linux/)
    - 定義在`~/.profile`裡,指令加在最後一行後面,範圍是該登入的使用者,**特殊字元用單引號**

### 參考資料
- [安裝Raspberry Pi OS](https://www.chipwaygo.com/doc/rpi_install.php)
- [如何更新 Raspbian?](https://piepie.com.tw/20004/faq-how-to-update-and-upgrade-raspbian)
- [headless解析度調整](https://m.clearbluedesign.com/make-headless-raspberry-pi-vnc-open-in-1080p-9f644ecc3cdd)
- [如何申請非固定至固定ip](https://support.1shop.tw/%E5%A6%82%E4%BD%95%E7%94%B3%E8%AB%8B%E5%9B%BA%E5%AE%9Aip-%E4%B8%AD%E8%8F%AF%E9%9B%BB%E4%BF%A1hinet/)
- [如何讓樹莓派取得固定ip](http://yhhuang1966.blogspot.com/2021/08/ppoe-ip.html)

## NO-IP

- 安裝noip
    ```batch
    cd /usr/local/src
    wget http://www.no-ip.com/client/linux/noip-duc-linux.tar.gz
    tar xzf noip-duc-linux.tar.gz
    cd noip-2.1.9-1
    make
    make install
    ```
- 重設noip
    ```batch
    /usr/local/bin/noip2 -C
    ```
- 啟動noip
    ```batch
    /usr/local/bin/noip2
    ```
- 關閉noip
    ```batch
    /usr/local/bin/noip2 -S #先查看PID
    /usr/local/bin/noip2 -K {PID} #再砍程序
    ```

> 要使用NO-IP連進租屋處網路要設定通訊埠轉發(Port Forwarding),指定樹莓派的IP位置與要對應的Port號  
VNC:5900  
Minecraft:25565  

###  參考資料
- [浮動IP照樣架站！NOIP DDNS 動態域名免費服務設定，遠端桌面也好用](https://iqmore.tw/no-ip-free-dynamic-dns)
- [NO-IP 使用教學 - 程式學習筆記](https://sites.google.com/site/chengshixuexipingtai/qi-ta/no-ip-shi-yong-jiao-xue)
- [How to Install the Dynamic Update Client on Linux](https://www.noip.com/support/knowledgebase/installing-the-linux-dynamic-update-client/)
- [NO-IP 安裝設定 · Raspberry Pi 安裝設定手冊](https://lins2000.gitbooks.io/raspberry-pi-installation-guide/content/di-yi-ci-qi-dong/noip-an-zhuang-she-ding.html)

## [Filebrower](https://github.com/filebrowser/filebrowser) (強大的免費檔案管理介面)

- 下載Filebrower
    ```batch
    curl -fsSL https://raw.githubusercontent.com/filebrowser/get/master/get.sh | bash #會下載到/usr/local/bin底下
    ```
- 初始化filebrowser設定，可以先把檔案移到想要的目錄在執行以下指令
    ```batch
    ./filebrowser config init #會產生一個filebrowser.db的設定檔
    ./filebrowser config set -a <your ip address> #預設為127.0.0.1不能外部訪問要改為IP位置
    ./filebrowser config set -p 8081 #預設port號為8080，要修改執行這行
    ./filebrowser config set -r <想要當檔案目錄的路徑> #上傳下載都會在這個目錄下進行
    ./filebrowser users add admin admin_pass #新增第一個使用者
    ./filebrowser users update admin --perm.admin #指定為管理員權限，後續設定可直接登入此帳號用WEB設定
    ```
- [nginx https 設定](https://xujinzh.github.io/2020/11/19/cloud-by-filebrowser-and-nginx/index.html)
- [參考資料](https://blog.icephenix.com/2020/08/%E4%BD%BF%E7%94%A8filebrowser%E6%90%AD%E5%BB%BA%E4%B8%80%E5%80%8B%E6%96%87%E4%BB%B6%E4%BC%BA%E6%9C%8D%E5%99%A8/)

## USB隨身碟開機

> 注意！！不能使用**NOOBS**的版本,這坑很大,試了好幾天重灌好幾次文章看好幾遍,差點放棄  
關鍵字：`mmc1 controller never released inhibit bit(s)`,正常看到這行還是能啟動,但是NOOBS會跳兩次這行就卡在第二次的畫面了  
所以沒試過把sd卡複製到usb裡在啟動能不能成功,我成功是直接用Imager燒在usb裡直接開機

- 照著[這篇](https://sleeplessbeastie.eu/2022/12/16/how-to-boot-raspberry-pi-4-from-usb-ssd/)操作應該不會錯




