# PowerBI-Sell-analyse-model


Purpose: The purpose of the report is to summarise weekly sales data and to visualise and analyse it as required by the sales department

It is divided into the following steps:  

1. The sales data are all in excel format, I summarize the data here using Power Query
2. Clean the data, quality check the data tables, eliminate spaces, null values, etc. and remove uncomputable data
3. Create relationships between tables
4. Write Mesure using DAX as required  


# 1.Summarise the sales data files collected on a weekly basis
Difficulty: There are approximately 300 sales data files per week, with approximately 1,000,000 data per week  
Entry point: the data deconstruction is the same for each file  
Batch process these files using M language, selecting the sheets in the data decomposition

```
Excel.Workbook([Content],true)
```

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




# 2.Create table-to-table management
Ensure data quality between individual tables, fact and dimension tables, to the extent that relationships between individual tables are successfully established, one-to-one/one-to-many relationships  
(In the chart below, all above are fact tables and all below are dimension tables)  
Fact sheets include, sales data sheets, stock sheets, order sheets, and planning sheets.
Dimension tables include, product SKU table, customer table, time date table (generated using the time function)

[//]: ![Diagramme](https://user-images.githubusercontent.com/20716430/236953993-ec3f672b-36e4-4d99-8968-02591ddaa34d.png)
<img decoding="async" src="https://user-images.githubusercontent.com/20716430/236953993-ec3f672b-36e4-4d99-8968-02591ddaa34d.png" width="50%">





# 3.Writing Mesure with DAX

```
Class Activated_DayDay = 
// 这里是所有激活的父类，所有激活都从这里继承可视
//后续继承填入day 筛选
// var R = IF(CONTAINS(RLS,RLS[User],USERPRINCIPALNAME()),1,0)
Var Role = LOOKUPVALUE(RLS[Role],RLS[User],USERPRINCIPALNAME())
var b2b = SELECTEDVALUE('Swith_B2B'[B2B])
//这个结果使用于Responbable / GTM / Tesetr / Developer 

var a =IF(b2b="On",
CALCULATE(SUM('FR激活IMEI'[Qte])
,FILTER('FR激活IMEI','FR激活IMEI'[type]="MOBILE")
,Filter('OPPO_Product_Mapping','OPPO_Product_Mapping'[Client]<>"LDU")
,FILTER('FR激活IMEI','FR激活IMEI'[激活时间]<>TODAY())
),
CALCULATE(SUM('FR激活IMEI'[Qte])
,FILTER('FR激活IMEI','FR激活IMEI'[type]="MOBILE")
,Filter('OPPO_Product_Mapping','OPPO_Product_Mapping'[Client]<>"LDU")
,FILTER('FR激活IMEI','FR激活IMEI'[激活时间]<>TODAY())
,FILTER('Settlement_Mapping','Settlement_Mapping'[KA_Short]<>"B2B")
)
)

//这个结果适用于Orange相关人员 Pierre Orange
var b = CALCULATE(SUM('FR激活IMEI'[Qte])
,FILTER('FR激活IMEI','FR激活IMEI'[type]="MOBILE")
,Filter('OPPO_Product_Mapping','OPPO_Product_Mapping'[Client]<>"LDU")
,FILTER('Settlement_Mapping','Settlement_Mapping'[KA_Short]="Orange")
,FILTER('FR激活IMEI','FR激活IMEI'[激活时间]<>TODAY())
)

//这个结果适用于SFR相关人员 Sonia SFR
var c = CALCULATE(SUM('FR激活IMEI'[Qte])
,FILTER('FR激活IMEI','FR激活IMEI'[type]="MOBILE")
,Filter('OPPO_Product_Mapping','OPPO_Product_Mapping'[Client]<>"LDU")
,FILTER('Settlement_Mapping','Settlement_Mapping'[KA_Short]="SFR")
,FILTER('FR激活IMEI','FR激活IMEI'[激活时间]<>TODAY())
)

//这个结果适用于BYT相关人员 Sebatien Bouygue
var d = CALCULATE(SUM('FR激活IMEI'[Qte])
,FILTER('FR激活IMEI','FR激活IMEI'[type]="MOBILE")
,Filter('OPPO_Product_Mapping','OPPO_Product_Mapping'[Client]<>"LDU")
,FILTER('Settlement_Mapping','Settlement_Mapping'[KA_Short]="BOUYGUES")
,FILTER('FR激活IMEI','FR激活IMEI'[激活时间]<>TODAY())
)

//这个结果适用于SalesOm1相关人员(Benoit) FnacDarty/LECLERC/La Poste/CORA/COSTCO
var SaleOm1 = CALCULATE(SUM('FR激活IMEI'[Qte])
,FILTER('FR激活IMEI','FR激活IMEI'[type]="MOBILE")
,Filter('OPPO_Product_Mapping','OPPO_Product_Mapping'[Client]<>"LDU")
,FILTER('Settlement_Mapping','Settlement_Mapping'[KA_Short] in{"FnacDarty","LECLERC","La Poste","CORA","COSTCO"})
,FILTER('FR激活IMEI','FR激活IMEI'[激活时间]<>TODAY())
)

//这个结果适用于SalesOm2相关人员(Jiaxin) Free/SystèmeU/Gpdis/LDLC
var SaleOm2 = CALCULATE(SUM('FR激活IMEI'[Qte])
,FILTER('FR激活IMEI','FR激活IMEI'[type]="MOBILE")
,Filter('OPPO_Product_Mapping','OPPO_Product_Mapping'[Client]<>"LDU")
,FILTER('Settlement_Mapping','Settlement_Mapping'[KA_Short] in{"Free","SystèmeU","Gpdis","LDLC","Cdiscount","UBALDI","RDC"})
,FILTER('FR激活IMEI','FR激活IMEI'[激活时间]<>TODAY())
)

//这个结果适用于SalesOm3相关人员(Sarah) Boulanger/Auchan/Carrefour/Casino/EléctroDépôt/INTERMARCHE
//2023/1/10 添加KA : BOUYGUES
var SaleOm3 = CALCULATE(SUM('FR激活IMEI'[Qte])
,FILTER('FR激活IMEI','FR激活IMEI'[type]="MOBILE")
,Filter('OPPO_Product_Mapping','OPPO_Product_Mapping'[Client]<>"LDU")
,FILTER('Settlement_Mapping','Settlement_Mapping'[KA_Short] in{"Boulanger","Auchan","Carrefour","Casino","EléctroDépôt","INTERMARCHE","BOUYGUES"})
,FILTER('FR激活IMEI','FR激活IMEI'[激活时间]<>TODAY())
)
//这个结果适用于SalesOm4相关人员(Edouard) Amazon/Cdiscount/UBALDI
var SaleOm4 = CALCULATE(SUM('FR激活IMEI'[Qte])
,FILTER('FR激活IMEI','FR激活IMEI'[type]="MOBILE")
,Filter('OPPO_Product_Mapping','OPPO_Product_Mapping'[Client]<>"LDU")
,FILTER('Settlement_Mapping','Settlement_Mapping'[KA_Short] in{"Amazon","RKT/E-shop"})
,FILTER('FR激活IMEI','FR激活IMEI'[激活时间]<>TODAY())
)

return
if(Role in{"Responsable","GTM","Developer","SupplyChain"},
    a,
        IF(Role ="Orange",
        b,
            IF(Role="SFR",
            c,
                IF(Role="BOUYGUES",
                d,
                    IF(Role="SaleOm1",
                    SaleOm1,
                        IF(Role="SaleOm2",
                        SaleOm2,
                            IF(Role="SaleOm3",
                            SaleOm3,
                                IF(Role="SaleOm4",
                                SaleOm4,
                                "Pass"
                                )
                            )
                        )

                    )

                ) 
            )
        )
    )
   
```
