# 📊 Japanese Restaurant Sales Report/Analysis (2024-2025)

## 🎯 Project Objective
To process 24 months of high-volume restaurant data by performing large-scale data cleaning on 220,000+ records, developing categorized tables, and creating interactive charts to identify operational trends.

## 🛠️ Data Transformation & ETL Process
The raw data was exported in a "Report-Style" format, which required significant restructuring to become a "Flat-File" database ready for analysis.

**Key Achievement:** Successfully reconciled 220,000+ rows down to 140,000 high-integrity records with **99.54% accuracy.**

### **Technical Implementation**
I developed a custom M-function in Power Query to handle the "dirty" data. This function used pattern recognition to extract dates and categories trapped in row headers.

<details>
  <summary><b>▶ Click to view Advanced ETL Logic (M Code)</b></summary>

  ```powerquery
  (pFileContents as binary) =>
  let
      Source = Excel.Workbook(pFileContents, null, true),
      Data = try Source{0}[Data] otherwise #table({}, {}),
      
      // Dynamically select and rename columns to ensure schema consistency
      ColsToKeep = List.FirstN(Table.ColumnNames(Data), 5),
      #"TrimmedTable" = Table.SelectColumns(Data, ColsToKeep),
      #"Renamed Columns" = Table.RenameColumns(#"TrimmedTable", {
          {ColsToKeep{0}, "Item"}, {ColsToKeep{1}, "Qty"}, {ColsToKeep{2}, "Subtotal"}, 
          {ColsToKeep{3}, "Discount"}, {ColsToKeep{4}, "Tax"}
      }),

      // Extracting Date headers embedded within rows via pattern recognition
      #"Added Date" = Table.AddColumn(#"Renamed Columns", "RealDate", each 
          let val = Text.From([Item]) in if val <> null and Text.Contains(val, "202") then val else null),

      // Identifying Category headers based on null quantity triggers
      #"Added Category" = Table.AddColumn(#"Added Date", "Category", each 
          let val = Text.From([Item]) in if [Qty] = null and val <> null and not Text.Contains(val, "202") then [Item] else null),

      // Down-filling attributes to flatten the hierarchical report
      #"Filled Down" = Table.FillDown(#"Added Category",{"RealDate", "Category"}),
      
      // Final data reduction: Removing header artifacts and system noise
      #"Filtered Rows" = Table.SelectRows(#"Filled Down", each 
          [Qty] <> null and [Item] <> "Item" and not Text.Contains(Text.From([Item]), "202"))
  in
      #"Filtered Rows"
```
</details>

<details>
<summary><b>▶ Click to view Main Pipeline Logic (Folder Ingestion & Tiering)</b></summary>

  ```
let
  // 1. DYNAMIC DATA INGESTION
  Source = Folder.Files("Source_Directory/Cleansed-2024-2025"),
  
  // 2. DATA ORCHESTRATION
  #"Removed Metadata" = Table.RemoveColumns(Source,{"Extension", "Date accessed", "Date modified", "Date created", "Attributes", "Folder Path"}),
  #"Invoked ETL Function" = Table.AddColumn(#"Removed Metadata", "Transform_Sample", each Transform_Sample([Content])),
  #"Expanded Data" = Table.ExpandTableColumn(#"Invoked ETL Function", "Transform_Sample", {"Item", "Qty", "Subtotal", "Discount", "Tax", "RealDate", "Category"}, {"Item", "Qty", "Subtotal", "Discount", "Tax", "RealDate", "Category"}),
  
  // 3. DATA STANDARDIZATION
  #"Set Data Types" = Table.TransformColumnTypes(#"Expanded Data",{{"RealDate", type date}, {"Subtotal", type number}, {"Tax", type number}, {"Discount", type number}, {"Qty", Int64.Type}, {"Category", type text}}),
  #"Normalize Text" = Table.TransformColumns(#"Set Data Types",{{"Item", Text.Clean, type text}}),
  #"Standardize Case" = Table.TransformColumns(#"Normalize Text",{{"Item", Text.Upper, type text}}),
  #"Clean Artifacts" = Table.ReplaceValue(#"Standardize Case",".","",Replacer.ReplaceText,{"Item"}),
  
  // 4. STRATEGIC CATEGORIZATION (Bilingual Mapping Logic)
  #"Categorized Departments" = Table.AddColumn(#"Clean Artifacts", "Department", each 
      if Text.Contains([Item], "SUSHI SASHIMI BENTO") then "Sushi Bar" 
      else if Text.Contains([Item], "ROLL COMBO") then "Sushi Bar" 
      else if Text.Contains([Category], "Roll") or Text.Contains([Category], "Sushi") then "Sushi Bar"
      else if Text.Contains([Category], "Bento") or Text.Contains([Category], "Lunch Special") then "Kitchen"
      else if Text.Contains([Category], "Drink") or Text.Contains([Category], "Sake") then "Beverage"
      else "Kitchen"),

  // 5. REVENUE ANALYSIS & TIERING
  #"Grouped for Analysis" = Table.Group(#"Categorized Departments", {"Item"}, {{"TotalRev", each List.Sum([Subtotal]), type number}, {"AllData", each _, type table}}),
  #"Final Expansion" = Table.ExpandTableColumn(#"Grouped for Analysis", "AllData", {"Qty", "Subtotal", "RealDate", "Category", "Department"}, {"Qty", "Subtotal", "RealDate", "Category", "Department"}),
  #"Assigned Menu Tier" = Table.AddColumn(#"Final Expansion", "Menu Tier", each if [TotalRev] >= 1000 then "Main" else "Minor")
in
  #"Assigned Menu Tier"
```
</details>
