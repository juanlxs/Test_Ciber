# Test_Ciber
```md
Get-Volume |
    Where-Object { $_.DriveType -eq 'Removable' } |
    Select-Object `
        @{Name='Nombre'; Expression={ $_.FileSystemLabel }},
        @{Name='Letra'; Expression={ $_.DriveLetter }},
        @{Name='Libre (GB)'; Expression={ "{0:N2}" -f ($_.SizeRemaining / 1GB) }},
        @{Name='Usado (GB)'; Expression={ "{0:N2}" -f (($_.Size - $_.SizeRemaining) / 1GB) }},
        @{Name='Total (GB)'; Expression={ "{0:N2}" -f ($_.Size / 1GB) }} |
    Format-Table -AutoSize

