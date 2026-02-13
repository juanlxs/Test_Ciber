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
$hashOrigen = $hashOrigen -replace "`r`n", "`n"   # Normaliza saltos
$hashOrigen = $hashOrigen.Trim()

# ================================
#   HASH DESTINO (SALIDA COMPLETA)
# ================================
Write-Host "Calculando hash de DESTINO..."
$hashDestino = 7z h -scrcSHA256 $destino | Out-String
$hashDestino = $hashDestino -replace "`r`n", "`n" # Normaliza saltos
$hashDestino = $hashDestino.Trim()

# ================================
#   CUADROS UNICODE (solo encabezado)
# ================================
$cuadroOrigen = @"
╔══════════════════════════════════════════════╗
║                HASH ORIGEN                   ║
╚══════════════════════════════════════════════╝
"@

$cuadroDestino = @"
╔══════════════════════════════════════════════╗
║                HASH DESTINO                  ║
╚══════════════════════════════════════════════╝
"@

# ================================
#   GUARDAR EN TXT
# ================================
@"
$cuadroOrigen
$hashOrigen

$cuadroDestino
$hashDestino
"@ | Out-File -FilePath $salida -Encoding UTF8
