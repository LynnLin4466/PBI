First of all, we will need to change the data format from raw data exported from eCM/NG to a format that ranks the attorney collection efficiency by proportions.
```Power BI
let
    Source = Excel.Workbook(File.Contents("C:\Users\LLin\OneDrive - PROCO, LLC\Attachments\2020 and Greater INACTIVE Patients 1.xlsx"), null, true),
    Table1_Table = Source{[Item="Table1",Kind="Table"]}[Data],
    #"Changed Type4" = Table.TransformColumnTypes(Table1_Table,{{"_NexGen Patient ID", Int64.Type}, {"DateTreatmentStart", type date}, {"Discharge Date", type date}, {"CaseType", type text}, {"Source", type text}, {"Attorney_Most Recent", type text}, {"Attorney_Most Recent_Status", type text}, {"Attached/Unattached", type text}, {"File Status", type text}, {"File Status Calculated INACTIVE Date", type date}}),
    #"Filtered Rows2" = Table.SelectRows(#"Changed Type4", each ([CaseType] = "MVA")),
    #"Replaced Value" = Table.ReplaceValue(#"Filtered Rows2","Ped","PED",Replacer.ReplaceText,{"CaseType"}),
    #"Replaced Value1" = Table.ReplaceValue(#"Replaced Value","Well","WELL",Replacer.ReplaceText,{"CaseType"}),
    #"Filtered Rows" = Table.SelectRows(#"Replaced Value1", each [_NexGen Patient ID] <> null and [_NexGen Patient ID] <> ""),
    #"Changed Type" = Table.TransformColumnTypes(#"Filtered Rows",{{"_NexGen Patient ID", type text}}),
    #"Merged Queries" = Table.NestedJoin(#"Changed Type", {"_NexGen Patient ID"}, Merged, {"_NexGen Patient ID"}, "Merged", JoinKind.LeftOuter),
    #"Expanded Merged" = Table.ExpandTableColumn(#"Merged Queries", "Merged", {"Collection Tier Stats"}, {"Merged.Collection Tier Stats"}),
    #"Removed Columns" = Table.RemoveColumns(#"Expanded Merged",{"DateTreatmentStart", "Discharge Date", "CaseType", "Source", "Attorney_Most Recent", "Attorney_Most Recent_Status",  "File Status", "File Status Calculated INACTIVE Date"}),
    #"Grouped Rows" = Table.Group(#"Removed Columns", {"Merged.Collection Tier Stats", "Attached/Unattached"}, {{"Count", each Table.RowCount(_), Int64.Type}}),
    #"Changed Type1" = Table.TransformColumnTypes(#"Grouped Rows",{{"Merged.Collection Tier Stats", Percentage.Type}}),
    #"Filtered Rows1" = Table.SelectRows(#"Changed Type1", each [Merged.Collection Tier Stats] <> null and [Merged.Collection Tier Stats] <> ""),
    #"Sorted Rows" = Table.Sort(#"Filtered Rows1",{{"Merged.Collection Tier Stats", Order.Ascending}}),
    #"Pivoted Column" = Table.Pivot(#"Sorted Rows", List.Distinct(#"Sorted Rows"[#"Attached/Unattached"]), "Attached/Unattached", "Count", List.Sum),
    #"Added Custom" = Table.AddColumn(#"Pivoted Column", "ATTACHED PROPORTION", each [ATTACHED]/([ATTACHED]+[UNATTACHED])),
    #"Changed Type2" = Table.TransformColumnTypes(#"Added Custom",{{"ATTACHED PROPORTION", Percentage.Type}}),
    #"Added Custom1" = Table.AddColumn(#"Changed Type2", "UNATTACHED PROPORTION", each [UNATTACHED]/([ATTACHED]+[UNATTACHED])),
    #"Changed Type3" = Table.TransformColumnTypes(#"Added Custom1",{{"UNATTACHED PROPORTION", Percentage.Type}})
in
    #"Changed Type3"
```
Now based on the new data format, we can use it to assess two sample z-proportion statistical stats.
```python
# -*- coding: utf-8 -*-
"""
Created on Fri May  2 12:38:35 2025

@author: LLin
"""

import pandas as pd
import scipy.stats as stats

# Manual input of your data
data = {
    "Collection Tier": [5, 10, 15, 20, 25, 30, 35, 40, 45, 55, 60, 65, 70, 75, 80, 85, 90, 95, 100],
    "ATTACHED": [1179, 1211, 2547, 3763, 4930, 5318, 5262, 4576, 3462, 4785, 1436, 898, 689, 439, 407, 271, 226, 193, 1165],
    "UNATTACHED": [5100, 960, 958, 655, 425, 377, 354, 253, 207, 430, 176, 153, 143, 137, 271, 115, 97, 100, 1181]
}

df = pd.DataFrame(data)

# Compute totals and proportions
df["Total"] = df["ATTACHED"] + df["UNATTACHED"]
df["p1"] = df["ATTACHED"] / df["Total"]
df["p2"] = df["UNATTACHED"] / df["Total"]
df["n"] = df["Total"]

# Use pooled proportion for standard error
p_pooled = 0.5  # Since p1 + p2 = 1 always
df["z_score"] = (df["p1"] - df["p2"]) / ( (p_pooled * (1 - p_pooled) * (2 / df["n"])) ** 0.5 )
df["p_value"] = 2 * (1 - stats.norm.cdf(abs(df["z_score"])))
df["Significant"] = df["p_value"] < 0.05

# Display
pd.set_option('display.float_format', lambda x: f'{x:,.4f}')
print(df[["Collection Tier", "ATTACHED", "UNATTACHED", "z_score", "p_value", "Significant"]])

df["Effect"] = df.apply(lambda row: "Attorney Helps" if row["p1"] > row["p2"] and row["Significant"] 
                        else ("Attorney Hurts" if row["p1"] < row["p2"] and row["Significant"] 
                              else "No Significant Difference"), axis=1)

print(df[["Collection Tier", "ATTACHED", "UNATTACHED", "z_score", "p_value", "Significant", "Effect"]])

df.to_csv("attorney_effect_by_tier.csv", index=False)
```
