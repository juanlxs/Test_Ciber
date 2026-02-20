# Test_Ciber
```md
Get-ChildItem "C:\" -Directory -Recurse |
    Sort-Object { (Get-ChildItem $_.FullName -Recurse -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum } -Descending |
    Select-Object -First 20 FullName,
        @{Name="SizeGB";Expression={[math]::Round((Get-ChildItem $_.FullName -Recurse -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum / 1GB,2)}}

