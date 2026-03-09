# Test_Ciber
## CONVERSION
```md
param(
    [Parameter(Mandatory = $true)]
    [string]$Path,   # Ruta de la imagen Clonezilla

    [string]$OutputBaseFolder   = "C:\imagen\conversion",
    [string]$SevenZipPath       = "C:\Program Files\7-Zip\7z.exe",
    [string]$QemuImgPath        = "C:\Program Files\qemu\qemu-img.exe",
    [string]$ClonezillaUtilPath = "C:\ruta\a\clonezilla-util.exe"   # <-- CAMBIA ESTA RUTA
)

# ============================
#   RUTAS DE SALIDA
# ============================
$ClonezillaFolder = $Path
$ImgFolder  = Join-Path $OutputBaseFolder "img"
$VhdxFolder = Join-Path $OutputBaseFolder "vhdx"
$LogFile    = Join-Path $OutputBaseFolder "conversion.log"

New-Item -ItemType Directory -Path $OutputBaseFolder -Force | Out-Null
New-Item -ItemType Directory -Path $ImgFolder        -Force | Out-Null
New-Item -ItemType Directory -Path $VhdxFolder       -Force | Out-Null

# ============================
#   FUNCIONES DE LOG
# ============================
function Write-Log {
    param([string]$Message, [ConsoleColor]$Color = [ConsoleColor]::Gray)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $line = "[$timestamp] $Message"
    $oldColor = $Host.UI.RawUI.ForegroundColor
    $Host.UI.RawUI.ForegroundColor = $Color
    Write-Host $line
    $Host.UI.RawUI.ForegroundColor = $oldColor
    Add-Content -Path $LogFile -Value $line
}

function Format-SizeMB {
    param([long]$Bytes)
    if ($Bytes -le 0) { return "0 MB" }
    return ("{0:N0} MB" -f ($Bytes / 1MB))
}

$globalStopwatch = [System.Diagnostics.Stopwatch]::StartNew()

Write-Log "===== INICIO DEL PROCESO =====" "Cyan"
Write-Log "Carpeta Clonezilla origen: $ClonezillaFolder"
Write-Log "Carpeta de salida: $OutputBaseFolder"
Write-Log "-----------------------------------------------"

# ============================
# 1. HASH SHA-256 DE LA CARPETA CON 7-ZIP
# ============================
$sw1 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 1] HASH SHA-256 de la carpeta Clonezilla (7-Zip)..." "Yellow"

$hashCmd = @("h", "-scrcSHA256", "`"$ClonezillaFolder`"")

try {
    $hashOutput = & "$SevenZipPath" $hashCmd 2>&1
    $hashOutput | ForEach-Object { Write-Log "7z: $_" }
}
catch {
    Write-Log "ERROR en 7-Zip: $($_.Exception.Message)" "Red"
}

$sw1.Stop()
Write-Log ("Tiempo paso 1: {0:N1} segundos" -f $sw1.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 2. CONVERTIR CLONEZILLA → IMG
# ============================
$sw2 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 2] Conversión Clonezilla → IMG (clonezilla-util.exe)..." "Yellow"

$ImageDirs = Get-ChildItem -Path $ClonezillaFolder -Directory

foreach ($dir in $ImageDirs) {
    Write-Log "Procesando imagen Clonezilla: $($dir.FullName)"

    $cmd = "`"$ClonezillaUtilPath`" extract-partition-image --input `"$($dir.FullName)`" --output `"$ImgFolder`""
    Write-Log "Ejecutando: $cmd"

    try {
        $output = Invoke-Expression $cmd 2>&1
        $output | ForEach-Object { Write-Log "clonezilla-util: $_" }
    }
    catch {
        Write-Log "ERROR en clonezilla-util: $($_.Exception.Message)" "Red"
    }
}

$imgFilesAfter = Get-ChildItem -Path $ImgFolder -File -Filter *.img
foreach ($img in $imgFilesAfter) {
    Write-Log "IMG generado: $($img.Name) (Tamaño: $(Format-SizeMB $img.Length))"
}

$sw2.Stop()
Write-Log ("Tiempo paso 2: {0:N1} segundos" -f $sw2.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 3. HASHES ORIGINAL → IMG
# ============================
$sw3 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 3] Hashes SHA-256 y comparación ORIGINAL → IMG..." "Yellow"

$originalPartitionFiles = Get-ChildItem -Path $ClonezillaFolder -Recurse -File |
                          Where-Object { $_.Name -match "\.img(\.gz|\.gzip|\.lz4)?$" }

foreach ($orig in $originalPartitionFiles) {
    $baseName = $orig.Name -replace "\.gz|\.gzip|\.lz4",""
    $imgPath  = Join-Path $ImgFolder $baseName

    $hashOrig = (Get-FileHash -Algorithm SHA256 -Path $orig.FullName).Hash
    Write-Log "HASH ORIGINAL: $($orig.FullName) = $hashOrig"

    if (Test-Path $imgPath) {
        $hashImg = (Get-FileHash -Algorithm SHA256 -Path $imgPath).Hash
        Write-Log "HASH IMG:      $imgPath = $hashImg"

        Write-Log "Integridad ORIGINAL→IMG: HASH DISTINTO (esperable por compresión)" "DarkYellow"
    }
    else {
        Write-Log "AVISO: No se encontró IMG correspondiente para $($orig.FullName)" "DarkYellow"
    }
}

$sw3.Stop()
Write-Log ("Tiempo paso 3: {0:N1} segundos" -f $sw3.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 4. CONVERTIR IMG → VHDX
# ============================
$sw4 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 4] Conversión IMG → VHDX (qemu-img)..." "Yellow"

$imgFiles = Get-ChildItem -Path $ImgFolder -File -Filter *.img

foreach ($img in $imgFiles) {
    $vhdxName = ($img.BaseName + ".vhdx")
    $vhdxPath = Join-Path $VhdxFolder $vhdxName

    Write-Log "Convirtiendo: $($img.FullName) → $vhdxPath"

    try {
        & "$QemuImgPath" convert -f raw -O vhdx "`"$($img.FullName)`"" "`"$vhdxPath`"" 2>&1 |
            ForEach-Object { Write-Log "QEMU: $_" }
    }
    catch {
        Write-Log "ERROR en qemu-img: $($_.Exception.Message)" "Red"
    }

    if (Test-Path $vhdxPath) {
        Write-Log "VHDX generado: $vhdxPath (Tamaño: $(Format-SizeMB ((Get-Item $vhdxPath).Length)))"
    }
}

$sw4.Stop()
Write-Log ("Tiempo paso 4: {0:N1} segundos" -f $sw4.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 5. HASHES DE LOS VHDX
# ============================
$sw5 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 5] Hashes SHA-256 de los VHDX..." "Yellow"

$vhdxFiles = Get-ChildItem -Path $VhdxFolder -File -Filter *.vhdx
foreach ($vhdx in $vhdxFiles) {
    $hashVhdx = (Get-FileHash -Algorithm SHA256 -Path $vhdx.FullName).Hash
    Write-Log "HASH VHDX: $($vhdx.FullName) = $hashVhdx"
}

$sw5.Stop()
Write-Log ("Tiempo paso 5: {0:N1} segundos" -f $sw5.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# RESUMEN FINAL
# ============================
$globalStopwatch.Stop()

$partCount   = $imgFiles.Count
$totalImgMB  = ($imgFiles | Measure-Object -Property Length -Sum).Sum
$totalVhdxMB = ($vhdxFiles | Measure-Object -Property Length -Sum).Sum

Write-Log "========== RESUMEN FINAL ==========" "Cyan"
Write-Log "Particiones procesadas: $partCount"
Write-Log "Tamaño total IMG:  $(Format-SizeMB $totalImgMB)"
Write-Log "Tamaño total VHDX: $(Format-SizeMB $totalVhdxMB)"
Write-Log ("Tiempo total: {0:N1} segundos" -f $globalStopwatch.Elapsed.TotalSeconds)
Write-Log "Estado final: COMPLETADO (revisar avisos/errores si los hay)"
Write-Log "==================================" "Cyan"

```
## CONVERSION CON BARRAS DE PROGRESO

```md
param(
    [Parameter(Mandatory = $true)]
    [string]$Path,   # Ruta de la imagen Clonezilla

    [string]$OutputBaseFolder   = "C:\imagen\conversion",
    [string]$SevenZipPath       = "C:\Program Files\7-Zip\7z.exe",
    [string]$QemuImgPath        = "C:\Program Files\qemu\qemu-img.exe",
    [string]$ClonezillaUtilPath = "C:\ruta\a\clonezilla-util.exe"
)

# ============================
#   RUTAS DE SALIDA
# ============================
$ClonezillaFolder = $Path
$ImgFolder  = Join-Path $OutputBaseFolder "img"
$VhdxFolder = Join-Path $OutputBaseFolder "vhdx"
$LogFile    = Join-Path $OutputBaseFolder "conversion.log"

New-Item -ItemType Directory -Path $OutputBaseFolder -Force | Out-Null
New-Item -ItemType Directory -Path $ImgFolder        -Force | Out-Null
New-Item -ItemType Directory -Path $VhdxFolder       -Force | Out-Null

# ============================
#   LOGGING
# ============================
function Write-Log {
    param([string]$Message, [ConsoleColor]$Color = [ConsoleColor]::Gray)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $line = "[$timestamp] $Message"
    $oldColor = $Host.UI.RawUI.ForegroundColor
    $Host.UI.RawUI.ForegroundColor = $Color
    Write-Host $line
    $Host.UI.RawUI.ForegroundColor = $oldColor
    Add-Content -Path $LogFile -Value $line
}

function Format-SizeMB {
    param([long]$Bytes)
    if ($Bytes -le 0) { return "0 MB" }
    return ("{0:N0} MB" -f ($Bytes / 1MB))
}

$globalStopwatch = [System.Diagnostics.Stopwatch]::StartNew()

Write-Log "===== INICIO DEL PROCESO =====" "Cyan"
Write-Log "Carpeta Clonezilla origen: $ClonezillaFolder"
Write-Log "Carpeta de salida: $OutputBaseFolder"
Write-Log "-----------------------------------------------"

# ============================
# 1. HASH SHA-256 DE LA CARPETA CON 7-ZIP
# ============================
$sw1 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 1] HASH SHA-256 de la carpeta Clonezilla (7-Zip)..." "Yellow"

Write-Progress -Activity "Paso 1/5 - Hash SHA-256 (7-Zip)" -Status "Ejecutando 7-Zip..." -PercentComplete 10

try {
    $hashOutput = & "$SevenZipPath" h -scrcSHA256 "`"$ClonezillaFolder`"" 2>&1
    $hashOutput | ForEach-Object { Write-Log "7z: $_" }
}
catch {
    Write-Log "ERROR en 7-Zip: $($_.Exception.Message)" "Red"
}

Write-Progress -Activity "Paso 1/5 - Hash SHA-256 (7-Zip)" -Completed

$sw1.Stop()
Write-Log ("Tiempo paso 1: {0:N1} segundos" -f $sw1.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 2. CONVERTIR CLONEZILLA → IMG (CON PROGRESO)
# ============================
$sw2 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 2] Conversión Clonezilla → IMG (clonezilla-util.exe)..." "Yellow"

$ImageDirs = Get-ChildItem -Path $ClonezillaFolder -Directory
$totalDirs = $ImageDirs.Count
$processedDirs = 0

foreach ($dir in $ImageDirs) {
    $processedDirs++
    $percent = [math]::Min(100, ($processedDirs / [math]::Max(1,$totalDirs)) * 100)

    Write-Progress `
        -Activity "Paso 2/5 - Clonezilla → IMG" `
        -Status "Procesando imagen: $($dir.Name)" `
        -PercentComplete $percent

    Write-Log "Procesando imagen Clonezilla: $($dir.FullName)"

    $cmd = "`"$ClonezillaUtilPath`" extract-partition-image --input `"$($dir.FullName)`" --output `"$ImgFolder`""
    Write-Log "Ejecutando: $cmd"

    try {
        $output = Invoke-Expression $cmd 2>&1
        $output | ForEach-Object { Write-Log "clonezilla-util: $_" }
    }
    catch {
        Write-Log "ERROR en clonezilla-util: $($_.Exception.Message)" "Red"
    }
}

Write-Progress -Activity "Paso 2/5 - Clonezilla → IMG" -Completed

$imgFilesAfter = Get-ChildItem -Path $ImgFolder -File -Filter *.img
foreach ($img in $imgFilesAfter) {
    Write-Log "IMG generado: $($img.Name) (Tamaño: $(Format-SizeMB $img.Length))"
}

$sw2.Stop()
Write-Log ("Tiempo paso 2: {0:N1} segundos" -f $sw2.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 3. HASHES ORIGINAL → IMG (CON PROGRESO)
# ============================
$sw3 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 3] Hashes SHA-256 y comparación ORIGINAL → IMG..." "Yellow"

$originalPartitionFiles = Get-ChildItem -Path $ClonezillaFolder -Recurse -File |
                          Where-Object { $_.Name -match "\.img(\.gz|\.gzip|\.lz4)?$" }

$totalOrig = $originalPartitionFiles.Count
$processedOrig = 0

foreach ($orig in $originalPartitionFiles) {
    $processedOrig++
    $percent = [math]::Min(100, ($processedOrig / [math]::Max(1,$totalOrig)) * 100)

    Write-Progress `
        -Activity "Paso 3/5 - Comparando ORIGINAL → IMG" `
        -Status "Archivo: $($orig.Name)" `
        -PercentComplete $percent

    $baseName = $orig.Name -replace "\.gz|\.gzip|\.lz4",""
    $imgPath  = Join-Path $ImgFolder $baseName

    $hashOrig = (Get-FileHash -Algorithm SHA256 -Path $orig.FullName).Hash
    Write-Log "HASH ORIGINAL: $($orig.FullName) = $hashOrig"

    if (Test-Path $imgPath) {
        $hashImg = (Get-FileHash -Algorithm SHA256 -Path $imgPath).Hash
        Write-Log "HASH IMG:      $imgPath = $hashImg"
        Write-Log "Integridad ORIGINAL→IMG: HASH DISTINTO (esperable por compresión)" "DarkYellow"
    }
    else {
        Write-Log "AVISO: No se encontró IMG correspondiente para $($orig.FullName)" "DarkYellow"
    }
}

Write-Progress -Activity "Paso 3/5 - Comparando ORIGINAL → IMG" -Completed

$sw3.Stop()
Write-Log ("Tiempo paso 3: {0:N1} segundos" -f $sw3.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 4. CONVERTIR IMG → VHDX (CON PROGRESO)
# ============================
$sw4 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 4] Conversión IMG → VHDX (qemu-img)..." "Yellow"

$imgFiles = Get-ChildItem -Path $ImgFolder -File -Filter *.img
$totalImg = $imgFiles.Count
$processedImg = 0

foreach ($img in $imgFiles) {
    $processedImg++
    $percent = [math]::Min(100, ($processedImg / [math]::Max(1,$totalImg)) * 100)

    Write-Progress `
        -Activity "Paso 4/5 - IMG → VHDX" `
        -Status "Convirtiendo: $($img.Name)" `
        -PercentComplete $percent

    $vhdxName = ($img.BaseName + ".vhdx")
    $vhdxPath = Join-Path $VhdxFolder $vhdxName

    Write-Log "Convirtiendo: $($img.FullName) → $vhdxPath"

    try {
        & "$QemuImgPath" convert -f raw -O vhdx "`"$($img.FullName)`"" "`"$vhdxPath`"" 2>&1 |
            ForEach-Object { Write-Log "QEMU: $_" }
    }
    catch {
        Write-Log "ERROR en qemu-img: $($_.Exception.Message)" "Red"
    }

    if (Test-Path $vhdxPath) {
        Write-Log "VHDX generado: $vhdxPath (Tamaño: $(Format-SizeMB ((Get-Item $vhdxPath).Length)))"
    }
}

Write-Progress -Activity "Paso 4/5 - IMG → VHDX" -Completed

$sw4.Stop()
Write-Log ("Tiempo paso 4: {0:N1} segundos" -f $sw4.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 5. HASHES DE LOS VHDX (CON PROGRESO)
# ============================
$sw5 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 5] Hashes SHA-256 de los VHDX..." "Yellow"

$vhdxFiles = Get-ChildItem -Path $VhdxFolder -File -Filter *.vhdx
$totalVhdx = $vhdxFiles.Count
$processedVhdx = 0

foreach ($vhdx in $vhdxFiles) {
    $processedVhdx++
    $percent = [math]::Min(100, ($processedVhdx / [math]::Max(1,$totalVhdx)) * 100)

    Write-Progress `
        -Activity "Paso 5/5 - Hashes VHDX" `
        -Status "Archivo: $($vhdx.Name)" `
        -PercentComplete $percent

    $hashVhdx = (Get-FileHash -Algorithm SHA256 -Path $vhdx.FullName).Hash
    Write-Log "HASH VHDX: $($vhdx.FullName) = $hashVhdx"
}

Write-Progress -Activity "Paso 5/5 - Hashes VHDX" -Completed

$sw5.Stop()
Write-Log ("Tiempo paso 5: {0:N1} segundos" -f $sw5.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# RESUMEN FINAL
# ============================
$globalStopwatch.Stop()

$partCount   = $imgFiles.Count
$totalImgMB  = ($imgFiles | Measure-Object -Property Length -Sum).Sum
$totalVhdxMB = ($vhdxFiles | Measure-Object -Property Length -Sum).Sum

Write-Log "========== RESUMEN FINAL ==========" "Cyan"
Write-Log "Particiones procesadas: $partCount"
Write-Log "Tamaño total IMG:  $(Format-SizeMB $totalImgMB)"
Write-Log "Tamaño total VHDX: $(Format-SizeMB $totalVhdxMB)"
Write-Log ("Tiempo total: {0:N1} segundos" -f $globalStopwatch.Elapsed.TotalSeconds)
Write-Log "Estado final: COMPLETADO (revisar avisos/errores si los hay)"
Write-Log "==================================" "Cyan"

```

## CONVERSION CON ESTILO COROPORATIVO

```md
param(
    [Parameter(Mandatory = $true)]
    [string]$Path,   # Ruta de la imagen Clonezilla

    [string]$OutputBaseFolder   = "C:\imagen\conversion",
    [string]$SevenZipPath       = "C:\Program Files\7-Zip\7z.exe",
    [string]$QemuImgPath        = "C:\Program Files\qemu\qemu-img.exe",
    [string]$ClonezillaUtilPath = "C:\ruta\a\clonezilla-util.exe"
)

Clear-Host
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host "                 ENTERPRISE CONVERSION SUITE           " -ForegroundColor Yellow
Write-Host "                     Clonezilla → VHDX                 " -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host ""

# ============================
#   RUTAS DE SALIDA
# ============================
$ClonezillaFolder = $Path
$ImgFolder  = Join-Path $OutputBaseFolder "img"
$VhdxFolder = Join-Path $OutputBaseFolder "vhdx"
$LogFile    = Join-Path $OutputBaseFolder "conversion.log"

New-Item -ItemType Directory -Path $OutputBaseFolder -Force | Out-Null
New-Item -ItemType Directory -Path $ImgFolder        -Force | Out-Null
New-Item -ItemType Directory -Path $VhdxFolder       -Force | Out-Null

# ============================
#   LOGGING
# ============================
function Write-Log {
    param([string]$Message, [ConsoleColor]$Color = [ConsoleColor]::Gray)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $line = "[$timestamp] $Message"
    $oldColor = $Host.UI.RawUI.ForegroundColor
    $Host.UI.RawUI.ForegroundColor = $Color
    Write-Host $line
    $Host.UI.RawUI.ForegroundColor = $oldColor
    Add-Content -Path $LogFile -Value $line
}

function Format-SizeMB {
    param([long]$Bytes)
    if ($Bytes -le 0) { return "0 MB" }
    return ("{0:N0} MB" -f ($Bytes / 1MB))
}

$globalStopwatch = [System.Diagnostics.Stopwatch]::StartNew()

Write-Log "===== INICIO DEL PROCESO =====" "Cyan"
Write-Log "Carpeta Clonezilla origen: $ClonezillaFolder"
Write-Log "Carpeta de salida: $OutputBaseFolder"
Write-Log "-----------------------------------------------"

# =====================================================
# 1. HASH SHA-256 DE LA CARPETA CON 7-ZIP
# =====================================================
Write-Host ""
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host "                 PASO 1 — HASH SHA‑256                 " -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

$sw1 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 1] HASH SHA-256 de la carpeta Clonezilla (7-Zip)..." "Yellow"

Write-Progress -Activity "Paso 1/5 - Hash SHA-256 (7-Zip)" -Status "Ejecutando 7-Zip..." -PercentComplete 10

try {
    $hashOutput = & "$SevenZipPath" h -scrcSHA256 "`"$ClonezillaFolder`"" 2>&1
    $hashOutput | ForEach-Object { Write-Log "7z: $_" }
}
catch {
    Write-Log "ERROR en 7-Zip: $($_.Exception.Message)" "Red"
}

Write-Progress -Activity "Paso 1/5 - Hash SHA-256 (7-Zip)" -Completed

$sw1.Stop()
Write-Log ("Tiempo paso 1: {0:N1} segundos" -f $sw1.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# =====================================================
# 2. CONVERTIR CLONEZILLA → IMG
# =====================================================
Write-Host ""
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host "           PASO 2 — Clonezilla → IMG (RAW)             " -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

$sw2 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 2] Conversión Clonezilla → IMG..." "Yellow"

$ImageDirs = Get-ChildItem -Path $ClonezillaFolder -Directory
$totalDirs = $ImageDirs.Count
$processedDirs = 0

foreach ($dir in $ImageDirs) {
    $processedDirs++
    $percent = ($processedDirs / $totalDirs) * 100

    Write-Progress -Activity "Paso 2/5 - Clonezilla → IMG" -Status "Procesando: $($dir.Name)" -PercentComplete $percent

    Write-Log "Procesando imagen Clonezilla: $($dir.FullName)"

    $cmd = "`"$ClonezillaUtilPath`" extract-partition-image --input `"$($dir.FullName)`" --output `"$ImgFolder`""
    Write-Log "Ejecutando: $cmd"

    try {
        $output = Invoke-Expression $cmd 2>&1
        $output | ForEach-Object { Write-Log "clonezilla-util: $_" }
    }
    catch {
        Write-Log "ERROR en clonezilla-util: $($_.Exception.Message)" "Red"
    }
}

Write-Progress -Activity "Paso 2/5 - Clonezilla → IMG" -Completed

$imgFilesAfter = Get-ChildItem -Path $ImgFolder -File -Filter *.img
foreach ($img in $imgFilesAfter) {
    Write-Log "IMG generado: $($img.Name) (Tamaño: $(Format-SizeMB $img.Length))"
}

$sw2.Stop()
Write-Log ("Tiempo paso 2: {0:N1} segundos" -f $sw2.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# =====================================================
# 3. HASHES ORIGINAL → IMG
# =====================================================
Write-Host ""
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host "         PASO 3 — Comparación ORIGINAL → IMG           " -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

$sw3 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 3] Comparación de hashes ORIGINAL → IMG..." "Yellow"

$originalPartitionFiles = Get-ChildItem -Path $ClonezillaFolder -Recurse -File |
                          Where-Object { $_.Name -match "\.img(\.gz|\.gzip|\.lz4)?$" }

$totalOrig = $originalPartitionFiles.Count
$processedOrig = 0

foreach ($orig in $originalPartitionFiles) {
    $processedOrig++
    $percent = ($processedOrig / $totalOrig) * 100

    Write-Progress -Activity "Paso 3/5 - Comparando ORIGINAL → IMG" -Status "Archivo: $($orig.Name)" -PercentComplete $percent

    $baseName = $orig.Name -replace "\.gz|\.gzip|\.lz4",""
    $imgPath  = Join-Path $ImgFolder $baseName

    $hashOrig = (Get-FileHash -Algorithm SHA256 -Path $orig.FullName).Hash
    Write-Log "HASH ORIGINAL: $($orig.FullName) = $hashOrig"

    if (Test-Path $imgPath) {
        $hashImg = (Get-FileHash -Algorithm SHA256 -Path $imgPath).Hash
        Write-Log "HASH IMG:      $imgPath = $hashImg"
        Write-Log "Integridad ORIGINAL→IMG: HASH DISTINTO (esperable por compresión)" "DarkYellow"
    }
    else {
        Write-Log "AVISO: No se encontró IMG correspondiente para $($orig.FullName)" "DarkYellow"
    }
}

Write-Progress -Activity "Paso 3/5 - Comparando ORIGINAL → IMG" -Completed

$sw3.Stop()
Write-Log ("Tiempo paso 3: {0:N1} segundos" -f $sw3.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# =====================================================
# 4. CONVERTIR IMG → VHDX
# =====================================================
Write-Host ""
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host "               PASO 4 — IMG → VHDX (QEMU)              " -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

$sw4 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 4] Conversión IMG → VHDX..." "Yellow"

$imgFiles = Get-ChildItem -Path $ImgFolder -File -Filter *.img
$totalImg = $imgFiles.Count
$processedImg = 0

foreach ($img in $imgFiles) {
    $processedImg++
    $percent = ($processedImg / $totalImg) * 100

    Write-Progress -Activity "Paso 4/5 - IMG → VHDX" -Status "Convirtiendo: $($img.Name)" -PercentComplete $percent

    $vhdxName = ($img.BaseName + ".vhdx")
    $vhdxPath = Join-Path $VhdxFolder $vhdxName

    Write-Log "Convirtiendo: $($img.FullName) → $vhdxPath"

    try {
        & "$QemuImgPath" convert -f raw -O vhdx "`"$($img.FullName)`"" "`"$vhdxPath`"" 2>&1 |
            ForEach-Object { Write-Log "QEMU: $_" }
    }
    catch {
        Write-Log "ERROR en qemu-img: $($_.Exception.Message)" "Red"
    }

    if (Test-Path $vhdxPath) {
        Write-Log "VHDX generado: $vhdxPath (Tamaño: $(Format-SizeMB ((Get-Item $vhdxPath).Length)))"
    }
}

Write-Progress -Activity "Paso 4/5 - IMG → VHDX" -Completed

$sw4.Stop()
Write-Log ("Tiempo paso 4: {0:N1} segundos" -f $sw4.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# =====================================================
# 5. HASHES DE LOS VHDX
# =====================================================
Write-Host ""
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host "             PASO 5 — Hashes SHA‑256 VHDX              " -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

$sw5 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 5] Hashes SHA-256 de los VHDX..." "Yellow"

$vhdxFiles = Get-ChildItem -Path $VhdxFolder -File -Filter *.vhdx
$totalVhdx = $vhdxFiles.Count
$processedVhdx = 0

foreach ($vhdx in $vhdxFiles) {
    $processedVhdx++
    $percent = ($processedVhdx / $totalVhdx) * 100

    Write-Progress -Activity "Paso 5/5 - Hashes VHDX" -Status "Archivo: $($vhdx.Name)" -PercentComplete $percent

    $hashVhdx = (Get-FileHash -Algorithm SHA256 -Path $vhdx.FullName).Hash
    Write-Log "HASH VHDX: $($vhdx.FullName) = $hashVhdx"
}

Write-Progress -Activity "Paso 5/5 - Hashes VHDX" -Completed

$sw5.Stop()
Write-Log ("Tiempo paso 5: {0:N1} segundos" -f $sw5.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# =====================================================
# RESUMEN FINAL
# =====================================================
Write-Host ""
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host "                     RESUMEN FINAL                     " -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

$globalStopwatch.Stop()

$partCount   = $imgFiles.Count
$totalImgMB  = ($imgFiles | Measure-Object -Property Length -Sum).Sum
$totalVhdxMB = ($vhdxFiles | Measure-Object -Property Length -Sum).Sum

Write-Log "========== RESUMEN FINAL ==========" "Cyan"
Write-Log "Particiones procesadas: $partCount"
Write-Log "Tamaño total IMG:  $(Format-SizeMB $totalImgMB)"
Write-Log "Tamaño total VHDX: $(Format-SizeMB $totalVhdxMB)"
Write-Log ("Tiempo total: {0:N1} segundos" -f $globalStopwatch.Elapsed.TotalSeconds)
Write-Log "Estado final: COMPLETADO"
Write-Log "==================================" "Cyan"

Write-Host "Particiones procesadas: $partCount"
Write-Host "Tamaño total IMG:  $(Format-SizeMB $totalImgMB)"
Write-Host "Tamaño total VHDX: $(Format-SizeMB $totalVhdxMB)"
Write-Host ("Tiempo total: {0:N1} segundos" -f $globalStopwatch.Elapsed.TotalSeconds)
Write-Host "Estado final: COMPLETADO" -ForegroundColor Green

Write-Host ""
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

```

## COPIA SEGURA

```md
param(
    [Parameter(Mandatory = $true)]
    [string]$Path,   # Carpeta origen

    [Parameter(Mandatory = $true)]
    [string]$DestinationName  # Ruta COMPLETA de destino
)

# ============================
#   PREPARAR RUTAS
# ============================
$SourceFolder      = $Path
$DestinationFolder = $DestinationName

# Crear carpeta destino si no existe
New-Item -ItemType Directory -Path $DestinationFolder -Force | Out-Null

# Timestamp profesional (formato original)
$TimeStamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"

# Carpeta superior al destino
$ParentFolder = Split-Path $DestinationFolder -Parent

# Carpeta Logs en la capa superior
$LogsFolder = Join-Path $ParentFolder "Logs"
New-Item -ItemType Directory -Path $LogsFolder -Force | Out-Null

# Logs con nombres cortos + timestamp
$LogFile       = Join-Path $LogsFolder "BackupLog_$TimeStamp.log"
$DetailedLog   = Join-Path $LogsFolder "Integrity_$TimeStamp.log"
$RoboCopyLog   = Join-Path $LogsFolder "Robocopy_$TimeStamp.log"

# ============================
#   FUNCIONES
# ============================
function Write-Log {
    param([string]$Message, [ConsoleColor]$Color = [ConsoleColor]::Gray)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $line = "[$timestamp] $Message"
    $oldColor = $Host.UI.RawUI.ForegroundColor
    $Host.UI.RawUI.ForegroundColor = $Color
    Write-Host $line
    $Host.UI.RawUI.ForegroundColor = $oldColor
    Add-Content -Path $LogFile -Value $line
}

function Format-SizeMB {
    param([long]$Bytes)
    if ($Bytes -le 0) { return "0 MB" }
    return ("{0:N0} MB" -f ($Bytes / 1MB))
}

function Get-FolderHash {
    param([string]$Folder)

    $files = Get-ChildItem -Path $Folder -Recurse -File
    $hashList = @()

    foreach ($file in $files) {
        $hash = Get-FileHash -Algorithm SHA256 -Path $file.FullName
        $hashList += "$($hash.Path) = $($hash.Hash)"
    }

    return $hashList
}

function Get-DiskInfoFromPath {
    param([string]$FolderPath)

    $driveLetter = $FolderPath.Substring(0,1)
    $disk = Get-CimInstance Win32_LogicalDisk | Where-Object { $_.DeviceID -eq "$driveLetter`:" }

    $physical = Get-CimInstance Win32_DiskDrive |
        Where-Object { $_.DeviceID -like "*$driveLetter*" }

    return [PSCustomObject]@{
        DriveLetter = $driveLetter
        VolumeName  = $disk.VolumeName
        FileSystem  = $disk.FileSystem
        SizeGB      = "{0:N2} GB" -f ($disk.Size / 1GB)
        FreeGB      = "{0:N2} GB" -f ($disk.FreeSpace / 1GB)
        Serial      = $physical.SerialNumber
        Model       = $physical.Model
        Interface   = $physical.InterfaceType
    }
}

$globalStopwatch = [System.Diagnostics.Stopwatch]::StartNew()

Write-Log "===== INICIO DEL PROCESO DE COPIA MIRROR =====" "Cyan"
Write-Log "Origen: $SourceFolder"
Write-Log "Destino: $DestinationFolder"
Write-Log "-----------------------------------------------"

# ============================
#   INFORMACIÓN DE LOS DISCOS
# ============================
$DiskSourceInfo = Get-DiskInfoFromPath -FolderPath $SourceFolder
$DiskDestInfo   = Get-DiskInfoFromPath -FolderPath $DestinationFolder

Write-Log "===== INFORMACIÓN DE LOS DISCOS =====" "Cyan"
Write-Log "Disco ORIGEN ($($DiskSourceInfo.DriveLetter):)"
Write-Log "  Modelo:     $($DiskSourceInfo.Model)"
Write-Log "  Serial:     $($DiskSourceInfo.Serial)"
Write-Log "  Sistema:    $($DiskSourceInfo.FileSystem)"
Write-Log "  Tamaño:     $($DiskSourceInfo.SizeGB)"
Write-Log "  Libre:      $($DiskSourceInfo.FreeGB)"
Write-Log ""
Write-Log "Disco DESTINO ($($DiskDestInfo.DriveLetter):)"
Write-Log "  Modelo:     $($DiskDestInfo.Model)"
Write-Log "  Serial:     $($DiskDestInfo.Serial)"
Write-Log "  Sistema:    $($DiskDestInfo.FileSystem)"
Write-Log "  Tamaño:     $($DiskDestInfo.SizeGB)"
Write-Log "  Libre:      $($DiskDestInfo.FreeGB)"
Write-Log "-----------------------------------------------"

# ============================
# 1. HASH ORIGEN
# ============================
$sw1 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 1] Calculando hash SHA-256 de la carpeta origen..." "Yellow"

$SourceHashes = Get-FolderHash -Folder $SourceFolder
$SourceHashes | ForEach-Object { Write-Log "ORIGEN: $_" }

$sw1.Stop()
Write-Log ("Tiempo paso 1: {0:N1} segundos" -f $sw1.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 2. COPIA MIRROR CON ROBOCOPY
# ============================
$sw2 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 2] Copiando con ROBOCOPY (modo MIRROR)..." "Yellow"

$cmd = "robocopy `"$SourceFolder`" `"$DestinationFolder`" /MIR /R:2 /W:2 /NFL /NDL /NP /LOG:`"$RoboCopyLog`""
Write-Log "Ejecutando: $cmd"

cmd.exe /c $cmd | ForEach-Object { Write-Log "ROBOCOPY: $_" }

$sw2.Stop()
Write-Log ("Tiempo paso 2: {0:N1} segundos" -f $sw2.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 3. HASH DESTINO
# ============================
$sw3 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 3] Calculando hash SHA-256 de la carpeta destino..." "Yellow"

$DestHashes = Get-FolderHash -Folder $DestinationFolder
$DestHashes | ForEach-Object { Write-Log "DESTINO: $_" }

$sw3.Stop()
Write-Log ("Tiempo paso 3: {0:N1} segundos" -f $sw3.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 3B. LOG DETALLADO PROFESIONAL
# ============================
Write-Log "[PASO 3B] Generando log detallado profesional de hashes y timestamps..." "Yellow"

@"
============================================================
                FILE HASH & METADATA REPORT
============================================================

Generated on: $(Get-Date -Format "yyyy-MM-dd HH:mm:ss")
Source Folder: $SourceFolder
Destination Folder: $DestinationFolder
------------------------------------------------------------

"@ | Out-File $DetailedLog

function Write-DetailedEntry {
    param(
        [string]$FilePath,
        [string]$HashValue
    )

    if (Test-Path $FilePath) {
        $item = Get-Item $FilePath
        $sizeBytes = $item.Length
        $sizeGB = "{0:N6}" -f ($sizeBytes / 1GB)

        $entry = @"
FILE PATH      : $FilePath
HASH (SHA256)  : $HashValue
SIZE (BYTES)   : $sizeBytes
SIZE (GB)      : $sizeGB
CREATED        : $($item.CreationTime)
MODIFIED       : $($item.LastWriteTime)
ACCESSED       : $($item.LastAccessTime)
------------------------------------------------------------

"@

        Add-Content -Path $DetailedLog -Value $entry
    }
}

Add-Content -Path $DetailedLog -Value "====================== ORIGIN FILES ========================`n"
foreach ($line in $SourceHashes) {
    $parts = $line -split " = "
    Write-DetailedEntry -FilePath $parts[0] -HashValue $parts[1]
}

Add-Content -Path $DetailedLog -Value "===================== DESTINATION FILES ====================`n"
foreach ($line in $DestHashes) {
    $parts = $line -split " = "
    Write-DetailedEntry -FilePath $parts[0] -HashValue $parts[1]
}

Add-Content -Path $DetailedLog -Value "========================= END REPORT ========================"

Write-Log "Log detallado generado: $DetailedLog" "Green"
Write-Log "-----------------------------------------------"

# ============================
# 4. COMPARACIÓN HASHES
# ============================
$sw4 = [System.Diagnostics.Stopwatch]::StartNew()
Write-Log "[PASO 4] Comparando integridad ORIGEN → DESTINO..." "Yellow"

$sourceMap = @{}
foreach ($line in $SourceHashes) {
    $parts = $line -split " = "
    $sourceMap[$parts[0].Replace($SourceFolder,"")] = $parts[1]
}

$destMap = @{}
foreach ($line in $DestHashes) {
    $parts = $line -split " = "
    $destMap[$parts[0].Replace($DestinationFolder,"")] = $parts[1]
}

$allFiles = $sourceMap.Keys

foreach ($file in $allFiles) {
    if ($destMap.ContainsKey($file)) {
        if ($sourceMap[$file] -eq $destMap[$file]) {
            Write-Log "OK: $file → HASH IGUAL" "Green"
        }
        else {
            Write-Log "ERROR: $file → HASH DIFERENTE" "Red"
        }
    }
    else {
        Write-Log "FALTA EN DESTINO: $file" "Red"
    }
}

$sw4.Stop()
Write-Log ("Tiempo paso 4: {0:N1} segundos" -f $sw4.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# RESUMEN FINAL
# ============================
$globalStopwatch.Stop()

$srcSize = (Get-ChildItem -Path $SourceFolder -Recurse -File | Measure-Object -Property Length -Sum).Sum
$dstSize = (Get-ChildItem -Path $DestinationFolder -Recurse -File | Measure-Object -Property Length -Sum).Sum

Write-Log "========== RESUMEN FINAL ==========" "Cyan"
Write-Log "Carpeta origen: $SourceFolder"
Write-Log "Carpeta destino: $DestinationFolder"
Write-Log "Tamaño origen:  $(Format-SizeMB $srcSize)"
Write-Log "Tamaño destino: $(Format-SizeMB $dstSize)"
Write-Log ("Tiempo total: {0:N1} segundos" -f $globalStopwatch.Elapsed.TotalSeconds)
Write-Log "Estado final: COMPLETADO SIN ERRORES"
Write-Log "==================================" "Cyan"

```


## Copia segura rapida

```md
param(
    [Parameter(Mandatory = $true)]
    [string]$Path,

    [Parameter(Mandatory = $true)]
    [string]$DestinationName
)

# ============================
#   OPTIMIZACIÓN DEL PROCESO
# ============================
(Get-Process -Id $PID).PriorityClass = "High"

# ============================
#   PREPARAR RUTAS
# ============================
$SourceFolder      = $Path
$DestinationFolder = $DestinationName

New-Item -ItemType Directory -Path $DestinationFolder -Force | Out-Null

$TimeStamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"

$ParentFolder = Split-Path $DestinationFolder -Parent
$LogsFolder   = Join-Path $ParentFolder "Logs"
New-Item -ItemType Directory -Path $LogsFolder -Force | Out-Null

$LogFile       = Join-Path $LogsFolder "BackupLog_$TimeStamp.log"
$DetailedLog   = Join-Path $LogsFolder "Integrity_$TimeStamp.log"
$RoboCopyLog   = Join-Path $LogsFolder "Robocopy_$TimeStamp.log"

# ============================
#   LOGGING
# ============================
function Write-Log {
    param([string]$Message, [ConsoleColor]$Color = [ConsoleColor]::Gray)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $line = "[$timestamp] $Message"
    $oldColor = $Host.UI.RawUI.ForegroundColor
    $Host.UI.RawUI.ForegroundColor = $Color
    Write-Host $line
    $Host.UI.RawUI.ForegroundColor = $oldColor
    Add-Content -Path $LogFile -Value $line
}

# ============================
#   HASH PARALELO (PowerShell 7)
# ============================
function Get-FolderHash {
    param([string]$Folder)

    $files = Get-ChildItem -Path $Folder -Recurse -File
    $total = $files.Count
    $counter = 0

    $hashList = $files | ForEach-Object -Parallel {
        $hash = Get-FileHash -Algorithm SHA256 -Path $_.FullName
        "$($_.FullName) = $($hash.Hash)"
    } -ThrottleLimit 12

    return $hashList
}

# ============================
#   INICIO
# ============================
$globalStopwatch = [System.Diagnostics.Stopwatch]::StartNew()

Write-Log "===== INICIO DEL PROCESO ULTRA-RÁPIDO =====" "Cyan"
Write-Log "Origen: $SourceFolder"
Write-Log "Destino: $DestinationFolder"
Write-Log "-----------------------------------------------"

# ============================
# 1. HASH ORIGEN (PARALELO)
# ============================
Write-Log "[PASO 1] Calculando hashes del ORIGEN (paralelo)..." "Yellow"
$sw1 = [System.Diagnostics.Stopwatch]::StartNew()

$SourceHashes = Get-FolderHash -Folder $SourceFolder

$sw1.Stop()
Write-Log ("Tiempo paso 1: {0:N1} segundos" -f $sw1.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 2. ROBOCOPY ULTRA-RÁPIDO
# ============================
Write-Log "[PASO 2] Copiando con ROBOCOPY ULTRA-RÁPIDO..." "Yellow"
$sw2 = [System.Diagnostics.Stopwatch]::StartNew()

$cmd = "robocopy `"$SourceFolder`" `"$DestinationFolder`" /MIR /MT:32 /R:1 /W:1 /J /NFL /NDL /NP /LOG:`"$RoboCopyLog`""
cmd.exe /c $cmd

$sw2.Stop()
Write-Log ("Tiempo paso 2: {0:N1} segundos" -f $sw2.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 3. HASH DESTINO (PARALELO)
# ============================
Write-Log "[PASO 3] Calculando hashes del DESTINO (paralelo)..." "Yellow"
$sw3 = [System.Diagnostics.Stopwatch]::StartNew()

$DestHashes = Get-FolderHash -Folder $DestinationFolder

$sw3.Stop()
Write-Log ("Tiempo paso 3: {0:N1} segundos" -f $sw3.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 4. COMPARACIÓN
# ============================
Write-Log "[PASO 4] Comparando integridad ORIGEN → DESTINO..." "Yellow"
$sw4 = [System.Diagnostics.Stopwatch]::StartNew()

$sourceMap = @{}
foreach ($line in $SourceHashes) {
    $parts = $line -split " = "
    $sourceMap[$parts[0].Replace($SourceFolder,"")] = $parts[1]
}

$destMap = @{}
foreach ($line in $DestHashes) {
    $parts = $line -split " = "
    $destMap[$parts[0].Replace($DestinationFolder,"")] = $parts[1]
}

foreach ($file in $sourceMap.Keys) {
    if ($destMap.ContainsKey($file)) {
        if ($sourceMap[$file] -eq $destMap[$file]) {
            Write-Log "OK: $file → HASH IGUAL" "Green"
        }
        else {
            Write-Log "ERROR: $file → HASH DIFERENTE" "Red"
        }
    }
    else {
        Write-Log "FALTA EN DESTINO: $file" "Red"
    }
}

$sw4.Stop()
Write-Log ("Tiempo paso 4: {0:N1} segundos" -f $sw4.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# RESUMEN FINAL
# ============================
$globalStopwatch.Stop()

Write-Log "========== RESUMEN FINAL ==========" "Cyan"
Write-Log ("Tiempo total: {0:N1} segundos" -f $globalStopwatch.Elapsed.TotalSeconds)
Write-Log "Estado final: COMPLETADO SIN ERRORES"
Write-Log "==================================" "Cyan"

```



## Copia segura y barra de progreso
```md
param(
    [Parameter(Mandatory = $true)]
    [string]$Path,

    [Parameter(Mandatory = $true)]
    [string]$DestinationName
)

# ============================
#   OPTIMIZACIÓN DEL PROCESO
# ============================
(Get-Process -Id $PID).PriorityClass = "High"

# ============================
#   PREPARAR RUTAS
# ============================
$SourceFolder      = $Path
$DestinationFolder = $DestinationName

New-Item -ItemType Directory -Path $DestinationFolder -Force | Out-Null

$TimeStamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"

$ParentFolder = Split-Path $DestinationFolder -Parent
$LogsFolder   = Join-Path $ParentFolder "Logs"
New-Item -ItemType Directory -Path $LogsFolder -Force | Out-Null

$LogFile       = Join-Path $LogsFolder "BackupLog_$TimeStamp.log"
$DetailedLog   = Join-Path $LogsFolder "Integrity_$TimeStamp.log"
$RoboCopyLog   = Join-Path $LogsFolder "Robocopy_$TimeStamp.log"

# ============================
#   LOGGING
# ============================
function Write-Log {
    param([string]$Message, [ConsoleColor]$Color = [ConsoleColor]::Gray)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $line = "[$timestamp] $Message"
    $oldColor = $Host.UI.RawUI.ForegroundColor
    $Host.UI.RawUI.ForegroundColor = $Color
    Write-Host $line
    $Host.UI.RawUI.ForegroundColor = $oldColor
    Add-Content -Path $LogFile -Value $line
}

# ============================
#   HASH PARALELO + PROGRESO GLOBAL
# ============================
function Get-FolderHash {
    param([string]$Folder, [string]$Label)

    $files = Get-ChildItem -Path $Folder -Recurse -File
    $total = $files.Count
    $script:counter = 0

    $hashList = $files | ForEach-Object -Parallel {
        $hash = Get-FileHash -Algorithm SHA256 -Path $_.FullName
        [PSCustomObject]@{
            Path = $_.FullName
            Hash = $hash.Hash
        }
    } -ThrottleLimit 12 -AsJob | Receive-Job -Wait -AutoRemoveJob -WriteEvents | ForEach-Object {

        $script:counter++
        $percent = ($script:counter / $total) * 100

        Write-Progress `
            -Activity $Label `
            -Status "Procesando archivos..." `
            -PercentComplete $percent

        "$($_.Path) = $($_.Hash)"
    }

    Write-Progress -Activity "Completado" -Completed
    return $hashList
}

# ============================
#   ROBOCOPY CON PROGRESO GLOBAL
# ============================
function Invoke-RobocopyWithProgress {
    param(
        [string]$Source,
        [string]$Destination,
        [string]$LogFile
    )

    $totalFiles = (Get-ChildItem -Path $Source -Recurse -File).Count
    $processed = 0

    Write-Host "Copiando archivos con ROBOCOPY..."

    $psi = New-Object System.Diagnostics.ProcessStartInfo
    $psi.FileName = "robocopy.exe"
    $psi.Arguments = "`"$Source`" `"$Destination`" /MIR /MT:32 /R:1 /W:1 /J /NP /NFL /NDL"
    $psi.RedirectStandardOutput = $true
    $psi.UseShellExecute = $false
    $psi.CreateNoWindow = $true

    $proc = [System.Diagnostics.Process]::Start($psi)

    while (-not $proc.StandardOutput.EndOfStream) {
        $line = $proc.StandardOutput.ReadLine()

        if ($line -match "New File" -or $line -match "Newer" -or $line -match "Older") {
            $processed++
            $percent = [math]::Min(100, ($processed / $totalFiles) * 100)

            Write-Progress `
                -Activity "Copiando archivos con ROBOCOPY..." `
                -Status "Procesando elementos..." `
                -PercentComplete $percent
        }

        Add-Content -Path $LogFile -Value $line
    }

    Write-Progress -Activity "Completado" -Completed
}

# ============================
#   INICIO
# ============================
$globalStopwatch = [System.Diagnostics.Stopwatch]::StartNew()

Write-Log "===== INICIO DEL PROCESO ULTRA-RÁPIDO =====" "Cyan"
Write-Log "Origen: $SourceFolder"
Write-Log "Destino: $DestinationFolder"
Write-Log "-----------------------------------------------"

# ============================
# 1. HASH ORIGEN (PARALELO)
# ============================
Write-Log "[PASO 1] Calculando hashes del ORIGEN (paralelo)..." "Yellow"
$sw1 = [System.Diagnostics.Stopwatch]::StartNew()

$SourceHashes = Get-FolderHash -Folder $SourceFolder -Label "Calculando hashes del ORIGEN..."

$sw1.Stop()
Write-Log ("Tiempo paso 1: {0:N1} segundos" -f $sw1.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 2. ROBOCOPY ULTRA-RÁPIDO
# ============================
Write-Log "[PASO 2] Copiando con ROBOCOPY ULTRA-RÁPIDO..." "Yellow"
$sw2 = [System.Diagnostics.Stopwatch]::StartNew()

Invoke-RobocopyWithProgress -Source $SourceFolder -Destination $DestinationFolder -LogFile $RoboCopyLog

$sw2.Stop()
Write-Log ("Tiempo paso 2: {0:N1} segundos" -f $sw2.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 3. HASH DESTINO (PARALELO)
# ============================
Write-Log "[PASO 3] Calculando hashes del DESTINO (paralelo)..." "Yellow"
$sw3 = [System.Diagnostics.Stopwatch]::StartNew()

$DestHashes = Get-FolderHash -Folder $DestinationFolder -Label "Calculando hashes del DESTINO..."

$sw3.Stop()
Write-Log ("Tiempo paso 3: {0:N1} segundos" -f $sw3.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 4. COMPARACIÓN
# ============================
Write-Log "[PASO 4] Comparando integridad ORIGEN → DESTINO..." "Yellow"
$sw4 = [System.Diagnostics.Stopwatch]::StartNew()

$sourceMap = @{}
foreach ($line in $SourceHashes) {
    $parts = $line -split " = "
    $sourceMap[$parts[0].Replace($SourceFolder,"")] = $parts[1]
}

$destMap = @{}
foreach ($line in $DestHashes) {
    $parts = $line -split " = "
    $destMap[$parts[0].Replace($DestinationFolder,"")] = $parts[1]
}

foreach ($file in $sourceMap.Keys) {
    if ($destMap.ContainsKey($file)) {
        if ($sourceMap[$file] -eq $destMap[$file]) {
            Write-Log "OK: $file → HASH IGUAL" "Green"
        }
        else {
            Write-Log "ERROR: $file → HASH DIFERENTE" "Red"
        }
    }
    else {
        Write-Log "FALTA EN DESTINO: $file" "Red"
    }
}

$sw4.Stop()
Write-Log ("Tiempo paso 4: {0:N1} segundos" -f $sw4.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# RESUMEN FINAL
# ============================
$globalStopwatch.Stop()

Write-Log "========== RESUMEN FINAL ==========" "Cyan"
Write-Log ("Tiempo total: {0:N1} segundos" -f $globalStopwatch.Elapsed.TotalSeconds)
Write-Log "Estado final: COMPLETADO SIN ERRORES"
Write-Log "==================================" "Cyan"

```


## Copia Segura con menu

```md
param(
    [Parameter(Mandatory = $true)]
    [string]$Path,

    [Parameter(Mandatory = $true)]
    [string]$DestinationName
)

# ============================
#   OPTIMIZACIÓN DEL PROCESO
# ============================
(Get-Process -Id $PID).PriorityClass = "High"

# ============================
#   PREPARAR RUTAS
# ============================
$SourceFolder      = $Path
$DestinationFolder = $DestinationName

New-Item -ItemType Directory -Path $DestinationFolder -Force | Out-Null

$TimeStamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"

$ParentFolder = Split-Path $DestinationFolder -Parent
$LogsFolder   = Join-Path $ParentFolder "Logs"
New-Item -ItemType Directory -Path $LogsFolder -Force | Out-Null

$LogFile       = Join-Path $LogsFolder "BackupLog_$TimeStamp.log"
$DetailedLog   = Join-Path $LogsFolder "Integrity_$TimeStamp.log"
$RoboCopyLog   = Join-Path $LogsFolder "Robocopy_$TimeStamp.log"

# ============================
#   LOGGING
# ============================
function Write-Log {
    param([string]$Message, [ConsoleColor]$Color = [ConsoleColor]::Gray)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $line = "[$timestamp] $Message"
    $oldColor = $Host.UI.RawUI.ForegroundColor
    $Host.UI.RawUI.ForegroundColor = $Color
    Write-Host $line
    $Host.UI.RawUI.ForegroundColor = $oldColor
    Add-Content -Path $LogFile -Value $line
}

# ============================
#   HASH PARALELO + PROGRESO GLOBAL
# ============================
function Get-FolderHash {
    param([string]$Folder, [string]$Label)

    $files = Get-ChildItem -Path $Folder -Recurse -File
    $total = $files.Count
    $script:counter = 0

    $hashList = $files | ForEach-Object -Parallel {
        $hash = Get-FileHash -Algorithm SHA256 -Path $_.FullName
        [PSCustomObject]@{
            Path = $_.FullName
            Hash = $hash.Hash
        }
    } -ThrottleLimit 12 -AsJob | Receive-Job -Wait -AutoRemoveJob -WriteEvents | ForEach-Object {

        $script:counter++
        $percent = ($script:counter / $total) * 100

        Write-Progress `
            -Activity $Label `
            -Status "Procesando archivos..." `
            -PercentComplete $percent

        "$($_.Path) = $($_.Hash)"
    }

    Write-Progress -Activity "Completado" -Completed
    return $hashList
}

# ============================
#   ROBOCOPY CON PROGRESO GLOBAL
# ============================
function Invoke-RobocopyWithProgress {
    param(
        [string]$Source,
        [string]$Destination,
        [string]$LogFile
    )

    $totalFiles = (Get-ChildItem -Path $Source -Recurse -File).Count
    $processed = 0

    Write-Host "Copiando archivos con ROBOCOPY..."

    $psi = New-Object System.Diagnostics.ProcessStartInfo
    $psi.FileName = "robocopy.exe"
    $psi.Arguments = "`"$Source`" `"$Destination`" /MIR /MT:32 /R:1 /W:1 /J /NP /NFL /NDL"
    $psi.RedirectStandardOutput = $true
    $psi.UseShellExecute = $false
    $psi.CreateNoWindow = $true

    $proc = [System.Diagnostics.Process]::Start($psi)

    while (-not $proc.StandardOutput.EndOfStream) {
        $line = $proc.StandardOutput.ReadLine()

        if ($line -match "New File" -or $line -match "Newer" -or $line -match "Older") {
            $processed++
            $percent = [math]::Min(100, ($processed / $totalFiles) * 100)

            Write-Progress `
                -Activity "Copiando archivos con ROBOCOPY..." `
                -Status "Procesando elementos..." `
                -PercentComplete $percent
        }

        Add-Content -Path $LogFile -Value $line
    }

    Write-Progress -Activity "Completado" -Completed
}

# ============================
#   COMPARACIÓN DE HASHES
# ============================
function Compare-Hashes {
    foreach ($file in $sourceMap.Keys) {
        if ($destMap.ContainsKey($file)) {
            if ($sourceMap[$file] -eq $destMap[$file]) {
                Write-Log "OK: $file → HASH IGUAL" "Green"
            }
            else {
                Write-Log "ERROR: $file → HASH DIFERENTE" "Red"
            }
        }
        else {
            Write-Log "FALTA EN DESTINO: $file" "Red"
        }
    }
}

# ============================
#   MENÚ PROFESIONAL
# ============================
function Show-Menu {
    Clear-Host
    Write-Host "======================================================" -ForegroundColor Cyan
    Write-Host "                SISTEMA DE BACKUP AVANZADO            " -ForegroundColor Yellow
    Write-Host "======================================================" -ForegroundColor Cyan
    Write-Host ""
    Write-Host "  1. Ejecutar proceso completo de sincronización (MIRROR)" -ForegroundColor Green
    Write-Host "  2. Generar hashes del directorio de origen" -ForegroundColor White
    Write-Host "  3. Realizar copia de seguridad (solo ROBOCOPY)" -ForegroundColor White
    Write-Host "  4. Generar hashes del directorio de destino" -ForegroundColor White
    Write-Host "  5. Comparar integridad entre origen y destino" -ForegroundColor White
    Write-Host "  6. Finalizar y cerrar la herramienta" -ForegroundColor Red
    Write-Host ""
}

function Start-Menu {
    do {
        Show-Menu
        $choice = Read-Host "Seleccione una opción"

        switch ($choice) {

            "1" {
                Write-Log "=== PROCESO COMPLETO DE SINCRONIZACIÓN ===" "Cyan"

                $SourceHashes = Get-FolderHash -Folder $SourceFolder -Label "Hashes ORIGEN"
                Invoke-RobocopyWithProgress -Source $SourceFolder -Destination $DestinationFolder -LogFile $RoboCopyLog
                $DestHashes = Get-FolderHash -Folder $DestinationFolder -Label "Hashes DESTINO"

                $global:sourceMap = @{}
                foreach ($line in $SourceHashes) {
                    $parts = $line -split " = "
                    $sourceMap[$parts[0].Replace($SourceFolder,"")] = $parts[1]
                }

                $global:destMap = @{}
                foreach ($line in $DestHashes) {
                    $parts = $line -split " = "
                    $destMap[$parts[0].Replace($DestinationFolder,"")] = $parts[1]
                }

                Compare-Hashes
                Pause
            }

            "2" {
                Write-Log "=== HASHES DEL ORIGEN ===" "Cyan"
                $global:SourceHashes = Get-FolderHash -Folder $SourceFolder -Label "Hashes ORIGEN"
                Pause
            }

            "3" {
                Write-Log "=== COPIA SOLO ROBOCOPY ===" "Cyan"
                Invoke-RobocopyWithProgress -Source $SourceFolder -Destination $DestinationFolder -LogFile $RoboCopyLog
                Pause
            }

            "4" {
                Write-Log "=== HASHES DEL DESTINO ===" "Cyan"
                $global:DestHashes = Get-FolderHash -Folder $DestinationFolder -Label "Hashes DESTINO"
                Pause
            }

            "5" {
                Write-Log "=== COMPARACIÓN DE INTEGRIDAD ===" "Cyan"
                Compare-Hashes
                Pause
            }

            "6" {
                Write-Host "Cerrando la herramienta..." -ForegroundColor Red
                return
            }

            default {
                Write-Host "Opción no válida. Intente nuevamente." -ForegroundColor Red
                Start-Sleep -Seconds 1
            }
        }

    } while ($true)
}

# ============================
#   EJECUCIÓN DEL MENÚ
# ============================
Start-Menu


```
## COPIA CON ENCABEZADOS CORPORATIVOS
```md
param(
    [Parameter(Mandatory = $true)]
    [string]$Path,

    [Parameter(Mandatory = $true)]
    [string]$DestinationName
)

Clear-Host
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host "                 ENTERPRISE BACKUP SUITE               " -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host ""

# ============================
#   OPTIMIZACIÓN DEL PROCESO
# ============================
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host "                 OPTIMIZACIÓN DEL PROCESO              " -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

(Get-Process -Id $PID).PriorityClass = "High"

# ============================
#   PREPARAR RUTAS
# ============================
Write-Host ""
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host "                     PREPARANDO RUTAS                  " -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

$SourceFolder      = $Path
$DestinationFolder = $DestinationName

New-Item -ItemType Directory -Path $DestinationFolder -Force | Out-Null

$TimeStamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"

$ParentFolder = Split-Path $DestinationFolder -Parent
$LogsFolder   = Join-Path $ParentFolder "Logs"
New-Item -ItemType Directory -Path $LogsFolder -Force | Out-Null

$LogFile       = Join-Path $LogsFolder "BackupLog_$TimeStamp.log"
$DetailedLog   = Join-Path $LogsFolder "Integrity_$TimeStamp.log"
$RoboCopyLog   = Join-Path $LogsFolder "Robocopy_$TimeStamp.log"

# ============================
#   LOGGING
# ============================
Write-Host ""
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host "                     SISTEMA DE LOGS                   " -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

function Write-Log {
    param([string]$Message, [ConsoleColor]$Color = [ConsoleColor]::Gray)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $line = "[$timestamp] $Message"
    $oldColor = $Host.UI.RawUI.ForegroundColor
    $Host.UI.RawUI.ForegroundColor = $Color
    Write-Host $line
    $Host.UI.RawUI.ForegroundColor = $oldColor
    Add-Content -Path $LogFile -Value $line
}

# ============================
#   HASH PARALELO + PROGRESO GLOBAL
# ============================
Write-Host ""
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host "                     HASH EN PARALELO                  " -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

function Get-FolderHash {
    param([string]$Folder, [string]$Label)

    $files = Get-ChildItem -Path $Folder -Recurse -File
    $total = $files.Count
    $script:counter = 0

    $hashList = $files | ForEach-Object -Parallel {
        $hash = Get-FileHash -Algorithm SHA256 -Path $_.FullName
        [PSCustomObject]@{
            Path = $_.FullName
            Hash = $hash.Hash
        }
    } -ThrottleLimit 12 -AsJob | Receive-Job -Wait -AutoRemoveJob -WriteEvents | ForEach-Object {

        $script:counter++
        $percent = ($script:counter / $total) * 100

        Write-Progress `
            -Activity $Label `
            -Status "Procesando archivos..." `
            -PercentComplete $percent

        "$($_.Path) = $($_.Hash)"
    }

    Write-Progress -Activity "Completado" -Completed
    return $hashList
}

# ============================
#   ROBOCOPY CON PROGRESO GLOBAL
# ============================
Write-Host ""
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host "                     COPIA ROBOCOPY                    " -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

function Invoke-RobocopyWithProgress {
    param(
        [string]$Source,
        [string]$Destination,
        [string]$LogFile
    )

    $totalFiles = (Get-ChildItem -Path $Source -Recurse -File).Count
    $processed = 0

    Write-Host "Copiando archivos con ROBOCOPY..."

    $psi = New-Object System.Diagnostics.ProcessStartInfo
    $psi.FileName = "robocopy.exe"
    $psi.Arguments = "`"$Source`" `"$Destination`" /MIR /MT:32 /R:1 /W:1 /J /NP /NFL /NDL"
    $psi.RedirectStandardOutput = $true
    $psi.UseShellExecute = $false
    $psi.CreateNoWindow = $true

    $proc = [System.Diagnostics.Process]::Start($psi)

    while (-not $proc.StandardOutput.EndOfStream) {
        $line = $proc.StandardOutput.ReadLine()

        if ($line -match "New File" -or $line -match "Newer" -or $line -match "Older") {
            $processed++
            $percent = [math]::Min(100, ($processed / $totalFiles) * 100)

            Write-Progress `
                -Activity "Copiando archivos con ROBOCOPY..." `
                -Status "Procesando elementos..." `
                -PercentComplete $percent
        }

        Add-Content -Path $LogFile -Value $line
    }

    Write-Progress -Activity "Completado" -Completed
}

# ============================
#   COMPARACIÓN DE HASHES
# ============================
Write-Host ""
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host "                   COMPARACIÓN DE HASHES               " -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

function Compare-Hashes {
    foreach ($file in $sourceMap.Keys) {
        if ($destMap.ContainsKey($file)) {
            if ($sourceMap[$file] -eq $destMap[$file]) {
                Write-Log "OK: $file → HASH IGUAL" "Green"
            }
            else {
                Write-Log "ERROR: $file → HASH DIFERENTE" "Red"
            }
        }
        else {
            Write-Log "FALTA EN DESTINO: $file" "Red"
        }
    }
}

# ============================
#   MENÚ PROFESIONAL
# ============================
function Show-Menu {
    Clear-Host
    Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
    Write-Host "                 ENTERPRISE BACKUP SUITE               " -ForegroundColor Yellow
    Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
    Write-Host ""
    Write-Host "  1. Ejecutar proceso completo de sincronización (MIRROR)" -ForegroundColor Green
    Write-Host "  2. Generar hashes del directorio de origen" -ForegroundColor White
    Write-Host "  3. Realizar copia de seguridad (solo ROBOCOPY)" -ForegroundColor White
    Write-Host "  4. Generar hashes del directorio de destino" -ForegroundColor White
    Write-Host "  5. Comparar integridad entre origen y destino" -ForegroundColor White
    Write-Host "  6. Finalizar y cerrar la herramienta" -ForegroundColor Red
    Write-Host ""
}

function Start-Menu {
    do {
        Show-Menu
        $choice = Read-Host "Seleccione una opción"

        switch ($choice) {

            "1" {
                Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
                Write-Host "         EJECUTANDO PROCESO COMPLETO DE BACKUP         " -ForegroundColor Yellow
                Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

                Write-Log "=== PROCESO COMPLETO DE SINCRONIZACIÓN ===" "Cyan"

                $SourceHashes = Get-FolderHash -Folder $SourceFolder -Label "Hashes ORIGEN"
                Invoke-RobocopyWithProgress -Source $SourceFolder -Destination $DestinationFolder -LogFile $RoboCopyLog
                $DestHashes = Get-FolderHash -Folder $DestinationFolder -Label "Hashes DESTINO"

                $global:sourceMap = @{}
                foreach ($line in $SourceHashes) {
                    $parts = $line -split " = "
                    $sourceMap[$parts[0].Replace($SourceFolder,"")] = $parts[1]
                }

                $global:destMap = @{}
                foreach ($line in $DestHashes) {
                    $parts = $line -split " = "
                    $destMap[$parts[0].Replace($DestinationFolder,"")] = $parts[1]
                }

                Compare-Hashes
                Pause
            }

            "2" {
                Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
                Write-Host "                 HASHES DEL DIRECTORIO ORIGEN          " -ForegroundColor Yellow
                Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

                Write-Log "=== HASHES DEL ORIGEN ===" "Cyan"
                $global:SourceHashes = Get-FolderHash -Folder $SourceFolder -Label "Hashes ORIGEN"
                Pause
            }

            "3" {
                Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
                Write-Host "                     COPIA SOLO ROBOCOPY               " -ForegroundColor Yellow
                Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

                Write-Log "=== COPIA SOLO ROBOCOPY ===" "Cyan"
                Invoke-RobocopyWithProgress -Source $SourceFolder -Destination $DestinationFolder -LogFile $RoboCopyLog
                Pause
            }

            "4" {
                Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
                Write-Host "                 HASHES DEL DIRECTORIO DESTINO         " -ForegroundColor Yellow
                Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

                Write-Log "=== HASHES DEL DESTINO ===" "Cyan"
                $global:DestHashes = Get-FolderHash -Folder $DestinationFolder -Label "Hashes DESTINO"
                Pause
            }

            "5" {
                Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
                Write-Host "                 COMPARACIÓN DE INTEGRIDAD             " -ForegroundColor Yellow
                Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

                Write-Log "=== COMPARACIÓN DE INTEGRIDAD ===" "Cyan"
                Compare-Hashes
                Pause
            }

            "6" {
                Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
                Write-Host "                     CERRANDO SISTEMA                  " -ForegroundColor Red
                Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
                return
            }

            default {
                Write-Host "Opción no válida. Intente nuevamente." -ForegroundColor Red
                Start-Sleep -Seconds 1
            }
        }

    } while ($true)
}

# ============================
#   EJECUCIÓN DEL MENÚ
# ============================
Start-Menu

```

## targets.json
```md
{
  "OCM": [
    { "Host": "192.168.1.10", "Ports": [80, 443] },
    { "Host": "srv-ocm01", "Ports": [3389] }
  ],
  "CORE": [
    { "Host": "10.0.0.5", "Ports": [22, 443, 8080] }
  ],
  "KIOSK": [
    { "Host": "kiosk-01", "Ports": [80] },
    { "Host": "kiosk-02", "Ports": [80, 443] }
  ],
  "OTROS": [
    { "Host": "google.com", "Ports": [443] }
  ]
}

```

## Prueba de red
```md
param(
    [Parameter(Mandatory = $true)]
    [string]$ConfigPath,          # Ruta al JSON (targets.json)

    [string]$LogFolder = "C:\NetTestSuite\Logs"
)

Clear-Host
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host "                     NET TEST SUITE                    " -ForegroundColor Yellow
Write-Host "                 Version 1.1 - by Juan                 " -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host ""

# ============================
#   PREPARAR RUTAS / LOG
# ============================
New-Item -ItemType Directory -Path $LogFolder -Force | Out-Null
$TimeStamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"
$LogFile   = Join-Path $LogFolder "NetTest_$TimeStamp.log"

function Write-Log {
    param([string]$Message, [ConsoleColor]$Color = [ConsoleColor]::Gray)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $line = "[$timestamp] $Message"
    $oldColor = $Host.UI.RawUI.ForegroundColor
    $Host.UI.RawUI.ForegroundColor = $Color
    Write-Host $line
    $Host.UI.RawUI.ForegroundColor = $oldColor
    Add-Content -Path $LogFile -Value $line
}

# ============================
#   DETECTAR IP LOCAL
# ============================
$MyIP = (Get-NetIPAddress -AddressFamily IPv4 `
         | Where-Object { $_.IPAddress -notlike "169.*" -and $_.IPAddress -notlike "127.*" } `
         | Select-Object -First 1 -ExpandProperty IPAddress)

Write-Host "IP local detectada: $MyIP" -ForegroundColor Cyan
Write-Log  "IP local detectada: $MyIP"
Write-Host ""

# ============================
#   CARGAR CONFIGURACIÓN JSON
# ============================
if (-not (Test-Path $ConfigPath)) {
    Write-Log "ERROR: No se encuentra el archivo de configuración JSON: $ConfigPath" "Red"
    exit 1
}

try {
    $jsonRaw = Get-Content -Path $ConfigPath -Raw
    $Targets = $jsonRaw | ConvertFrom-Json
}
catch {
    Write-Log "ERROR al leer o parsear el JSON: $($_.Exception.Message)" "Red"
    exit 1
}

Write-Log "Cargando configuración desde: $ConfigPath" "Cyan"
Write-Log "Grupos definidos: $($Targets.PSObject.Properties.Name -join ', ')"
Write-Log "------------------------------------------------------------"

# ============================
#   LISTAR OBJETIVOS
# ============================
function Show-Targets {
    Write-Host ""
    Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
    Write-Host "                 LISTA DE OBJETIVOS (JSON)             " -ForegroundColor Yellow
    Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

    Write-Log "Mostrando lista de objetivos desde JSON."

    foreach ($groupProp in $Targets.PSObject.Properties) {
        $groupName = $groupProp.Name
        $items     = $groupProp.Value

        Write-Host ""
        Write-Host "Grupo: $groupName" -ForegroundColor Cyan
        Write-Log  "Grupo: $groupName"

        foreach ($item in $items) {
            $ports = ($item.Ports -join ",")
            Write-Host " - Host: $($item.Host)  Puertos: $ports"
            Write-Log  " - Host: $($item.Host)  Puertos: $ports"
        }
    }

    Write-Host ""
}

# ============================
#   PRUEBAS DE RED
# ============================
function Test-NetworkTargets {
    $allItems = @()

    foreach ($groupProp in $Targets.PSObject.Properties) {
        $groupName = $groupProp.Name
        foreach ($item in $groupProp.Value) {
            $allItems += [PSCustomObject]@{
                Group = $groupName
                Host  = $item.Host
                Ports = $item.Ports
            }
        }
    }

    if ($allItems.Count -eq 0) {
        Write-Log "No hay objetivos definidos en el JSON." "Yellow"
        return
    }

    foreach ($entry in $allItems) {

        Write-Host ""
        Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
        Write-Host ("GRUPO: {0}  HOST: {1}" -f $entry.Group, $entry.Host) -ForegroundColor Yellow
        Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

        Write-Log "Grupo: $($entry.Group) - Probando host: $($entry.Host)" "Cyan"

        # PING
        $pingResult = Test-NetConnection -ComputerName $entry.Host -WarningAction SilentlyContinue
        if ($pingResult.PingSucceeded) {
            Write-Host "PING OK → $($entry.Host)" -ForegroundColor Green
            Write-Log  "PING OK → $($entry.Host)"
        }
        else {
            Write-Host "PING FALLÓ → $($entry.Host)" -ForegroundColor Red
            Write-Log  "PING FALLÓ → $($entry.Host)"
        }

        # PUERTOS
        foreach ($port in $entry.Ports) {
            $tcpResult = Test-NetConnection -ComputerName $entry.Host -Port $port -WarningAction SilentlyContinue
            if ($tcpResult.TcpTestSucceeded) {
                Write-Host "TCP OK (puerto $port) → $($entry.Host)" -ForegroundColor Green
                Write-Log  "TCP OK (puerto $port) → $($entry.Host)"
            }
            else {
                Write-Host "TCP FALLÓ (puerto $port) → $($entry.Host)" -ForegroundColor Red
                Write-Log  "TCP FALLÓ (puerto $port) → $($entry.Host)"
            }
        }

        Write-Host "------------------------------------------------------------"
        Write-Log  "------------------------------------------------------------"
    }

    Write-Log "Pruebas de red completadas."
}

# ============================
#   MENÚ PROFESIONAL
# ============================
function Show-Menu {
    Clear-Host
    Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
    Write-Host "                     NET TEST SUITE                    " -ForegroundColor Yellow
    Write-Host "                 Version 1.1 - by Juan                 " -ForegroundColor Yellow
    Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
    Write-Host ""
    Write-Host "IP local detectada: $MyIP" -ForegroundColor Cyan
    Write-Host ""
    Write-Host "  1. Ejecutar pruebas de red" -ForegroundColor Green
    Write-Host "  2. Mostrar lista de objetivos (desde JSON)" -ForegroundColor White
    Write-Host "  3. Salir" -ForegroundColor Red
    Write-Host ""
}

function Start-Menu {
    do {
        Show-Menu
        $choice = Read-Host "Seleccione una opción"

        switch ($choice) {

            "1" {
                Test-NetworkTargets
                Pause
            }

            "2" {
                Show-Targets
                Pause
            }

            "3" {
                Write-Host "Cerrando Net Test Suite..." -ForegroundColor Red
                Write-Log "Cierre de la herramienta solicitado por el usuario." "Yellow"
                return
            }

            default {
                Write-Host "Opción no válida." -ForegroundColor Red
                Start-Sleep -Seconds 1
            }
        }

    } while ($true)
}

# ============================
#   EJECUCIÓN DEL MENÚ
# ============================
Start-Menu

```

## No se guarda en carpeta, el log tambien pone nombre, tambien muestra el host por pantalla
```md
param(
    [Parameter(Mandatory = $true)]
    [string]$ConfigPath
)

Clear-Host
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host "                     NET TEST SUITE                    " -ForegroundColor Yellow
Write-Host "                 Version 1.1 - by Juan                 " -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host ""

# ============================
#   RUTA DEL LOG (MISMA CARPETA DEL SCRIPT)
# ============================
$ScriptFolder = Split-Path -Parent $MyInvocation.MyCommand.Path
$LogFolder    = Join-Path $ScriptFolder "Logs"

New-Item -ItemType Directory -Path $LogFolder -Force | Out-Null

$MyHostName = $env:COMPUTERNAME
$TimeStamp  = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"

$LogFile = Join-Path $LogFolder "NetTest_${MyHostName}_$TimeStamp.log"

Write-Host "Log generado en: $LogFile" -ForegroundColor Cyan
Write-Host ""

# ============================
#   LOGGING
# ============================
function Write-Log {
    param([string]$Message)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    Add-Content -Path $LogFile -Value "[$timestamp] $Message"
}

# ============================
#   DETECTAR IP LOCAL + HOSTNAME
# ============================
$MyIP = (Get-NetIPAddress -AddressFamily IPv4 `
         | Where-Object { $_.IPAddress -notlike "169.*" -and $_.IPAddress -notlike "127.*" } `
         | Select-Object -First 1 -ExpandProperty IPAddress)

Write-Host "IP local detectada: $MyIP" -ForegroundColor Cyan
Write-Host "Hostname local: $MyHostName" -ForegroundColor Cyan
Write-Host ""

Write-Log "IP local detectada: $MyIP"
Write-Log "Hostname local: $MyHostName"

# ============================
#   CARGAR CONFIGURACIÓN JSON
# ============================
if (-not (Test-Path $ConfigPath)) {
    Write-Host "ERROR: No se encuentra el archivo JSON." -ForegroundColor Red
    Write-Log  "ERROR: No se encuentra el archivo JSON."
    exit 1
}

try {
    $jsonRaw = Get-Content -Path $ConfigPath -Raw
    $Targets = $jsonRaw | ConvertFrom-Json
}
catch {
    Write-Host "ERROR al leer el JSON." -ForegroundColor Red
    Write-Log  "ERROR al leer el JSON."
    exit 1
}

Write-Log "Cargando configuración desde: $ConfigPath"
Write-Log "Grupos definidos: $($Targets.PSObject.Properties.Name -join ', ')"
Write-Log "------------------------------------------------------------"

# ============================
#   LISTAR OBJETIVOS
# ============================
function Show-Targets {
    Write-Host ""
    Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
    Write-Host "                 LISTA DE OBJETIVOS (JSON)             " -ForegroundColor Yellow
    Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

    foreach ($groupProp in $Targets.PSObject.Properties) {
        $groupName = $groupProp.Name
        Write-Host ""
        Write-Host "Grupo: $groupName" -ForegroundColor Cyan
        Write-Log  "Grupo: $groupName"

        foreach ($item in $groupProp.Value) {
            $ports = ($item.Ports -join ",")
            Write-Host " - Host: $($item.Host)  Puertos: $ports"
            Write-Log  " - Host: $($item.Host)  Puertos: $ports"
        }
    }
}

# ============================
#   PRUEBAS DE RED
# ============================
function Test-NetworkTargets {

    $allItems = @()

    foreach ($groupProp in $Targets.PSObject.Properties) {
        foreach ($item in $groupProp.Value) {
            $allItems += [PSCustomObject]@{
                Group = $groupProp.Name
                Host  = $item.Host
                Ports = $item.Ports
            }
        }
    }

    foreach ($entry in $allItems) {

        Write-Host ""
        Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
        Write-Host ("GRUPO: {0}  HOST: {1}" -f $entry.Group, $entry.Host) -ForegroundColor Yellow
        Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

        Write-Log "Grupo: $($entry.Group) - Probando host: $($entry.Host)"

        # PING
        $pingResult = Test-NetConnection -ComputerName $entry.Host -WarningAction SilentlyContinue
        if ($pingResult.PingSucceeded) {
            Write-Host "PING OK → $($entry.Host)" -ForegroundColor Green
            Write-Log  "PING OK → $($entry.Host)"
        }
        else {
            Write-Host "PING FALLÓ → $($entry.Host)" -ForegroundColor Red
            Write-Log  "PING FALLÓ → $($entry.Host)"
        }

        # PUERTOS
        foreach ($port in $entry.Ports) {
            $tcpResult = Test-NetConnection -ComputerName $entry.Host -Port $port -WarningAction SilentlyContinue
            if ($tcpResult.TcpTestSucceeded) {
                Write-Host "TCP OK (puerto $port) → $($entry.Host)" -ForegroundColor Green
                Write-Log  "TCP OK (puerto $port) → $($entry.Host)"
            }
            else {
                Write-Host "TCP FALLÓ (puerto $port) → $($entry.Host)" -ForegroundColor Red
                Write-Log  "TCP FALLÓ (puerto $port) → $($entry.Host)"
            }
        }

        Write-Host "------------------------------------------------------------"
        Write-Log  "------------------------------------------------------------"
    }

    Write-Log "Pruebas de red completadas."
}

# ============================
#   MENÚ
# ============================
function Show-Menu {
    Clear-Host
    Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
    Write-Host "                     NET TEST SUITE                    " -ForegroundColor Yellow
    Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
    Write-Host ""
    Write-Host "IP local detectada: $MyIP" -ForegroundColor Cyan
    Write-Host "Hostname local: $MyHostName" -ForegroundColor Cyan
    Write-Host "Log generado en: $LogFile" -ForegroundColor Cyan
    Write-Host ""
    Write-Host "  1. Ejecutar pruebas de red" -ForegroundColor Green
    Write-Host "  2. Mostrar lista de objetivos" -ForegroundColor White
    Write-Host "  3. Salir" -ForegroundColor Red
    Write-Host ""
}

function Start-Menu {
    do {
        Show-Menu
        $choice = Read-Host "Seleccione una opción"

        switch ($choice) {
            "1" { Test-NetworkTargets; Pause }
            "2" { Show-Targets; Pause }
            "3" { return }
            default { Write-Host "Opción no válida." -ForegroundColor Red }
        }

    } while ($true)
}

Start-Menu

```




