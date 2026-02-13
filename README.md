# Test_Ciber
```md
Get-Disk |
    Where-Object BusType -eq 'USB' |
    ForEach-Object {
        $disk = $_

        $serial = (Get-CimInstance Win32_PhysicalMedia |
                   Where-Object { $_.Tag -eq ("\\.\PHYSICALDRIVE" + $disk.Number) }).SerialNumber

        Get-Partition -DiskNumber $disk.Number |
            Where-Object DriveLetter -ne $null |
            ForEach-Object {

                $vol = Get-Volume -DriveLetter $_.DriveLetter -ErrorAction SilentlyContinue
                if ($null -eq $vol) { return }

                $letra = $vol.DriveLetter

                # Obtener contenido
                $contenido = Get-ChildItem "$letra`:\" -Directory | Select-Object -ExpandProperty Name
                if (-not $contenido) { $contenido = @("(Sin carpetas)") }

                # Construir líneas del cuadro
                $lineas = @(
                    "Información de la unidad USB"
                    ""
                    "Nombre     : $($vol.FileSystemLabel)"
                    "Letra      : $letra"
                    "Libre (GB) : {0:N2}" -f ($vol.SizeRemaining / 1GB)
                    "Usado (GB) : {0:N2}" -f (($vol.Size - $vol.SizeRemaining) / 1GB)
                    "Total (GB) : {0:N2}" -f ($vol.Size / 1GB)
                    "Nº Serie   : $($serial.Trim())"
                    ""
                    "Contenido:"
                )

                foreach ($c in $contenido) {
                    $lineas += " - $c"
                }

                # Calcular ancho máximo
                $max = ($lineas | Measure-Object -Maximum Length).Maximum

                # Dibujar cuadro
                Write-Host ""
                Write-Host ("╔" + ("═" * ($max + 2)) + "╗")

                foreach ($l in $lineas) {
                    Write-Host ("║ " + $l.PadRight($max) + " ║")
                }

                Write-Host ("╚" + ("═" * ($max + 2)) + "╝")
                Write-Host ""
            }
    }
