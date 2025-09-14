# ZDT_DEFECT_VISION_MAS

## 新增檔案說明
```
zdt_defect_vision_mas
├── .env                               # 存放 Grafana 連線的相關資訊，詳細內容後面會說明
└── mas_ag05/backend/app/utils/
       ├── database_helpers.py         # 管理 SQLite 資料庫，記錄訓練歷史。
       ├── grafana_utils.py            # 產生 SQL 並上傳圖表至 Grafana。
       ├── mcp_server.py               # 建立 mcp server
       └── data/
            └── training_history.db    # 儲存模型訓練歷史紀錄的資料庫檔案。
```

## Grafana 安裝步驟 (使用 Docker 的方式)
* 使用 Docker 快速部署 Grafana
* 使用 Grafana Image Renderer plugins --> 用於把 Grafna 上的圖片下載到本地端，並在 Chainlit 前端介面上呈現


### 步驟一 : 建立 Docker 網路
首先，建立一個獨立的 Docker 網路，讓 Grafana 和 Image Renderer 可以在一個隔離的環境中互相溝通。

```bash
docker network create grafana-net
```

**說明**：這個指令會建立一個名為 grafana-net 的 bridge 網路。


###  步驟二 : 啟動 Grafana Image Renderer
接著，啟動 Grafana Image Renderer 容器。這個 plugins 可以把儀表板的圖表下載到本地端。

```bash
docker run -d --name=renderer --network=grafana-net --restart=unless-stopped grafana/grafana-image-renderer:latest
```

**說明：**  
| 參數 | 說明 |
| :--- | :--- |
| `-d` | 在背景執行容器。 |
| `--name=renderer` | 將容器命名為 renderer。 |
| `--network=grafana-net` | 將容器連接到我們剛建立的網路。 |
| `--restart=unless-stopped` | 除非手動停止，否則容器會在 Docker 服務啟動時自動重啟。 |


### 步驟三 : 啟動 Grafana 
最後，啟動 Grafana 容器，並將它連接到 Image Renderer。

```bash
docker run -d --name=grafana -p 3000:3000 --network=grafana-net --restart=unless-stopped -v /mnt/backups:/backups_data -e "GF_RENDERING_SERVER_URL=http://renderer:8081/render" -e "GF_RENDERING_CALLBACK_URL=http://grafana:3000/" grafana/grafana
```

**說明：**  
| 參數 | 說明 |
| :--- | :--- |
| `-d` | 在背景執行容器。 |
| `--name=grafana` | 將容器命名為 grafana。 |
| `-p 3000:3000` | 將主機的 3000 連接埠映射到容器的 3000 連接埠。 |
| `-v /mnt/backups:/backups_data` | 將主機的 /mnt/backups 目錄掛載到容器內的 /backups_data，這表示你主機 (Host) 上的 /mnt/backups 目錄，在 Grafana 容器 (Container) 內對應的路徑是 /backups_data。這能讓容器能直接存取到主機上的檔案，實現資料共享。 |
| `-e "GF_RENDERING_SERVER_URL=..."` | 設定環境變數，告訴 Grafana，Image Renderer 的內部網路位址。 |
| `-e "GF_RENDERING_CALLBACK_URL=..."` | 設定環境變數，讓 Image Renderer 知道如何回呼 Grafana。讓 Image Renderer 完成任務後，知道要把圖片回傳到哪裡。 |

## 登入 Grafana
* 安裝完成後，打開網頁瀏覽器，並訪問以下網址：`http://<你的伺服器IP>:3000 `  
(例如 : http://140.115.59.151:3000)
* 預設的登入帳號與密碼如下：  
    *  **帳號:** admin  
    *  **密碼:** admin

    ![](/images/1.png)
* 成功登入後，系統會詢問你是否要修改密碼，若是不想要修改密碼的話，點選 Skip 就可以了
    ![](/images/2.png)

## Grafana 基礎設定
首次登入 Grafana 後，會需要進行以下三項基本設定。
### 1. 新增資料來源 (Data Source)
資料來源是 Grafana 獲取數據的地方，這裡使用的資料來源是 「SQLite Database」 (為目前專案程式碼中使用的 Database )。

**(1) 點擊左側選單的 `Add new data source` 按鈕。**
![](/images/3.png)

**(2) 在搜尋框中輸入「SQLite」**
![](/images/4.png)

**(3) 點進「SQLite」之後，點擊 `Install` 進行安裝**
![](/images/5.png)

**(4) 安裝完成後，點擊左側選單的 `Data sources` 按鈕**
![](/images/6.png)

**(5)點擊 `Add new data source` 按鈕後，在搜尋框中輸入 SQLite ，並點選 SQLite，就可以成功加入 datasource 了**
![](/images/7.png)
![](/images/8.png)

**(6) 填寫SQLite Settings相關資訊**
* **Name:** 隨便設置一個名字即可
* **Path:** 填寫資料來源的連線位址。(如何填寫請見下方的 Note 區塊!)
* **記得將「Default」選項設定為開啟狀態** (如圖所示)，將這個資料來源設定為預設值。
> [!NOTE]
> * 由於 Grafana 運行在 Docker 容器這個獨立的環境中，它無法直接看到你主機上的檔案。
> * 為了解決這個問題，我們在啟動 Grafana 容器時使用了 `-v /mnt/backups:/backups_data` 指令。這個指令的作用是建立一個路徑對應 (Mapping)。
> * 可以把它理解為：我們將主機上的 /mnt/backups 資料夾，掛載到容器裡，並指定它在容器內的名字叫做 /backups_data。
> * 因此，對 Grafana 來說，它只認得容器內的路徑。所以當你要存取主機上的資料庫檔案時：
>     * 主機上的實際路徑是：`/mnt/backups/.../training_history.db`
>     * 你必須提供 Grafana 在容器內看得到的「對應路徑」(也就是在 SQLite Settings 中要填寫的 Path 欄位)：`/backups_data/.../training_history.db`


![](/images/9.png)

**(7) 儲存與測試**

填寫完 SQLite Settings 相關資訊後，點擊頁面最下方的 `Save & test` 按鈕。
如果看到` "Data source is working" `的綠色提示，代表 Grafana 可以成功讀取到你的資料庫檔案！
![](/images/10.png)

### 2. 建立 API Key
**(1) 前往 API Keys 頁面**
點擊左側選單的 `Administration` --> `User and access` --> `Service account` 按鈕。
![](/images/11.png)

(2) 先建立 Service accounts
點擊 `Add service account` 按鈕後，填寫相關資訊 : 
* **Display name (顯示名稱) :** 隨便設置一個名字即可
* **Role (角色權限) :** 選擇 Admin

完成以上設定後，直接點擊 `Create` 按鈕，即可成功建立。
![](/images/12.png)
![](/images/13.png)

**(3) 新增 API Key：**
* 點擊 `Add service account token` 按鈕，填寫 Display name 後，按下 `Generate token` 按鈕即可完成建立 API key
* 建立成功後，畫面會顯示一組專屬的 Token，**請點擊 `Copy to clipboard` 按鈕立即複製你的 Token** ( 因為這組 Token 只會完整顯示這一次！如果之後忘記的話，只能再重新建立一個新的 API key )
![](/images/14.png)
![](/images/15.png)


### 3.新增儀表板 (Dashboard)
(1) 點擊左側選單的 Dashboards 按鈕。
![](/images/16.png)

(2) 點擊 `New` --> `New dashboard` 按鈕
![](/images/17.png)

(3) 點擊 `Settings` 按鈕可以為這個 Dashborad 重新命名 (在 Title 欄位輸入即可)，設定完成後，點選 `Save Dashboard` 按鈕後即可儲存剛剛變更的資訊
![](/images/18.png)
![](/images/19.png)


## .env檔案內容說明
在根目錄創建 .env 檔案，這個檔案內需要放置的資訊有以下4個 : 
1.GRAFANA_URL
2.GRAFANA_API_KEY
3.GRAFANA_DASHBOARD_UID
4.GRAFANA_DATASOURCE_UID

### 1.GRAFANA_URL
GRAFANA_URL：填寫你登入 Grafana 時，在瀏覽器網址列所輸入的完整網址。格式如下：
```
GRAFANA_URL=http://<你的伺服器IP>:3000
```

### 2.GRAFANA_API_KEY
GRAFANA_API_KEY : 填寫剛剛建立的 API Key。格式如下：
```
GRAFANA_API_KEY=你的API Key
```

### 3.GRAFANA_DASHBOARD_UID
**(1) 進入儀表板**
* 點擊左側選單的 `Dashboard` 按鈕。
* 在 Grafana 介面中，點開你剛剛建立的那個儀表板。
![](/images/20.png)
![](/images/21.png)

**(2) 查看網址列**
瀏覽器的網址會長得像下面這樣：
```
http://<你的 Grafana 網址>/d/<這就是UID>/<儀表板名稱>
```
網址中，介於 `/d/` 和下一個 `/` 之間的那一長串字元，就是 UID。

舉例來說，如下方圖片紅框處所示，該儀表板的 UID 為 16e325c6-b17c-490d-a3d0-79c1b7042270。因此，你的 .env 檔案中應設定為：
```
GRAFANA_DASHBOARD_UID=16e325c6-b17c-490d-a3d0-79c1b7042270
```
![](/images/22.png)

### 4.GRAFANA_DATASOURCE_UID
**(1) 前往 Data source 頁面**
* 點擊左側選單的 `Data source` 按鈕。
* 再點擊你先前設定好的那個資料來源 (例如：SQLite)。
![](/images/23.png)
![](/images/24.png)

**(2) 查看網址列**
進入設定頁面後，瀏覽器的網址會看起來像這樣：
```
http://<你的 Grafana 網址>/connections/datasources/edit/<這就是UID>
```
網址最後 `/edit/` 後面的那一長串字元，就是這個資料來源的 UID。

舉例來說，如下方圖片紅框處所示，該儀表板的 UID 為 aex1fha3fpkowd。因此，你的 .env 檔案中應設定為：

```
GRAFANA_DATASOURCE_UID=aex1fha3fpkowd
```
![](/images/25.png)



