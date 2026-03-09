# Test_Ciber

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



```md

```
