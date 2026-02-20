# Test_Ciber
```md
Get-Volume | Select-Object DriveLetter, FileSystemLabel,
@{Name="SizeGB";Expression={[math]::Round($_.Size/1GB,2)}},
@{Name="FreeGB";Expression={[math]::Round($_.SizeRemaining/1GB,2)}}
