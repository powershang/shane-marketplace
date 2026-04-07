---
name: dd-init
description: 初始化 rtl-ddd 專案目錄結構，包含 requirement、dd、dv 子目錄及 requirement.md 模板。觸發：「初始化」、「init dd」、「建立專案結構」。
allowed-tools: Bash, Write, Read, Glob
---

# DD Init - 初始化 RTL 開發目錄

## 功能

在指定路徑建立 rtl-ddd 專案目錄結構，並產出空的 requirement.md 模板。

## 目標目錄結構

```
<專案目錄>/
└── rtl-ddd/
    ├── requirement/
    │   └── requirement.md      # 模板檔案
    ├── dd/
    │   ├── micro-arch/
    │   └── rtl/
    ├── dv/
    │   ├── testplan/
    │   └── testbench/
    └── issue/
```

## 使用方式

### 1. 在當前目錄初始化
```
/dd-init
```

### 2. 指定專案路徑初始化
```
/dd-init C:\project\my_design
```

## 執行步驟

1. **解析路徑**：
   - 如果用戶指定了路徑，使用該路徑
   - 否則使用當前工作目錄

2. **建立目錄結構**：
   - 使用 Bash mkdir 建立所需目錄
   - 目標路徑：
     - `rtl-ddd/requirement`
     - `rtl-ddd/dd/micro-arch`
     - `rtl-ddd/dd/rtl`
     - `rtl-ddd/dv/testplan`
     - `rtl-ddd/dv/testbench`
     - `rtl-ddd/issue`

3. **產出 requirement.md 模板**：
   寫入 `rtl-ddd/requirement/requirement.md`：
   ```markdown
   # Requirement: <模組名稱>

   ## 功能需求
   - [請描述模組的主要功能]

   ## 接口需求
   | 訊號名稱 | 方向 | 寬度 | 說明 |
   |----------|------|------|------|
   |          |      |      |      |

   ## 時序需求
   - [請描述時序要求]

   ## 約束條件
   - [請描述面積、功耗等約束]

   ## 備註
   - [其他需要注意的事項]
   ```

4. **報告完成**：
   - 列出建立的目錄
   - 告知如何使用 /pm、/dd、/dv

## 錯誤處理

- 如果目錄已存在，詢問用戶是否覆蓋
- 如果路徑無效，報告錯誤