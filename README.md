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

                $letra = $vol.DriveLetter

                # Obtener contenido de la unidad
                $contenido = Get-ChildItem "$letra`:\" -Directory | Select-Object -ExpandProperty Name
                if (-not $contenido) { $contenido = @("(Sin carpetas)") }

                # ============================
                #   CUADRO UNICODE PRO
                # ============================

                Write-Host ""
                Write-Host "╔══════════════════════════════════════╗"
                Write-Host "║   Información de la unidad USB       ║"
                Write-Host "╠══════════════════════════════════════╣"

                Write-Host ("║ Nombre     : {0,-24}║" -f $vol.FileSystemLabel)
                Write-Host ("║ Letra      : {0,-24}║" -f $letra)
                Write-Host ("║ Libre (GB) : {0,-24}║" -f ("{0:N2}" -f ($vol.SizeRemaining / 1GB)))
                Write-Host ("║ Usado (GB) : {0,-24}║" -f ("{0:N2}" -f (($vol.Size - $vol.SizeRemaining) / 1GB)))
                Write-Host ("║ Total (GB) : {0,-24}║" -f ("{0:N2}" -f ($vol.Size / 1GB)))
                Write-Host ("║ Nº Serie   : {0,-24}║" -f $serial.Trim())

                Write-Host "╠══════════════════════════════════════╣"
                Write-Host ("║ Contenido:{0, -27}║" -f "")

                foreach ($c in $contenido) {
                    Write-Host ("║  - {0,-28}║" -f $c)
                }

                Write-Host "╚══════════════════════════════════════╝"
                Write-Host ""
            }
    }
