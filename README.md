# Test_Ciber
```md
$bloques = @()   # Aquí guardaremos cada bloque formateado

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

                if (-not $contenido) {
                    $contenido = @("(Sin carpetas)")
                }

                # Crear bloque formateado
                $bloque = @()
                $bloque += "Nombre     : $($vol.FileSystemLabel)"
                $bloque += "Letra      : $letra"
                $bloque += ("Libre (GB) : {0:N2}" -f ($vol.SizeRemaining / 1GB))
                $bloque += ("Usado (GB) : {0:N2}" -f (($vol.Size - $vol.SizeRemaining) / 1GB))
                $bloque += ("Total (GB) : {0:N2}" -f ($vol.Size / 1GB))
                $bloque += "Nº Serie   : $($serial.Trim())"
                $bloque += ""
                $bloque += "Contenido:"
                foreach ($c in $contenido) {
                    $bloque += " - $c"
                }

                # Guardar bloque como un solo string
                $bloques += ,($bloque -join "`n")
            }
    }

# ============================
#   MOSTRAR BLOQUES EN COLUMNAS
# ============================

# Calcular ancho máximo de columna
$ancho = ($bloques | ForEach-Object { $_.Split("`n") | Measure-Object -Maximum Length }).Maximum + 4

# Convertir cada bloque en líneas
$lineas = $bloques | ForEach-Object { $_.Split("`n") }

# Calcular número máximo de líneas
$maxLineas = ($lineas | ForEach-Object { $_.Count } | Measure-Object -Maximum).Maximum

# Imprimir en columnas
for ($i = 0; $i -lt $maxLineas; $i++) {
    $fila = ""
    foreach ($bloque in $lineas) {
        if ($i -lt $bloque.Count) {
            $fila += $bloque[$i].PadRight($ancho)
        } else {
            $fila += "".PadRight($ancho)
        }
    }
    Write-Host $fila
}
