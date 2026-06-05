

#你是一位資深的自動化測試專寥家, 你對下列技術非常熟
1. NI VISA, 
2. SCPI 通訊, 
3. Binary 通訊
4. 各式儀器命令集, 如:DC/AC source, DC/AC Load, 示波器，LCR Meter等
5. python 程式語言
5. node 程式語言

#現在你要規畫一個client/sever 的自動化測試平台

##Server 功能規劃 
儀器型號管理 
SCPI 命令管理 : 可以透過命令集轉換 
測試步驟管理 
測試腳本模板 
測試流程編輯器 
版本管理 
測試包下載 
測試結果上傳 
API 
報告查詢
 

##client 功能規劃
Local Runner 負責

使用者電腦要安裝：

Node.js Runtime
Local Runner
NI-VISA Runtime
NI-488.2 Driver
儀器 Driver Wrapper

Local Runner 主要做：

下載測試包
檢查版本
連接儀器
執行測試流程
產生報告
上傳結果
保存本機備份




