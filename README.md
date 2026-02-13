# Test_Ciber
```md

Get-Disk |
    Where-Object BusType -eq 'USB' |
    ForEach-Object {
        $disk = $_
        Get-Partition -DiskNumber $disk.Number |
            Where-Object DriveLetter -ne $null |
            ForEach-Object {
                $vol = Get-Volume -DriveLetter $_.DriveLetter -ErrorAction SilentlyContinue

                # Si Get-Volume falla, evitamos errores
                if ($null -eq $vol) {
                    return
                }

                [PSCustomObject]@{
                    Nombre       = if ($vol.FileSystemLabel) { $vol.FileSystemLabel } else { "(Sin nombre)" }
                    Letra        = $vol.DriveLetter
                    'Libre (GB)' = "{0:N2}" -f ($vol.SizeRemaining / 1GB)
                    'Usado (GB)' = "{0:N2}" -f (($vol.Size - $vol.SizeRemaining) / 1GB)
                    'Total (GB)' = "{0:N2}" -f ($vol.Size / 1GB)
                }
            }
    } | Format-Table -AutoSize
