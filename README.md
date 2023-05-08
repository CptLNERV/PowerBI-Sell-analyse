# PowerBI-Sell-analyse-model

目的：报告的目的是汇总每周的销售数据，并按照销售部门的要求进行可视化和分析
一共分成以下几步：  

 
1. 因为销售数据全部都是excel格式，我在这里将数据用Power Query进行汇总
2. 将数据进行清洗，对数据表格进行质量检测，消除空格，null值，等等不可计算数据去除


```
let
    源 = Folder.Files("C:\Users\JieLIAO\OneDrive - OPPO FRANCE\Documents - supplychain\入库IMEI&激活IMEI\FR激活IMEI"),
    DeleteColumns = Table.SelectColumns(源,{"Content"}),
    AddCustomColumn = Table.AddColumn(DeleteColumns, "自定义", each Excel.Workbook([Content],true)),
    DeleteColumns2 = Table.RemoveColumns(AddCustomColumn,{"Content"}),
    ExpandTableColumn = Table.ExpandTableColumn(DeleteColumns2, "自定义", {"Name", "Data", "Item", "Kind", "Hidden"}, {"Name", "Data", "Item", "Kind", "Hidden"}),
    #"Filtered Rows" = Table.SelectRows(ExpandTableColumn, each ([Kind] = "Sheet")),
    #"Removed Other Columns" = Table.SelectColumns(#"Filtered Rows",{"Data"}),
    #"Expanded {0}" = Table.ExpandTableColumn(#"Removed Other Columns", "Data", {"imei", "是否激活", "激活时间", "最终客户", "SKU编码", "SKU名称"}, {"imei", "是否激活", "激活时间", "最终客户", "SKU编码", "SKU名称"}),
    Distinct = Table.Distinct(#"Expanded {0}", {"imei"}),
    TransformColumnTypes = Table.TransformColumnTypes(Distinct, {{"激活时间", type datetime}}, "zh-CN"),
    TransformColumnTypes2 = Table.TransformColumnTypes(TransformColumnTypes,{{"激活时间", type date}}),
    Distinct2 = Table.Distinct(TransformColumnTypes2, {"imei"}),
    NestedJoin = Table.Nest![Uploading Diagramme de base de données.png…]()
edJoin(Distinct2, {"imei"}, All_IMEI_Sell_Through, {"IMEI"}, "IMEI_Sell_Through", JoinKind.LeftOuter),
    #"ExpandTableColumn“IMEI_Sell_Through”" = Table.ExpandTableColumn(NestedJoin, "IMEI_Sell_Through", {"Client"}, {"Client"}),
    AddColumn2 = Table.AddColumn(#"ExpandTableColumn“IMEI_Sell_Through”", "自定义", each if [Client] = null then [最终客户] else if [Client] = """" then [最终客户] else if Text.Contains([Client], "OPPO France")       then [最终客户] else [Client]),
    RenameColumns = Table.RenameColumns(AddColumn2,{{"自定义", "Final Client"}}),
    RenameColumns2 = Table.Group(RenameColumns, {"激活时间", "SKU名称", "SKU编码", "Final Client", "最终客户"}, {{"Qte", each Table.RowCount(_), Int64.Type}}),
    AddColumn3 = Table.AddColumn(RenameColumns2, "YearWeek", each if Date.WeekOfYear([激活时间],Day.Monday)-1 = 0 
  then
  (Date.Year([激活时间])-1)*100+52
  else
  Date.Year([激活时间])*100+Date.WeekOfYear([激活时间],Day.Monday)-1),
      TransformColumnTypes3 = Table.TransformColumnTypes(AddColumn3,{{"YearWeek", Int64.Type}}),
      #"Filtered Rows1" = Table.SelectRows(TransformColumnTypes3, each ([SKU名称] <> null))
  in
      #"Filtered Rows1"
```


