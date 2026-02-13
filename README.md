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

                # Guardar la letra ANTES de que cambie
                $letra = $vol.DriveLetter

                # ============================
                #   BLOQUE DE INFORMACIÓN PRO
                # ============================

                Write-Host ""
                Write-Host "----------------------------------------" -ForegroundColor DarkGray
                Write-Host "   Información de la unidad USB" -ForegroundColor Yellow
                Write-Host "----------------------------------------" -ForegroundColor DarkGray

                Write-Host ("Nombre     : {0}" -f ($vol.FileSystemLabel)) -ForegroundColor White
                Write-Host ("Letra      : {0}" -f $letra) -ForegroundColor White
                Write-Host ("Libre (GB) : {0:N2}" -f ($vol.SizeRemaining / 1GB)) -ForegroundColor White
                Write-Host ("Usado (GB) : {0:N2}" -f (($vol.Size - $vol.SizeRemaining) / 1GB)) -ForegroundColor White
                Write-Host ("Total (GB) : {0:N2}" -f ($vol.Size / 1GB)) -ForegroundColor White
                Write-Host ("Nº Serie   : {0}" -f ($serial.Trim())) -ForegroundColor White

                # ============================
                #   CONTENIDO DE LA UNIDAD
                # ============================

                Write-Host ""
                Write-Host "=== Contenido de $letra`: ===" -ForegroundColor Cyan

                $contenido = Get-ChildItem "$letra`:\" -Directory | Select-Object -ExpandProperty Name

                if ($contenido) {
                    $contenido | ForEach-Object {
                        Write-Host " - $_" -ForegroundColor Green
                    }
                }
                else {
                    Write-Host " (Sin carpetas)" -ForegroundColor DarkYellow
                }

                Write-Host ""
            }
    }
