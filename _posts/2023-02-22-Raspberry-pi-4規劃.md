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
- [浮動IP照樣架站！NOIP DDNS 動態域名免費服務設定，遠端桌面也好用 | 老貓測3C](https://iqmore.tw/no-ip-free-dynamic-dns)
- [NO-IP 使用教學 - 程式學習筆記](https://sites.google.com/site/chengshixuexipingtai/qi-ta/no-ip-shi-yong-jiao-xue)
- [How to Install the Dynamic Update Client on Linux](https://www.noip.com/support/knowledgebase/installing-the-linux-dynamic-update-client/)
- [NO-IP 安裝設定 · Raspberry Pi 安裝設定手冊](https://lins2000.gitbooks.io/raspberry-pi-installation-guide/content/di-yi-ci-qi-dong/noip-an-zhuang-she-ding.html)

## USB隨身碟開機

> 注意！！不能使用**NOOBS**的版本,這坑很大,試了好幾天重灌好幾次文章看好幾遍,差點放棄,關鍵字：`mmc1 controller never released inhibit bit(s)`  
沒試過把sd卡複製到usb裡在啟動能不能成功,我成功是直接用Imager燒在usb裡直接開機

- 照著[這篇](https://sleeplessbeastie.eu/2022/12/16/how-to-boot-raspberry-pi-4-from-usb-ssd/)操作應該不會錯




