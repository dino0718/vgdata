# 資料來源
“https://amis.afa.gov.tw/m_veg/VegProdDayTransInfo.aspx”


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

