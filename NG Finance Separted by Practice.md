Includes first using SQL to extract the data and then using PBI to transform the data format.
```PBI
let
    Source = Sql.Database("10.121.107.88", "NGProd_108367", [Query="-- Aggregate charges per person and practice#(lf)WITH charge_sum AS (#(lf)    SELECT #(lf)        c.person_id,#(lf)        c.practice_id,#(lf)        SUM(c.amt) AS total_charge#(lf)    FROM charges c#(lf)    JOIN person p ON c.person_id = p.person_id#(lf)    GROUP BY c.person_id, c.practice_id#(lf)),#(lf)#(lf)-- Aggregate payments & adjustments per person and practice#(lf)payment_sum AS (#(lf)    SELECT #(lf)        c.person_id,#(lf)        c.practice_id,#(lf)        SUM(td.paid_amt) AS total_payment,#(lf)        SUM(td.adj_amt) AS total_adjustment#(lf)    FROM charges c#(lf)    JOIN trans_detail td ON td.charge_id = c.charge_id#(lf)    JOIN person p ON c.person_id = p.person_id#(lf)    GROUP BY c.person_id, c.practice_id#(lf)),#(lf)#(lf)-- Aggregate refunds (assumed not tied to practice)#(lf)refund_sum AS (#(lf)    SELECT #(lf)        t.person_id,#(lf)        SUM(t.tran_amt) AS refund_amt#(lf)    FROM transactions t#(lf)    JOIN tran_code_mstr tcm ON tcm.tran_code_id = t.tran_code_id#(lf)    JOIN person p ON t.person_id = p.person_id#(lf)    WHERE tcm.description LIKE '%refund%'#(lf)    GROUP BY t.person_id#(lf))#(lf)#(lf)-- Final output#(lf)SELECT#(lf)    p.person_nbr,#(lf)    pr.practice_name,#(lf)    cs.total_charge AS [Charge],#(lf)    ISNULL(ps.total_payment, 0) AS [Payment],#(lf)    ISNULL(ps.total_adjustment, 0) AS [Adjustment],#(lf)    CASE #(lf)        WHEN COALESCE(r.refund_amt, 0) > 0 THEN COALESCE(r.refund_amt, 0) - ISNULL(ps.total_adjustment, 0)#(lf)        ELSE 0#(lf)    END AS [Refund]#(lf)FROM person p#(lf)LEFT JOIN charge_sum cs ON p.person_id = cs.person_id#(lf)LEFT JOIN payment_sum ps ON p.person_id = ps.person_id AND cs.practice_id = ps.practice_id#(lf)LEFT JOIN refund_sum r ON p.person_id = r.person_id#(lf)LEFT JOIN practice pr ON cs.practice_id = pr.practice_id#(lf)ORDER BY p.person_nbr, pr.practice_name;#(lf)"]),
    #"Cleaned Text" = Table.TransformColumns(Source,{{"person_nbr", Text.Clean, type text}}),
    #"Split Column by Delimiter" = Table.SplitColumn(#"Cleaned Text", "person_nbr", Splitter.SplitTextByDelimiter("#(tab)", QuoteStyle.Csv), {"person_nbr.1", "person_nbr.2"}),
    #"Changed Type" = Table.TransformColumnTypes(#"Split Column by Delimiter",{{"person_nbr.1", Int64.Type}, {"person_nbr.2", type text}}),
    #"Removed Columns" = Table.RemoveColumns(#"Changed Type",{"person_nbr.2"}),
    #"Changed Type1" = Table.TransformColumnTypes(#"Removed Columns",{{"person_nbr.1", type text}}),
    #"Filtered Rows" = Table.SelectRows(#"Changed Type1", each ([practice_name] <> null)),
    #"Added Custom" = Table.AddColumn(#"Filtered Rows", "Practice Name", each if Text.Contains([practice_name], "AICA") then "AICA" else if Text.Contains([practice_name], "Atlas") then "ATLAS" else if Text.Contains([practice_name], "MDS") then "MDS" else if Text.Contains([practice_name], "Summit") then "SUMMIT" else null),
    #"Removed Columns1" = Table.RemoveColumns(#"Added Custom",{"practice_name"}),
    #"Unpivoted Columns" = Table.UnpivotOtherColumns(#"Removed Columns1", {"person_nbr.1", "Practice Name"}, "Attribute", "Value"),
    #"Merged Columns" = Table.CombineColumns(#"Unpivoted Columns",{"Practice Name", "Attribute"},Combiner.CombineTextByDelimiter(" ", QuoteStyle.None),"Merged"),
    #"Pivoted Column" = Table.Pivot(#"Merged Columns", List.Distinct(#"Merged Columns"[Merged]), "Merged", "Value", List.Sum),
    #"Sorted Rows" = Table.Sort(#"Pivoted Column",{{"person_nbr.1", Order.Ascending}}),
    #"Replaced Value" = Table.ReplaceValue(#"Sorted Rows",null,0,Replacer.ReplaceValue,{"AICA Charge", "AICA Adjustment", "AICA Refund", "SUMMIT Charge", "SUMMIT Payment", "SUMMIT Adjustment", "SUMMIT Refund", "AICA Payment", "ATLAS Charge", "ATLAS Adjustment", "ATLAS Refund", "ATLAS Payment", "MDS Charge", "MDS Adjustment", "MDS Refund", "MDS Payment"}),
    #"Added Custom1" = Table.AddColumn(#"Replaced Value", "TOTAL Charge", each [AICA Charge]+[ATLAS Charge]+[MDS Charge]+[SUMMIT Charge]),
    #"Added Custom2" = Table.AddColumn(#"Added Custom1", "TOTAL Payment", each [AICA Payment] + [ATLAS Payment] + [MDS Payment] + [SUMMIT Payment]),
    #"Added Custom3" = Table.AddColumn(#"Added Custom2", "TOTAL Adjustment", each [AICA Adjustment]+[ATLAS Adjustment]+[MDS Adjustment]+[SUMMIT Adjustment]),
    #"Added Custom4" = Table.AddColumn(#"Added Custom3", "TOTAL Refund", each [AICA Refund] + [ATLAS Refund] + [MDS Refund] + [SUMMIT Refund]),
    #"Added Custom5" = Table.AddColumn(#"Added Custom4", "TOTAL Collection Percentage", each if [TOTAL Charge] = 0 then 0 else -[TOTAL Payment] / [TOTAL Charge]),
    #"Changed Type2" = Table.TransformColumnTypes(#"Added Custom5",{{"TOTAL Collection Percentage", Percentage.Type}})
in
    #"Changed Type2"
```
The sql script lives in NG Data Dictionary as "Separating Refund from Adjustment.md: https://github.com/LynnLin4466/NGDataDictionary/blob/fe13970199ac363c5bc67c2535f815e54c7c5f97/Separating%20Refund%20from%20Adjustment.md."
