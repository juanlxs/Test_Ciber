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

                # Obtener contenido
                $contenido = Get-ChildItem "$letra`:\" -Directory | Select-Object -ExpandProperty Name
                if (-not $contenido) { $contenido = @("(Sin carpetas)") }

                # Construir líneas de información
                $info = @(
                    "Información de la unidad USB"
                    "Nombre     : $($vol.FileSystemLabel)"
                    "Letra      : $letra"
                    "Libre (GB) : {0:N2}" -f ($vol.SizeRemaining / 1GB)
                    "Usado (GB) : {0:N2}" -f (($vol.Size - $vol.SizeRemaining) / 1GB)
                    "Total (GB) : {0:N2}" -f ($vol.Size / 1GB)
                    "Nº Serie   : $($serial.Trim())"
                )

                # Construir líneas de contenido
                $cont = @("Contenido:")
                foreach ($c in $contenido) {
                    $cont += " - $c"
                }

                # Unir todo para calcular ancho
                $todas = $info + $cont
                $max = ($todas | Measure-Object -Maximum Length).Maximum

                # Dibujar cuadro
                Write-Host ""
                Write-Host ("╔" + ("═" * ($max + 2)) + "╗")

                # Bloque de información
                foreach ($l in $info) {
                    Write-Host ("║ " + $l.PadRight($max) + " ║")
                }

                # Separador
                Write-Host ("╠" + ("═" * ($max + 2)) + "╣")

                # Bloque de contenido
                foreach ($l in $cont) {
                    Write-Host ("║ " + $l.PadRight($max) + " ║")
                }

                Write-Host ("╚" + ("═" * ($max + 2)) + "╝")
                Write-Host ""
            }
    }
