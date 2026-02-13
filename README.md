# Test_Ciber
```md
# ================================
#   RUTAS
# ================================
$origen  = "C:\Origen"
$destino = "C:\Destino"
$salida  = "C:\resultado_hash.txt"

# ================================
#   COPIA DE CARPETA
# ================================
Write-Host "Copiando carpeta..."
Copy-Item -Path $origen -Destination $destino -Recurse -Force
Write-Host "Copia finalizada."
Write-Host ""

# ================================
#   HASH ORIGEN (SALIDA COMPLETA)
# ================================
Write-Host "Calculando hash de ORIGEN..."
$hashOrigen = 7z h -scrcSHA256 $origen | Out-String

# ================================
#   HASH DESTINO (SALIDA COMPLETA)
# ================================
Write-Host "Calculando hash de DESTINO..."
$hashDestino = 7z h -scrcSHA256 $destino | Out-String

# ================================
#   CREAR CUADROS UNICODE
# ================================
function Crear-Cuadro {
    param(
        [string]$titulo,
        [string]$contenido
    )

    $lineas = $contenido -split "`n"
    $max = ($lineas | Measure-Object -Maximum Length).Maximum
    if ($max -lt $titulo.Length) { $max = $titulo.Length }

    $cuadro = @()
    $cuadro += "╔" + ("═" * ($max + 2)) + "╗"
    $cuadro += "║ " + $titulo.PadRight($max) + " ║"
    $cuadro += "╠" + ("═" * ($max + 2)) + "╣"

    foreach ($l in $lineas) {
        $cuadro += "║ " + $l.PadRight($max) + " ║"
    }

    $cuadro += "╚" + ("═" * ($max + 2)) + "╝"
    return $cuadro -join "`n"
}

$cuadroOrigen  = Crear-Cuadro -titulo "HASH ORIGEN"  -contenido $hashOrigen
$cuadroDestino = Crear-Cuadro -titulo "HASH DESTINO" -contenido $hashDestino

# ================================
#   GUARDAR EN TXT
# ================================
@"
$cuadroOrigen

$cuadroDestino
"@ | Out-File -FilePath $salida -Encoding UTF8

Write-Host ""
Write-Host "Hashes generados y guardados en:"
Write-Host $salida
