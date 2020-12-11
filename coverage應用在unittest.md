# coverage應用在unittest

### 目的: 結合pytest and coverage兩個套件，分析目前單元測試覆蓋整個專案程式碼的比率

### 參考圖片
![1](https://i.imgur.com/mYknGsY.png)

### 定義
- run
- missing
- excluded

以上定義，在文件的搜尋列上輸入"report"，無找到相關資訊

[cmd line頁面](https://coverage.readthedocs.io/en/coverage-5.0.4/cmd.html#coverage-summary)有report的範例，但未說明missing的定義

[HTML annotation頁面](https://coverage.readthedocs.io/en/coverage-5.0.4/cmd.html#html-annotation)有以下說明:
Lines are highlighted green for executed, red for missing, and gray for excluded. The counts at the top of the file are buttons to turn on and off the highlighting.
表明綠色的段落表示被執行(executed)，因此紅色的missing就是未被執行(?)

可以用evercomm的unittest函式來測試: run and missing的差異

### 如何排除要分析的code
在要忽略的code，加上註解: # pragma: no cover
```python
a = my_function1()
if debug:   # pragma: no cover
    msg = "blah blah"
    log_message(msg, a)
b = my_function2()
```
citation: https://coverage.readthedocs.io/en/coverage-5.3/excluding.html

### How coverage.py works
For advanced use of coverage.py, or just because you are curious, it helps to understand what’s happening behind the scenes.

Coverage.py works in three phases:

- Execution: Coverage.py runs your code, and monitors it to see what lines were executed.
- Analysis: Coverage.py examines your code to determine what lines could have run.
- Reporting: Coverage.py combines the results of execution and analysis to produce a coverage number and an indication of missing execution.

citaion: https://coverage.readthedocs.io/en/coverage-5.3/howitworks.html

### coverage report的比率是怎麼算的?
(statement總數 - missing 總數)/statement總數 = 最終比率

As an example, a coverage report with 1514 statements and 901 missed statements would calculate a total percentage of (1514-901)/1514, or 40.49%.
citation: https://coverage.readthedocs.io/en/coverage-5.3/faq.html?highlight=missing#q-how-is-the-total-percentage-calculated


### 基本指令
`coverage run -m pytest` 執行單元測試
`coverage html`產生報表