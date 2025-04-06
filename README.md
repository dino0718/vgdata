# 蔬果交易資料處理系統配置說明

## vgdata202401-202502.conf 詳細解析

這個 Logstash 配置檔案用於處理蔬果市場交易資料，將 CSV 格式轉換並載入 Elasticsearch。

### 1. 輸入配置 (Input)

```
input {
  file {
    path => "/home/dino/vgdata/data/*.csv"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => plain { charset => "UTF-8" }
  }
}
```

- **path**：指定讀取 CSV 檔案的位置
- **start_position**：每次從檔案開頭開始讀取
- **sincedb_path**：設定為 "/dev/null" 表示不記錄讀取位置，每次都重新處理全部檔案
- **codec**：使用 UTF-8 編碼讀取檔案

### 2. 過濾處理 (Filter)

#### 2.1 移除非資料行
```
if [message] =~ "交易日期" or [message] =~ "市　　場：" or [message] =~ "產　　品：" or [message] =~ "日　　期" {
  drop { }
}
```
- 丟棄 CSV 檔案中的前 4 行（標題和元資料）

#### 2.2 CSV 解析
```
csv {
  separator => ","
  skip_header => true
  autodetect_column_names => false
  columns => ["日期", "市場", "產品", "上價", "中價", "下價", "平均價(元/公斤)", "增減%", "交易量(公斤)", "增減%(額外)"]
}
```
- 定義 CSV 欄位名稱，使用逗號作為分隔符

#### 2.3 資料清理
```
mutate {
  strip => ["日期", "市場", "產品", "上價", "中價", "下價", "平均價(元/公斤)", "增減%", "交易量(公斤)", "增減%(額外)"]
  
  gsub => [
    "交易量(公斤)", ",", "",  # 移除數字中的千分位逗號
    "增減%", "[^0-9.-]", "",  # 移除非數字字元
    "增減%(額外)", "[^0-9.-]", ""  # 移除非數字字元
  ]

  convert => {
    "上價" => "float"
    "中價" => "float"
    "下價" => "float"
    "平均價(元/公斤)" => "float"
    "交易量(公斤)" => "integer"
    "增減%" => "float"
    "增減%(額外)" => "float"
  }
}
```
- **strip**：移除所有欄位的前後空白
- **gsub**：移除數字中的千分位逗號和非數字字元
- **convert**：將文字欄位轉換為適當的數值類型

#### 2.4 日期轉換 (民國→西元)
```
ruby {
  code => "
    require 'date'
    begin
      year, month, day = event.get('日期').split('/')
      new_year = 1911 + year.to_i  # 民國轉西元
      event.set('日期', Date.new(new_year, month.to_i, day.to_i).to_s)
    rescue
      event.tag('_dateparsefailure')
    end
  "
}
```
- 使用 Ruby 腳本將民國日期轉換為西元日期
- 例如：民國 114/01/01 → 西元 2025-01-01

#### 2.5 日期解析
```
date {
  match => ["日期", "yyyy-MM-dd"]
  target => "@timestamp"
}
```
- 設定標準日期格式並指派給 Elasticsearch 的 @timestamp 欄位

#### 2.6 錯誤處理
```
if "_csvparsefailure" in [tags] {
  mutate { add_tag => ["CSV_PARSE_ERROR"] }
}
```
- 為解析失敗的記錄加上標籤以便除錯

### 3. 輸出配置 (Output)
```
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "vgdata202401-202502"
    document_id => "%{日期}-%{市場}-%{產品}"
    user => "elastic"
    password => "dinoliu0718"
  }
  stdout { codec => rubydebug }
}
```
- **elasticsearch**：將處理後的資料輸出到本機 Elasticsearch
- **index**：指定索引名稱為 "vgdata202401-202502"
- **document_id**：使用 "日期-市場-產品" 格式作為唯一識別碼
- **user/password**：Elasticsearch 認證資訊
- **stdout**：同時輸出到控制台以便除錯

## 使用方法

1. 確保 Elasticsearch 服務已啟動
2. 將欲處理的 CSV 檔案放入 `/home/dino/vgdata/data/` 目錄
3. 執行 Logstash 指令：
   ```
   logstash -f /home/dino/vgdata/conf/vgdata202401-202502.conf
   ```
4. 處理完成後，可在 Elasticsearch 中查詢 "vgdata202401-202502" 索引以分析資料

## 資料欄位說明

配置處理的欄位包括:
- 日期: 交易日期
- 市場: 交易市場名稱
- 產品: 蔬果名稱
- 上價/中價/下價: 當日價格範圍
- 平均價(元/公斤): 平均交易價格
- 增減%: 價格漲跌百分比
- 交易量(公斤): 交易總量
- 增減%(額外): 交易量漲跌百分比

## 使用Kibana建立蔬果交易儀表板

處理完資料後，您可以使用Kibana建立豐富的視覺化儀表板，分析蔬果市場趨勢。

### 1. 設定資料視圖 (Data View)

1. 開啟 Kibana 介面 (預設位置: http://localhost:5601)
2. 導航至 `Stack Management` > `Kibana` > `Data Views`
3. 點擊 `Create data view`
4. 設定 index pattern: `vgdata202401-202502`
5. 時間欄位選擇: `@timestamp`
6. 點擊 `Save data view to Kibana`

### 2. 建立視覺化元素

#### 2.1 價格趨勢折線圖

1. 進入 `Analytics` > `Visualize Library`
2. 選擇 `Create new visualization` > `Line`
3. 設定:
   - X軸: `@timestamp` (日期)，間隔選擇 `Daily`或`Weekly`
   - Y軸: `平均價(元/公斤)` (使用平均值聚合)
   - 細分: `產品` (Top 5-10種蔬果)
4. 儲存為 "蔬果價格趨勢"

#### 2.2 交易量熱力圖

1. 選擇 `Create new visualization` > `Heat map`
2. 設定:
   - X軸: `@timestamp` (日期)，間隔選擇 `Weekly`
   - Y軸: `產品` (使用Terms聚合)
   - 數值: `交易量(公斤)` (使用總和聚合)
3. 儲存為 "蔬果交易量熱力圖"

#### 2.3 市場交易佔比圓餅圖

1. 選擇 `Create new visualization` > `Pie`
2. 設定:
   - 切片: `市場` (使用Terms聚合)
   - 數值: `交易量(公斤)` (使用總和聚合)
3. 儲存為 "市場交易量佔比"

#### 2.4 價格區間直方圖

1. 選擇 `Create new visualization` > `Vertical bar`
2. 設定:
   - X軸: `平均價(元/公斤)` (使用Histogram聚合，間隔適當設定)
   - Y軸: `Doc Count` (預設)
   - 細分: `產品` (Top 10種蔬果)
3. 儲存為 "蔬果價格分布"

#### 2.5 漲跌幅度資料表

1. 選擇 `Create new visualization` > `Data Table`
2. 設定:
   - 分割行: `產品` (使用Terms聚合)
   - 指標: 
     - `平均價(元/公斤)` (使用平均值聚合)
     - `增減%` (使用平均值聚合)
     - `交易量(公斤)` (使用總和聚合)
     - `增減%(額外)` (使用平均值聚合)
3. 儲存為 "蔬果漲跌統計表"

### 3. 建立綜合儀表板

1. 進入 `Analytics` > `Dashboard`
2. 選擇 `Create new dashboard`
3. 點擊 `Add` 並選擇已儲存的視覺化元素
4. 拖拉調整各視覺化元素大小與位置
5. 新增篩選器 (例如特定日期區間、特定產品類別)
6. 儲存儀表板為 "蔬果市場交易分析"

### 4. 設定自動刷新與分享

1. 在儀表板右上角設定自動刷新間隔 (適合資料持續更新情境)
2. 使用分享功能產生URL或嵌入碼，便於分享給其他人
3. 可選: 設定導出PDF功能，定期產生報表

### 5. 進階分析範例

#### 5.1 特定產品季節性分析

建立視覺化元素聚焦於單一產品，分析其價格與交易量的月度變化，可透過熱圖展示季節性模式。

#### 5.2 市場比較分析

使用分割畫面視覺化元素，同時比較不同市場間相同產品的價格差異與交易量關係。

#### 5.3 價格與交易量關聯分析

使用散點圖顯示平均價格與交易量的關係，揭示價格彈性相關模式。

## 資料欄位說明

配置處理的欄位包括:
- 日期: 交易日期
- 市場: 交易市場名稱
- 產品: 蔬果名稱
- 上價/中價/下價: 當日價格範圍
- 平均價(元/公斤): 平均交易價格
- 增減%: 價格漲跌百分比
- 交易量(公斤): 交易總量
- 增減%(額外): 交易量漲跌百分比
