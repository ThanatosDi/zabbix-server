# zabbix-server

Zabbix 監控伺服器的 docker compose repository

對外連線透過 Cloudflare 的 cloudflared tunnel 進行穿透連線，主機防火牆不開放 80、443 port

對外僅需要開放 10051

如有需要 phpmyadmin 或其他服務請查看 docker-compose.plugin.yml

# 安裝流程
複製 .env 
```bash
cp .env.example .env
```
修改 .env 檔案中的資料 (此步驟不多加闡述，請自行修改)

啟動 Zabbix 伺服器及 UI
```bash
docker compose up -d mariadb zabbix-server zabbix-frontend tunnel
```
Zabbix Server 的自體監控是安裝在 host 環境中  
請依照 https://www.zabbix.com/download 進行版本選擇及系統選擇安裝  
`ZABBIX COMPONENT` 要選擇 `AGENT2`

然後在 Zabbix UI 中的 `Zabbix Server` 調整設定為如圖片所示  
DSN Name: `host.docker.internal`  
改由 DNS 連線  
![image](https://github.com/user-attachments/assets/60500a5e-a93d-4430-a4fb-a73964d66208)

如有需要其他的服務，如 Grafana 等可透過 docker-compose.plugin.yml 安裝
```bash
docker compose -f docker-compose.yml -f docker-compose.plugin.yml up -d grafana
```

# 已知問題

## Unsupported charset or collation for tables 紅字警告
此步驟比較繁雜，請小心操作  
此為 MariaDB 預設的編碼及排序編碼為 utf8mb4 和 utf8mb4_general_ci  
但是 Zabbix 需要的編碼及排序編碼為 utf8mb4 和 utf8mb4_bin  
所以需要以下步驟轉換排序編碼  
先進入到 mariadb 容器中
```bash
docker compose exec mariadb bash
```
然後執行以下指令登入到 mariadb，輸入完後會要求輸入密碼
```bash
mariadb -u root -p
```
當看到開頭變為 `MariaDB [(none)]` 代表成功登入  
先修改資料庫的編碼：
```sql
use zabbix;
ALTER DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
```
產生 SQL 命令修改所有資料表的
```
SELECT CONCAT('ALTER TABLE ', table_name, ' CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;')
FROM information_schema.tables
WHERE table_schema = 'zabbix' AND table_type = 'BASE TABLE';
```
這邊的回傳結果用 IDE 將 `|` 替換掉就可以了整個複製貼到 Terminal 中執行  