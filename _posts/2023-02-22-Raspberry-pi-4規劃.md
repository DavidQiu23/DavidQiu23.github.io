---
title:  "Raspberry Pi 4 伺服器建置與規劃"
toc: true
toc_label: "目錄"
tags: 
    - Raspberry Pi
---

## 作業系統安裝與初始化

- **燒錄系統**：建議使用官方提供的 [Raspberry Pi Imager](https://www.raspberrypi.com/software/) 將作業系統燒錄至 SD 卡或 USB 隨身碟。
- **系統版本**：推薦選擇 **Raspberry Pi OS (64-bit)** 以獲得更好的效能。
- **中文輸入法**：安裝新酷音輸入法：
    ```bash
    sudo apt-get update
    sudo apt-get install scim-chewing
    ```
    *安裝完成後需重新啟動系統以使設定生效。*
- **啟用遠端桌面**：於設定介面開啟 **VNC** 功能。
- **環境變數設定**：
    - 編輯 `~/.profile` 檔案，將環境變數指令添加於末尾。
    - 生效範圍為當前登入使用者。**注意：特殊字元請務必使用單引號包覆。**

### 參考資料
- [Raspberry Pi OS 安裝指南](https://www.chipwaygo.com/doc/rpi_install.php)
- [如何更新與升級 Raspberry Pi OS](https://piepie.com.tw/20004/faq-how-to-update-and-upgrade-raspbian)
- [VNC 遠端桌面的解析度調整方法](https://m.clearbluedesign.com/make-headless-raspberry-pi-vnc-open-in-1080p-9f644ecc3cdd)

---

## NO-IP 動態域名 (DDNS) 設定

若網路環境為浮動 IP，可透過 NO-IP 提供穩定的域名訪問。

- **下載與安裝 NO-IP 客戶端 (DUC)**：
    ```bash
    cd /usr/local/src
    sudo wget http://www.no-ip.com/client/linux/noip-duc-linux.tar.gz
    sudo tar xzf noip-duc-linux.tar.gz
    cd noip-2.1.9-1
    sudo make
    sudo make install
    ```
- **基本管理指令**：
    - **重新配置**：`/usr/local/bin/noip2 -C`
    - **啟動服務**：`/usr/local/bin/noip2`
    - **停止服務**：先執行 `/usr/local/bin/noip2 -S` 查詢 PID，再以 `sudo kill {PID}` 終止程序。

- **SSL 憑證設定 (使用 Certbot)**：
    ```bash
    sudo apt-get update
    sudo apt-get install certbot python3-certbot-nginx
    sudo certbot --nginx
    # 測試自動續約：sudo certbot renew --dry-run
    ```

> **注意：** 若要從外部網路連回，需於路由器（Router）設定 **連接埠轉發 (Port Forwarding)**，將對應的 Port (如 VNC: 5900, Web: 80/443) 對應至樹莓派的內部 IP。

---

## Filebrowser：強大的網頁檔案管理系統

[Filebrowser](https://github.com/filebrowser/filebrowser) 提供直觀的 Web 介面來管理伺服器檔案。

- **一鍵安裝**：
    ```bash
    curl -fsSL https://raw.githubusercontent.com/filebrowser/get/master/get.sh | bash
    ```
- **初始化設定指令**：
    ```bash
    filebrowser config init                      # 產生設定檔資料庫 (filebrowser.db)
    filebrowser config set -a 0.0.0.0            # 監聽所有 IP 位址
    filebrowser config set -p 8081               # 指定 Port 號
    filebrowser config set -r /path/to/files     # 設定檔案根目錄
    filebrowser config set -b /file              # 設定 URL 前綴 (對應 Nginx 的 location)
    filebrowser users add admin admin_pass       # 建立初始管理員帳號
    filebrowser users update admin --perm.admin  # 賦予完整管理權限
    ```

---

## USB 隨身碟開機 (Boot from USB)

相較於 SD 卡，使用 USB SSD 或隨身碟開機可顯著提升讀寫速度與穩定性。

> **踩坑心得：** 請**避免使用 NOOBS 版本**進行 USB 開機設定，否則容易卡在 `mmc1 controller never released inhibit bit(s)`。建議直接使用 Raspberry Pi Imager 將系統燒錄至 USB 裝置中。

- **參考操作指南**：[How to Boot Raspberry Pi 4 from USB/SSD](https://sleeplessbeastie.eu/2022/12/16/how-to-boot-raspberry-pi-4-from-usb-ssd/)
