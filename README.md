# Test_Ciber
```md
Get-Disk |
    Where-Object BusType -eq 'USB' |
    ForEach-Object {
        $disk = $_

        # Obtener número de serie físico del USB
        $serial = (Get-CimInstance Win32_PhysicalMedia |
                   Where-Object { $_.Tag -eq ("\\.\PHYSICALDRIVE" + $disk.Number) }).SerialNumber

        Get-Partition -DiskNumber $disk.Number |
            Where-Object DriveLetter -ne $null |
            ForEach-Object {

                $vol = Get-Volume -DriveLetter $_.DriveLetter -ErrorAction SilentlyContinue
                if ($null -eq $vol) { return }

                # GUARDAR LA LETRA ANTES DE QUE CAMBIE
                $letra = $vol.DriveLetter

                # Mostrar información de la unidad
                [PSCustomObject]@{
                    Nombre        = if ($vol.FileSystemLabel) { $vol.FileSystemLabel } else { "(Sin nombre)" }
                    Letra         = $letra
                    'Libre (GB)'  = "{0:N2}" -f ($vol.SizeRemaining / 1GB)
                    'Usado (GB)'  = "{0:N2}" -f (($vol.Size - $vol.SizeRemaining) / 1GB)
                    'Total (GB)'  = "{0:N2}" -f ($vol.Size / 1GB)
                    'Nº Serie'    = if ($serial) { $serial.Trim() } else { "(No disponible)" }
                } | Format-Table -AutoSize

                # Mostrar contenido de la unidad CORRECTA
                Write-Host "`n=== Contenido de $letra`: ===" -ForegroundColor Cyan
                Get-ChildItem "$letra`:\" -Directory | Select-Object Name
                Write-Host ""
            }
    }
