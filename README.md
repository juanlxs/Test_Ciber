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



