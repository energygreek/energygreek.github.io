---
comments: true
date: 2025-01-16
tags: [linux,tool]
---

# Linux 文本处理三剑客
grep awk sed

## sed 将vmstat的输出转成csv

vmstat的输出如下，空格分割的。sed命令`sed 's/^  *//;s/  */,/g'  vmstat.dat > vmstat.02.csv`做了两件事：
1. 将行首的空格去掉
2. 将其他空格转成','符合csv的格式

多个sed命令用';'分隔，且共用/g命令

```
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 1  0 999208 3603968 353020 8258580    0    0    22   166 1016    2  4  1 94  0  0  0
```
处理后
```
procs,-----------memory----------,---swap--,-----io----,-system--,-------cpu-------
r,b,swpd,free,buff,cache,si,so,bi,bo,in,cs,us,sy,id,wa,st,gu
1,0,999208,3603968,353020,8258580,0,0,22,166,1016,2,4,1,94,0,0,0
```

## awk 遍历

我之前的awk通常用来打印某一列的内容，例如`cat vmstat.dat | awk -F' ' '{print $1}'` 打印$1第一列, -F设定分隔符。除此之外还能遍历查找。
例如查找'TARGET'
```sh
awk '{for(i=1;i<=NF;i++)if($i=="TARGET"){print "TARGET located at row: " NR " col: " i }}' 
```

这里NR表示当前的`行`， NF代表当前的`列`。 NR还可以用来指定要处理的行号，例如vmstat中有两个header。例如
1. 只打印第二行的命令是`awk '(NR==2){print $0}'`
2. 要知道'id'在哪一列，则遍历第二行的每一列数据 `awk '(NR==2){for(i=1;i<NF;i++) if($i=="id"){print i}'` 会打印'15'
3. 要打印id这列的第一行的值, `awk '(NR==2){for(i=1;i<NF;i++) if($i=="id"){getline; print $i}'` 会打印'94'

要打印id这列的第N行的值时，就循环调用getline N次。

## 使用python库 openpyxl 处理excel

体验了 openpyxl, 接口设计非常自然， 功能也非常全， 包括合并格子、字体、颜色。典型用法如下
```py
import openpyxl


workbook = openpyxl.load_workbook('output.xlsx')
print(workbook.sheetnames)

sheet = workbook['Sheet']
print(sheet.max_row) #总行数
print(sheet.max_column) #总列数
print(sheet.dimensions) #左上角:右下角

cell1 = sheet['A1']
print(cell1.row) #1
print(cell1.column) #A
print(cell1.coordinate) #A1

# 单元格的内容
print(cell1.value)

# 遍历
#先行后列
for row in sheet.iter_rows():
  for c in row:
    print(f"[{c.row}][{c.column}] = {c.value}")

#先列后行
for col in sheet.iter_cols():
  for r in col:
    print(f"[{r.row}][{r.column}] = {r.value}")

# 保存
workbook.save('test.xlsx')
```

### 插入图片
openpyxl还能插入图片，感觉非常棒

```py
from openpyxl import Workbook
from openpyxl.drawing.image import Image

wb = Workbook()
ws = wb.active
ws['A1'] = 'You should see three logos below'

# create an image
img = Image('logo.png')

# add to worksheet and anchor next to cells
ws.add_image(img, 'A1')
wb.save('logo.xlsx')
```
https://openpyxl.readthedocs.io/en/stable/images.html


## xlwings
Currently, xlwings can only run as a remote interpreter on Linux, see: https://docs.xlwings.org/en/stable/remote_interpreter.html
There are plans to add read/write capabilities, but for now you have to use one of OpenPyXL, XlsxWriter, or pyxlsb,

## Libreoffice
类比Excel使用vba来编程，Libreoffice sheet使用Basic，同时也支持 [python](https://help.libreoffice.org/latest/lo/text/sbasic/python/python_programming.html?&DbPAR=BASIC&System=UNIX)

在chatgpt帮助下写了个url转图片的BASIC脚本
```basic
REM  *****  BASIC  *****
Option Explicit

Sub InsertAndResizeImages()                
    Dim oSheet As Object
    DIM oCell as Object 
    Dim oGraphic As Object
    Dim i As Long
    Dim imageURL As String
    Dim oSize As New com.sun.star.awt.Size
	  Dim oColumns as Object
	  Dim oRows As Object
	
	
	oSheet = ThisComponent.CurrentController.getActiveSheet()
    
    oColumns = oSheet.getColumns()
    oColumns.getByIndex(0).Width = 3200
    
    
    i = 160
    Do
                   
        oCell = oSheet.getCellByPosition(0, i)
        oRows = oSheet.getRows()
        oRows.getByIndex(i).Height = 3200
        
        imageURL = oCell.String
        If imageURL = "" Then Exit Do
        'Debug.Print oCell.String
        'MsgBox imageURL
        
        On Error GoTo NextImage
        oGraphic = ThisComponent.createInstance("com.sun.star.drawing.GraphicObjectShape") 
        oGraphic.GraphicURL = imageURL
        oGraphic.Position = oCell.Position

		oSheet.DrawPage.add(oGraphic)
		
		oSize.Width = 3200
        oSize.Height = 3200
        oGraphic.Size = oSize
        
        Wait 500  ' Pauses for 0.5 seconds

	Exit Do
NextImage:
        i = i + 1
    Loop
        
End Sub
```

### libreoffice 也支持 python
libreoffice 的UNO API是独立于编程语言的，BASIC可以直接用 `ThisComponent`，而python需要通过'XSCRIPTCONTEXT'来用，获取sheet的BOOK对象方法是`doc = XSCRIPTCONTEXT.getDocument()`

有个Basic和Python的[对比写法](https://help.libreoffice.org/latest/lo/text/sbasic/guide/read_write_values.html?DbPAR=BASIC#bm_id41582391760114)，总结是很相似。用python重写BASIC代码就是

```python
def insert_and_resize_images():
    from com.sun.star.awt import Size
    import time

    doc = XSCRIPTCONTEXT.getDocument()
    sheet = doc.CurrentController.ActiveSheet

    columns = sheet.getColumns()
    columns.getByIndex(0).Width = 3200

    i = 160
    while True:
        cell = sheet.getCellByPosition(0, i)
        rows = sheet.getRows()
        rows.getByIndex(i).Height = 3200

        image_url = cell.String
        if image_url == "":
            break

        try:
            graphic = doc.createInstance("com.sun.star.drawing.GraphicObjectShape")
            graphic.GraphicURL = image_url
            graphic.Position = cell.Position

            sheet.DrawPage.add(graphic)

            size = Size()
            size.Width = 3200
            size.Height = 3200
            graphic.Size = size

            time.sleep(0.5)  # 0.5 seconds pause

        except:
            pass  # Continue on error

        i += 1
```

无论是excel的xlwings还是libreoffice sheet的pyUNO, 都不能独立使用，都要安装套件。 