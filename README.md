# Test_Ciber
```md
# 1) PRIMER PASO: Recoger toda la información antes de imprimir nada
$unidades = @()

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

                # Construir líneas de información
                $info = @(
                    "INFORMACIÓN DE LA UNIDAD USB"
                    "Nombre     : $($vol.FileSystemLabel)"
                    "Letra      : $letra"
                    "Libre (GB) : {0:N2}" -f ($vol.SizeRemaining / 1GB)
                    "Usado (GB) : {0:N2}" -f (($vol.Size - $vol.SizeRemaining) / 1GB)
                    "Total (GB) : {0:N2}" -f ($vol.Size / 1GB)
                    "Nº Serie   : $($serial.Trim())"
                )

                # Construir líneas de contenido
                $cont = @("CONTENIDO:")
                foreach ($c in $contenido) {
                    $cont += " - $c"
                }

                # Guardar en memoria
                $unidades += [PSCustomObject]@{
                    Info = $info
                    Cont = $cont
                }
            }
    }

# 2) SEGUNDO PASO: Calcular el ancho máximo entre TODAS las unidades
$max = 0
foreach ($u in $unidades) {
    $todas = $u.Info + $u.Cont
    $lmax = ($todas | Measure-Object -Maximum Length).Maximum
    if ($lmax -gt $max) { $max = $lmax }
}

# 3) TERCER PASO: Imprimir todos los cuadros con el MISMO ancho
foreach ($u in $unidades) {

    Write-Host ""
    Write-Host ("╔" + ("═" * ($max + 2)) + "╗")

    # TÍTULO CENTRADO
    $titulo = $u.Info[0]
    $tituloCentrado = $titulo.PadLeft((($max + $titulo.Length) / 2), " ").PadRight($max)
    Write-Host ("║ " + $tituloCentrado + " ║")

    # Separador
    Write-Host ("╠" + ("═" * ($max + 2)) + "╣")

    # Resto de la información
    foreach ($l in $u.Info[1..($u.Info.Count - 1)]) {
        Write-Host ("║ " + $l.PadRight($max) + " ║")
    }

    # Separador entre INFO y CONTENIDO
    Write-Host ("╠" + ("═" * ($max + 2)) + "╣")

    # Contenido
    foreach ($l in $u.Cont) {
        Write-Host ("║ " + $l.PadRight($max) + " ║")
    }

    Write-Host ("╚" + ("═" * ($max + 2)) + "╝")
    Write-Host ""
}
