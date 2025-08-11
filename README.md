# CV LaTeX 文件結構

這個專案展示了如何將 LaTeX 文件分離成不同的模組，使內容和樣式更容易管理和維護。

## 文件結構

### 主要文件
- **`main.tex`** - 原始的完整CV文件
- **`main-new.tex`** - 新的模組化主文件

### 模組化文件
- **`styles.tex`** - 樣式定義和框框樣式
- **`content.tex`** - CV內容部分

## 使用方法

### 方法1：使用模組化文件（推薦）
```bash
pdflatex main-new.tex
```

### 方法2：使用原始文件
```bash
pdflatex main.tex
```

## 模組化優勢

### 1. 樣式管理 (`styles.tex`)
- 所有框框樣式定義集中管理
- 顏色定義統一
- 頁面佈局設定
- 水印樣式定義

### 2. 內容管理 (`content.tex`)
- CV內容與樣式分離
- 易於編輯和更新內容
- 不影響樣式設定

### 3. 主文件 (`main-new.tex`)
- 使用 `\input{styles}` 引入樣式
- 使用 `\input{content}` 引入內容
- 清晰的文件結構
- 易於維護和擴展

## 框框樣式

### 可用的框框類型
- `theorembox` - 藍色定理框（用於技術堆疊儀表板）
- `definitionbox` - 綠色定義框（用於聯絡資訊）
- `examplebox` - 橙色範例框（用於快速導航和測試）
- `skillbox` - 紫色技能框（用於技能部分）
- `experiencebox` - 紅色經驗框（用於工作經驗）
- `educationbox` - 青色教育框（用於教育背景）
- `researchbox` - 棕色研究框（用於研究興趣）

### 自定義樣式
每個框框都可以通過修改 `styles.tex` 中的參數來自定義：
- `colback` - 背景顏色
- `colframe` - 邊框顏色
- `boxrule` - 邊框粗細
- `arc` - 圓角半徑
- `before skip` / `after skip` - 前後間距
- `left` / `right` / `top` / `bottom` - 內部邊距

## 擴展建議

### 添加新的框框樣式
在 `styles.tex` 中添加新的 `\newtcolorbox` 定義：

```latex
\newtcolorbox{newbox}{
    enhanced,
    colback=color!5,
    colframe=color!75,
    boxrule=0.5pt,
    arc=2pt,
    fonttitle=\bfseries,
    title=New Title,
    before skip=0.5pt,
    after skip=0.5pt,
    left=1pt,
    right=1pt,
    top=1pt,
    bottom=1pt
}
```

### 添加新的內容部分
在 `content.tex` 中使用新的框框：

```latex
\begin{newbox}
% 新內容
\end{newbox}
```

## 編譯注意事項

1. 確保所有 `.tex` 文件在同一目錄下
2. 使用 `pdflatex` 編譯器
3. 如果遇到交叉引用問題，可能需要編譯兩次
4. 檢查 `.log` 文件以獲取詳細的編譯信息

## 版本控制

建議將所有 `.tex` 文件都加入版本控制，這樣可以：
- 追蹤樣式和內容的變更歷史
- 協作編輯時更容易管理衝突
- 回滾到之前的版本
