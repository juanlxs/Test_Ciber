# Test_Ciber
```md
$LetrasUSB = @()

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

                # Guardar letra
                $LetrasUSB += $vol.DriveLetter

                # --- MOSTRAR INFO DE LA UNIDAD (sin Format-Table) ---
                $info = [PSCustomObject]@{
                    Nombre        = if ($vol.FileSystemLabel) { $vol.FileSystemLabel } else { "(Sin nombre)" }
                    Letra         = $vol.DriveLetter
                    'Libre (GB)'  = "{0:N2}" -f ($vol.SizeRemaining / 1GB)
                    'Usado (GB)'  = "{0:N2}" -f (($vol.Size - $vol.SizeRemaining) / 1GB)
                    'Total (GB)'  = "{0:N2}" -f ($vol.Size / 1GB)
                    'Nº Serie'    = if ($serial) { $serial.Trim() } else { "(No disponible)" }
                }

                # Convertir a texto para que se imprima YA
                $info | Out-String | Write-Host

                # --- MOSTRAR CONTENIDO DE LA UNIDAD ---
                Write-Host "=== Contenido de $($vol.DriveLetter): ===" -ForegroundColor Cyan
                Get-ChildItem "$($vol.DriveLetter):\" -Directory | Select-Object Name
                Write-Host ""
            }
    }
