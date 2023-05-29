# PowerBI-Sell-analyse-model

（All data appearing in this data model are imaginary and any similarities are entirely coincidental）  
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

Determine the file storage path and read the online Sharepoint path to:

```
Source = SharePoint.Files("https://ooo.sharepoint.com/sites/ooo-Fr", [ApiVersion = 15])
```

Successful reading of online data source file


<img decoding="async" src="https://user-images.githubusercontent.com/20716430/241747022-abe6d6b6-f9e8-4c98-b102-9f11e73c98f0.png" width="80%">\


Use the formula for parsing Excel files here


```
= Table.AddColumn(DeleteColumns, "Data", each Excel.Workbook([Content],true))

Excel.Workbook([Content],true)
```

Successfully parsing the corresponding data from the data source and then starting data cleansing and processing

The full steps are as follows:

The data is large because the original dimension is too large: for example, if the time dimension is minutes, I convert it to date and use Group By; if the geographic dimension is city, I convert it to departemnt and use group by.  
The original dimension of product dimension is SKU, I convert it to large category and use Group By
```
= Table.Group(RenameColumns, {"activetime", "SKU", "SKU code", "Final Client", "clien"}, {{"Qte", each Table.RowCount(_), Int64.Type}})
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




# 2.Create table-to-table relationship management
Ensure data quality between individual tables, fact and dimension tables, to the extent that relationships between individual tables are successfully established, one-to-one/one-to-many relationships  

Generating time dimension tables using the Time Intelligence function
```
DateTable = ADDCOLUMNS (
CALENDAR ( date(2020,1,1),date(2030,12,31) ),
"Year", YEAR ( [Date] ),
"Quarter", ROUNDUP( MONTH ( [Date] )/3,0 ),
"Month", MONTH ( [Date] ),
"Week", WEEKNUM([Date],2)-1,
"YearQuarter", YEAR ( [Date] ) & "Q" & ROUNDUP( MONTH ( [Date] )/3,0 ) ,
"Year+Month", YEAR ( [Date] ) * 100 + MONTH ( [Date] ),
"Year+Week",IF(WEEKNUM ([Date],2)-1 = 0,(YEAR([Date])-1) * 100 + 52, YEAR([Date]) * 100 + WEEKNUM ([Date],2)-1 ),
//避免第0周的出现，如果周数等于0，则Year减一，we变为52
"Weekday", WEEKDAY([Date])
)
```

(In the chart below, all above are fact tables and all below are dimension tables)  
Fact sheets include, sales data sheets, stock sheets, order sheets, and planning sheets.
Dimension tables include, product SKU table, customer table, time date table (generated using the time function)


[//]: ![Diagramme](https://user-images.githubusercontent.com/20716430/236953993-ec3f672b-36e4-4d99-8968-02591ddaa34d.png)
<img decoding="async" src="https://user-images.githubusercontent.com/20716430/236953993-ec3f672b-36e4-4d99-8968-02591ddaa34d.png" width="80%">



# 3.Writing Mesure with DAX
Here I use the techniques of an object-oriented programming language (of course we know that Dax is not an object-oriented programming language)  
I have named the most basic metric [Class sales] for ease of use later on, where we will present the different dimensions of sales



```
Class Activated_DayDay = 
// This is the parent class of all activations and all activations are inherited visually from here
// Subsequent succession fills in the day filter
// var R = IF(CONTAINS(RLS,RLS[User],USERPRINCIPALNAME()),1,0)
Var Role = LOOKUPVALUE(RLS[Role],RLS[User],USERPRINCIPALNAME())
var b2b = SELECTEDVALUE('Swith_B2B'[B2B])
//This result is used in Responbable / GTM / Tesetr / Developer 

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

// This result applies to product category A related persons group A
var b = CALCULATE(SUM('FR激活IMEI'[Qte])
,FILTER('FR激活IMEI','FR激活IMEI'[type]="MOBILE")
,Filter('OPPO_Product_Mapping','OPPO_Product_Mapping'[Client]<>"LDU")
,FILTER('Settlement_Mapping','Settlement_Mapping'[KA_Short]="Orange")
,FILTER('FR激活IMEI','FR激活IMEI'[激活时间]<>TODAY())
)

//This result applies to product category B related persons group B
var c = CALCULATE(SUM('FR激活IMEI'[Qte])
,FILTER('FR激活IMEI','FR激活IMEI'[type]="MOBILE")
,Filter('OPPO_Product_Mapping','OPPO_Product_Mapping'[Client]<>"LDU")
,FILTER('Settlement_Mapping','Settlement_Mapping'[KA_Short]="SFR")
,FILTER('FR激活IMEI','FR激活IMEI'[激活时间]<>TODAY())
)

//This result applies to product category C related persons group C
var d = CALCULATE(SUM('FR激活IMEI'[Qte])
,FILTER('FR激活IMEI','FR激活IMEI'[type]="MOBILE")
,Filter('OPPO_Product_Mapping','OPPO_Product_Mapping'[Client]<>"LDU")
,FILTER('Settlement_Mapping','Settlement_Mapping'[KA_Short]="BOUYGUES")
,FILTER('FR激活IMEI','FR激活IMEI'[激活时间]<>TODAY())
)

//This result applies to product category D related persons group D
var SaleOm1 = CALCULATE(SUM('FR激活IMEI'[Qte])
,FILTER('FR激活IMEI','FR激活IMEI'[type]="MOBILE")
,Filter('OPPO_Product_Mapping','OPPO_Product_Mapping'[Client]<>"LDU")
,FILTER('Settlement_Mapping','Settlement_Mapping'[KA_Short] in{"FnacDarty","LECLERC","La Poste","CORA","COSTCO"})
,FILTER('FR激活IMEI','FR激活IMEI'[激活时间]<>TODAY())
)

//This result applies to product category E relevant persons group E
var SaleOm2 = CALCULATE(SUM('FR激活IMEI'[Qte])
,FILTER('FR激活IMEI','FR激活IMEI'[type]="MOBILE")
,Filter('OPPO_Product_Mapping','OPPO_Product_Mapping'[Client]<>"LDU")
,FILTER('Settlement_Mapping','Settlement_Mapping'[KA_Short] in{"Free","SystèmeU","Gpdis","LDLC","Cdiscount","UBALDI","RDC"})
,FILTER('FR激活IMEI','FR激活IMEI'[激活时间]<>TODAY())
)

//This result applies to product category F relevant persons group F
//2023/1/10 添加KA : BOUYGUES
var SaleOm3 = CALCULATE(SUM('FR激活IMEI'[Qte])
,FILTER('FR激活IMEI','FR激活IMEI'[type]="MOBILE")
,Filter('OPPO_Product_Mapping','OPPO_Product_Mapping'[Client]<>"LDU")
,FILTER('Settlement_Mapping','Settlement_Mapping'[KA_Short] in{"Boulanger","Auchan","Carrefour","Casino","EléctroDépôt","INTERMARCHE","BOUYGUES"})
,FILTER('FR激活IMEI','FR激活IMEI'[激活时间]<>TODAY())
)

//This result applies to product category G relevant persons group G
var SaleOm4 = CALCULATE(SUM('FR激活IMEI'[Qte])
,FILTER('FR激活IMEI','FR激活IMEI'[type]="MOBILE")
,Filter('OPPO_Product_Mapping','OPPO_Product_Mapping'[Client]<>"LDU")
,FILTER('Settlement_Mapping','Settlement_Mapping'[KA_Short] in{"Amazon","RKT/E-shop"})
,FILTER('FR激活IMEI','FR激活IMEI'[激活时间]<>TODAY())
)

return
if(Role in{"Responsable","GTM","Developer","SupplyChain"},
    a,
        IF(Role ="b",
        b,
            IF(Role="c",
            c,
                IF(Role="d",
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

I usually have to write a dozen or more Mesures, and it is very efficient to manage them through this inheritance
We then use Class sales to inherit the results we need to calculate, for example, in the example below, to calculate the sales for the day last Friday, we introduce Class Sale in Calculate as the base for the calculation

```
Act_Last_Friday = 
CALCULATE([Class Activated_DayDay]<img width="158" alt="Mesure_Group" src="https://github.com/CptLNERV/PowerBI-Sell-analyse/assets/20716430/d8f7964d-0c01-4d8a-94be-c3df46a39486">

,FILTER('FRIMEI', 'FRIMEI'[Date]=[Last_Friday])
)
```
There are dozens of Mesures that need to be calculated like this. By modifying the calculation method in the parent class, you can achieve the purpose of batch modifying dozens of Mesures, grouping them by prefix and putting them in the same folder to achieve the purpose of quick batch management.

<img decoding="async" src="https://user-images.githubusercontent.com/20716430/237149361-00a2e690-5b25-4ac8-aebd-4163bf1e03ed.png" width="20%">
![Uploading Mesure_Group.png…]()



# 4.Creating visual objects
Select the appropriate visual object according to your needs and place the prepared Mesure into it







