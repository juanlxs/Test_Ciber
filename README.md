# Test_Ciber
```md
$ancho = 60  # ancho fijo para cada línea dentro del cuadro

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

                # Función para ajustar texto al ancho
                function Ajustar([string]$texto) {
                    if ($texto.Length -ge $ancho) {
                        return $texto.Substring(0, $ancho)
                    } else {
                        return $texto.PadRight($ancho)
                    }
                }

                # ============================
                #   CUADRO UNICODE PRO
                # ============================

                Write-Host ""
                Write-Host "╔" + ("═" * ($ancho + 2)) + "╗"
                Write-Host ("║ " + (Ajustar "Información de la unidad USB") + " ║")
                Write-Host "╠" + ("═" * ($ancho + 2)) + "╣"

                Write-Host ("║ " + (Ajustar ("Nombre     : $($vol.FileSystemLabel)")) + " ║")
                Write-Host ("║ " + (Ajustar ("Letra      : $letra")) + " ║")
                Write-Host ("║ " + (Ajustar ("Libre (GB) : {0:N2}" -f ($vol.SizeRemaining / 1GB))) + " ║")
                Write-Host ("║ " + (Ajustar ("Usado (GB) : {0:N2}" -f (($vol.Size - $vol.SizeRemaining) / 1GB))) + " ║")
                Write-Host ("║ " + (Ajustar ("Total (GB) : {0:N2}" -f ($vol.Size / 1GB))) + " ║")
                Write-Host ("║ " + (Ajustar ("Nº Serie   : $($serial.Trim())")) + " ║")

                Write-Host "╠" + ("═" * ($ancho + 2)) + "╣"
                Write-Host ("║ " + (Ajustar "Contenido:") + " ║")

                foreach ($c in $contenido) {
                    Write-Host ("║ " + (Ajustar (" - $c")) + " ║")
                }

                Write-Host "╚" + ("═" * ($ancho + 2)) + "╝"
                Write-Host ""
            }
    }
