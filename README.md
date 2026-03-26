# Test_Ciber

##json
```md
{
    "metadata": {
        "name": "Anonymization Rules",
        "version": "2.0",
        "author": "Security & Compliance",
        "created": "2026-03-26",
        "description": "Standardized rules for literal and regex-based anonymization of sensitive information."
    },

    "settings": {
        "case_sensitive": false,
        "whole_word_only": true,
        "default_encoding": "UTF-8"
    },

    "replacements": [
        {
            "match": "Juan",
            "replace": "UserA",
            "category": "personal_name",
            "enabled": true
        },
        {
            "match": "Madrid",
            "replace": "CityX",
            "category": "location",
            "enabled": true
        },
        {
            "match": "Empresa",
            "replace": "OrgY",
            "category": "company",
            "enabled": true
        },
        {
            "match": "Servidor",
            "replace": "NodeZ",
            "category": "infrastructure",
            "enabled": true
        }
    ],

    "regex_rules": [
        {
            "pattern": "\\b\\d{8}[A-Z]\\b",
            "replace": "DNI-XXXX",
            "category": "dni",
            "description": "Spanish DNI format (8 digits + letter)",
            "enabled": true
        },
        {
            "pattern": "\\b[0-9]{1,3}(?:\\.[0-9]{1,3}){3}\\b",
            "replace": "IP-REDACTED",
            "category": "ip_address",
            "description": "IPv4 address",
            "enabled": true
        },
        {
            "pattern": "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}",
            "replace": "email@hidden",
            "category": "email",
            "description": "Email address",
            "enabled": true
        },
        {
            "pattern": "\\b[0-9]{9}\\b",
            "replace": "PHONE-XXXX",
            "category": "phone",
            "description": "Generic 9-digit phone number",
            "enabled": false
        }
    ]
}

```

## copia-segura_v4.1.ps1
```md
param(
    [Parameter(Mandatory = $true)]
    [string]$Path,

    [Parameter(Mandatory = $true)]
    [string]$DestinationName,

    [Parameter(Mandatory = $false)]
    [string]$SevenZipPath = "C:\Program Files\7-Zip\7z.exe"
)

Clear-Host
Write-Host "  ███████╗██╗  ██╗███████╗███████╗██████╗ ██████╗ ██╗██████╗  " -ForegroundColor Yellow
Write-Host "  ██╔════╝██║  ██║██╔════╝██╔════╝██╔══██╗██╔══██╗██║██╔══██╗ " -ForegroundColor Yellow
Write-Host "  ███████╗███████║█████╗  █████╗  ██████╔╝██║  ██║██║██████╔╝ " -ForegroundColor Yellow
Write-Host "  ╚════██║██╔══██║██╔══╝  ██╔══╝  ██╔═══╝ ██║  ██║██║██╔═══╝  " -ForegroundColor Yellow
Write-Host "  ███████║██║  ██║███████╗███████╗██║     ██████╔╝██║██║      " -ForegroundColor Yellow
Write-Host "  ╚══════╝╚═╝  ╚═╝╚══════╝╚══════╝╚═╝     ╚═════╝ ╚═╝╚═╝      " -ForegroundColor Yellow
Write-Host "══════════════════════════════════════════════════════════════" -ForegroundColor Yellow
Write-Host "              COPIA SEGURA v4.1 · SHEEPDIP SUITE              " -ForegroundColor Yellow
Write-Host "══════════════════════════════════════════════════════════════" -ForegroundColor Yellow
Write-Host ""

#INICIADOR DE MEDICIÓN DE TIEMPO
$startTime = Get-Date

# OPTIMIZACION DEL PROCESO
Write-Host "Ajustando prioridad del proceso..." -ForegroundColor DarkGray
(Get-Process -Id $PID).PriorityClass = "High"

# PREPARAR RUTAS
$SourceFolder = $Path
$DestinationFolder = $DestinationName

New-Item -ItemType Directory -Path $DestinationFolder -Force | Out-Null

$TimeStamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"

$ParentFolder = Split-Path $DestinationFolder -Parent
$LogsFolder = Join-Path $ParentFolder "logs"
New-Item -ItemType Directory -Path $LogsFolder -Force | Out-Null

# LOG ÚNICO
$LogUnified = Join-Path $LogsFolder "copia_segura_$TimeStamp.log"

# LOGGING
function Write-Log {
    param(
        [string]$Message,
        [string]$Tag = "INFO",
        [ConsoleColor]$Color = "Gray"
    )
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $line = "[$timestamp] [$Tag] $Message"
    Write-Host $line -ForegroundColor $Color
    Add-Content -Path $LogUnified -Value $line
}

function Write-ColoredLine {
    param(
        [string]$Prefix,
        [string]$Text,
        [ConsoleColor]$Color = "Cyan"
    )

    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"

    if ([string]::IsNullOrWhiteSpace($Text)) {
        $line = "[$timestamp] [$Prefix]"
    }
    else {
        $line = "[$timestamp] [$Prefix] $Text"
    }

    Write-Host $line -ForegroundColor $Color
    Add-Content -Path $LogUnified -Value $line
}

# INFORMACIÓN PROFESIONAL DE DISCO
function Get-DiskInfoBlock {
    param(
        [string]$Path,
        [string]$DiskTag
    )

    $driveLetter = ($Path.Substring(0, 1)).ToUpper()

    # Obtener volumen
    $vol = Get-Volume -DriveLetter $driveLetter -ErrorAction SilentlyContinue
    $psd = Get-PSDrive -Name $driveLetter -ErrorAction SilentlyContinue

    # Obtener partición asociada
    $partition = Get-Partition -DriveLetter $driveLetter -ErrorAction SilentlyContinue

    # Obtener disco físico real
    $disk = Get-Disk -Number $partition.DiskNumber -ErrorAction SilentlyContinue

    # Obtener información avanzada WMI
    $wmiDisk = Get-CimInstance Win32_DiskDrive | Where-Object {
        $_.Index -eq $partition.DiskNumber
    }

    # Fallbacks
    $model = $wmiDisk.Model
    if ([string]::IsNullOrWhiteSpace($model)) { $model = "No disponible" }

    $serial = $wmiDisk.SerialNumber
    if ([string]::IsNullOrWhiteSpace($serial)) { $serial = "No disponible" }

    $manufacturer = $wmiDisk.Manufacturer
    if ([string]::IsNullOrWhiteSpace($manufacturer)) { $manufacturer = "No disponible" }

    $interface = $wmiDisk.InterfaceType
    if ([string]::IsNullOrWhiteSpace($interface)) { $interface = "No disponible" }

    $mediaType = $wmiDisk.MediaType
    if ([string]::IsNullOrWhiteSpace($mediaType)) { $mediaType = "No disponible" }

    # Espacio
    $totalGB = [math]::Round(($psd.Used + $psd.Free) / 1GB, 2)
    $usedGB = [math]::Round($psd.Used / 1GB, 2)
    $freeGB = [math]::Round($psd.Free / 1GB, 2)
    $pctFree = [math]::Round(($psd.Free / ($psd.Used + $psd.Free)) * 100, 2)

    # Salida profesional
    Write-ColoredLine -Prefix $DiskTag -Text "=== INFORMACIÓN DEL DISCO (${driveLetter}:) ===" "Yellow"
    Write-ColoredLine -Prefix $DiskTag -Text "Ruta: $Path" "Gray"
    Write-ColoredLine -Prefix $DiskTag -Text "Modelo: $model" "Gray"
    Write-ColoredLine -Prefix $DiskTag -Text "Número de serie: $serial" "Gray"
    Write-ColoredLine -Prefix $DiskTag -Text "Fabricante: $manufacturer" "Gray"
    Write-ColoredLine -Prefix $DiskTag -Text "Tipo de medio: $mediaType" "Gray"
    Write-ColoredLine -Prefix $DiskTag -Text "Interfaz: $interface" "Gray"
    Write-ColoredLine -Prefix $DiskTag -Text "Sistema de archivos: $($vol.FileSystem)" "Gray"
    Write-ColoredLine -Prefix $DiskTag -Text "Etiqueta del volumen: $($vol.FileSystemLabel)" "Gray"
    Write-ColoredLine -Prefix $DiskTag -Text "Tamaño total: $totalGB GB" "Gray"
    Write-ColoredLine -Prefix $DiskTag -Text "Usado: $usedGB GB" "Gray"
    Write-ColoredLine -Prefix $DiskTag -Text "Libre: $freeGB GB" "Gray"
    Write-ColoredLine -Prefix $DiskTag -Text "Porcentaje libre: $pctFree%" "Gray"
}

# PROGRESO GLOBAL
function Show-GlobalProgress {
    param(
        [int]$Step,
        [int]$TotalSteps,
        [string]$Activity
    )
    $percent = [math]::Round(($Step / $TotalSteps) * 100)
    Write-Progress -Activity $Activity -Status "$percent% completado" -PercentComplete $percent
}

# HASH 7-ZIP
function Get-7ZipHash {
    param([string]$Folder, [string]$Label)

    Write-Log "Calculando hashes: $Label" "INFO" "Yellow"

    $output = & "$SevenZipPath" h -scrcSHA256 -r "$Folder" 2>&1

    $output | ForEach-Object {
        Write-ColoredLine -Prefix "7z" -Text $_ "Blue"
    }

    return $output
}

# ROBOCOPY
function Invoke-Robocopy {
    param([string]$Source, [string]$Destination)

    Write-Log "Ejecutando ROBOCOPY..." "INFO" "Yellow"

    $cmd = "robocopy `"$Source`" `"$Destination`" /MIR /MT:32 /R:1 /W:1 /NP /TEE"
    Write-ColoredLine -Prefix "CMD" -Text $cmd "Blue"

    Write-ColoredLine -Prefix "robocopy" -Text "" "Blue"

    cmd.exe /c $cmd | ForEach-Object {
        Write-ColoredLine -Prefix "robocopy" -Text $_ "Blue"
    }
}

# COMPARACION DE INTEGRIDAD
function Get-GlobalHashesFrom7zOutput {
    param([string[]]$Lines)

    $dataHash = $null
    $nameDataHash = $null

    foreach ($line in $Lines) {

        if ($line -match "SHA256 for data:\s+([0-9a-fA-F\-]+)") {
            $dataHash = $matches[1]
        }

        if ($line -match "SHA256 for data and names:\s+([0-9a-fA-F\-]+)") {
            $nameDataHash = $matches[1]
        }
    }

    return [PSCustomObject]@{
        DataHash     = $dataHash
        NameDataHash = $nameDataHash
    }
}

function Compare-Integrity {
    param($Src, $Dst)

    Write-Log "Comparando integridad..." "INFO" "Yellow"

    if ($Src.DataHash -eq $Dst.DataHash) {
        Write-Log "OK: SHA256 for data -> MATCH" "INFO" "Green"
    }
    else {
        Write-Log "ERROR: SHA256 for data -> MISMATCH" "INFO" "Red"
    }

    if ($Src.NameDataHash -eq $Dst.NameDataHash) {
        Write-Log "OK: SHA256 for data and names -> MATCH" "INFO" "Green"
    }
    else {
        Write-Log "ERROR: SHA256 for data and names -> MISMATCH" "INFO" "Red"
    }
}

# MENU PROFESIONAL
function Show-Menu {
    Clear-Host
    Write-Host "  ███████╗██╗  ██╗███████╗███████╗██████╗ ██████╗ ██╗██████╗  " -ForegroundColor Yellow
    Write-Host "  ██╔════╝██║  ██║██╔════╝██╔════╝██╔══██╗██╔══██╗██║██╔══██╗ " -ForegroundColor Yellow
    Write-Host "  ███████╗███████║█████╗  █████╗  ██████╔╝██║  ██║██║██████╔╝ " -ForegroundColor Yellow
    Write-Host "  ╚════██║██╔══██║██╔══╝  ██╔══╝  ██╔═══╝ ██║  ██║██║██╔═══╝  " -ForegroundColor Yellow
    Write-Host "  ███████║██║  ██║███████╗███████╗██║     ██████╔╝██║██║      " -ForegroundColor Yellow
    Write-Host "  ╚══════╝╚═╝  ╚═╝╚══════╝╚══════╝╚═╝     ╚═════╝ ╚═╝╚═╝      " -ForegroundColor Yellow
    Write-Host "══════════════════════════════════════════════════════════════" -ForegroundColor Yellow
    Write-Host "              COPIA SEGURA v4.1 · SHEEPDIP SUITE              " -ForegroundColor Yellow
    Write-Host "══════════════════════════════════════════════════════════════" -ForegroundColor Yellow
    Write-Host ""
    Write-Host "  1. Proceso completo (hash origen -> copia -> hash destino -> comparacion)" -ForegroundColor Green
    Write-Host "  2. Calcular hashes del origen" -ForegroundColor White
    Write-Host "  3. Copiar solo con Robocopy" -ForegroundColor White
    Write-Host "  4. Calcular hashes del destino" -ForegroundColor White
    Write-Host "  5. Comparar integridad" -ForegroundColor White
    Write-Host "  6. Salir" -ForegroundColor Red
    Write-Host ""
}

function Start-Menu {
    do {
        Show-Menu
        $choice = Read-Host "Seleccione una opcion"

        switch ($choice) {

            "1" {
                Get-DiskInfoBlock -Path $SourceFolder -DiskTag "DISK 1"
                Get-DiskInfoBlock -Path $DestinationFolder -DiskTag "DISK 2"

                $src = Get-7ZipHash -Folder $SourceFolder -Label "SOURCE CHECKSUMS"
                Invoke-Robocopy -Source $SourceFolder -Destination $DestinationFolder
                $dst = Get-7ZipHash -Folder $DestinationFolder -Label "DESTINATION CHECKSUMS"

                $srcGlobal = Get-GlobalHashesFrom7zOutput -Lines $src
                $dstGlobal = Get-GlobalHashesFrom7zOutput -Lines $dst

                Compare-Integrity -Src $srcGlobal -Dst $dstGlobal
                # ============================
                #   FINAL TIMING
                # ============================

                $endTime = Get-Date
                $duration = $endTime - $startTime

                if ($duration.TotalSeconds -lt 60) {
                    $formatted = "{0:N2} segundos" -f $duration.TotalSeconds
                }
                elseif ($duration.TotalMinutes -lt 60) {
                    $formatted = "{0:N2} minutos" -f $duration.TotalMinutes
                }
                else {
                    $formatted = "{0:N2} horas" -f $duration.TotalHours
                }

                Write-ColoredLine -Prefix "TIMING" -Text "Tiempo total del proceso: $formatted" "Yellow"
                Write-ColoredLine -Prefix "TIMING" -Text "Finalizado correctamente." "Yellow"

                Write-Progress -Activity "Procesando..." -Completed
                Pause
            }

            "2" {
                Get-DiskInfoBlock -Path $SourceFolder -DiskTag "DISK 1"
                Get-7ZipHash -Folder $SourceFolder -Label "SOURCE CHECKSUMS"
                Pause
            }

            "3" {
                Get-DiskInfoBlock -Path $SourceFolder -DiskTag "DISK 1"
                Get-DiskInfoBlock -Path $DestinationFolder -DiskTag "DISK 2"
                Invoke-Robocopy -Source $SourceFolder -Destination $DestinationFolder
                Pause
            }

            "4" {
                Get-DiskInfoBlock -Path $DestinationFolder -DiskTag "DISK 2"
                Get-7ZipHash -Folder $DestinationFolder -Label "DESTINATION CHECKSUMS"
                Pause
            }

            "5" {
                Get-DiskInfoBlock -Path $SourceFolder -DiskTag "DISK 1"
                Get-DiskInfoBlock -Path $DestinationFolder -DiskTag "DISK 2"

                $src = Get-7ZipHash -Folder $SourceFolder -Label "SOURCE CHECKSUMS"
                $dst = Get-7ZipHash -Folder $DestinationFolder -Label "DESTINATION CHECKSUMS"

                $srcGlobal = Get-GlobalHashesFrom7zOutput -Lines $src
                $dstGlobal = Get-GlobalHashesFrom7zOutput -Lines $dst

                Compare-Integrity -Src $srcGlobal -Dst $dstGlobal
                Pause
            }

            "6" {
                Write-Host "Cerrando..." -ForegroundColor Red
                return
            }

            default {
                Write-Host "Opcion no valida." -ForegroundColor Red
                Start-Sleep -Seconds 1
            }
        }

    } while ($true)
}

# EJECUCION DEL MENU
Start-Menu
```

## copia-segura_v4.0.ps1
```md
param(
    [Parameter(Mandatory = $true)]
    [string]$Path,

    [Parameter(Mandatory = $true)]
    [string]$DestinationName,

    [Parameter(Mandatory = $false)]
    [string]$SevenZipPath = "C:\Program Files\7-Zip\7z.exe"
)

Clear-Host
Write-Host "=======================================================" -ForegroundColor Cyan
Write-Host "                 COPIA SEGURA - SUITE 4.2              " -ForegroundColor Yellow
Write-Host "=======================================================" -ForegroundColor Cyan
Write-Host ""

# OPTIMIZACION DEL PROCESO
Write-Host "Ajustando prioridad del proceso..." -ForegroundColor DarkGray
(Get-Process -Id $PID).PriorityClass = "High"

# PREPARAR RUTAS
$SourceFolder      = $Path
$DestinationFolder = $DestinationName

New-Item -ItemType Directory -Path $DestinationFolder -Force | Out-Null

$TimeStamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"

$ParentFolder = Split-Path $DestinationFolder -Parent
$LogsFolder   = Join-Path $ParentFolder "logs"
New-Item -ItemType Directory -Path $LogsFolder -Force | Out-Null

# LOG ÚNICO
$LogUnified = Join-Path $LogsFolder "copia_segura_$TimeStamp.log"

# LOGGING
function Write-Log {
    param(
        [string]$Message,
        [string]$Tag = "INFO",
        [ConsoleColor]$Color = "Gray"
    )
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $line = "[$timestamp] [$Tag] $Message"
    Write-Host $line -ForegroundColor $Color
    Add-Content -Path $LogUnified -Value $line
}

function Write-ColoredLine {
    param(
        [string]$Prefix,
        [string]$Text,
        [ConsoleColor]$Color = "Cyan"
    )
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $line = "[$timestamp] [$Prefix] $Text"
    Write-Host $line -ForegroundColor $Color
    Add-Content -Path $LogUnified -Value $line
}

# PROGRESO GLOBAL
function Show-GlobalProgress {
    param(
        [int]$Step,
        [int]$TotalSteps,
        [string]$Activity
    )
    $percent = [math]::Round(($Step / $TotalSteps) * 100)
    Write-Progress -Activity $Activity -Status "$percent% completado" -PercentComplete $percent
}

# HASH 7-ZIP
function Get-7ZipHash {
    param([string]$Folder, [string]$Label)

    Write-Log "Calculando hashes: $Label" "INFO" "Yellow"

    $output = & "$SevenZipPath" h -scrcSHA256 -r "$Folder" 2>&1

    $output | ForEach-Object {
        Write-ColoredLine -Prefix "7z" -Text $_ "Blue"
    }

    return $output
}

# ROBOCOPY
function Invoke-Robocopy {
    param([string]$Source, [string]$Destination)

    Write-Log "Ejecutando ROBOCOPY..." "INFO" "Yellow"

    $cmd = "robocopy `"$Source`" `"$Destination`" /MIR /MT:32 /R:1 /W:1 /NP /TEE"
    Write-ColoredLine -Prefix "CMD" -Text $cmd "Magenta"

    cmd.exe /c $cmd | ForEach-Object {
        Write-ColoredLine -Prefix "robocopy" -Text $_ "Blue"
    }
}

# COMPARACION DE INTEGRIDAD
function Get-GlobalHashesFrom7zOutput {
    param([string[]]$Lines)

    $dataHash      = $null
    $nameDataHash  = $null

    foreach ($line in $Lines) {

        if ($line -match "SHA256 for data:\s+([0-9a-fA-F\-]+)") {
            $dataHash = $matches[1]
        }

        if ($line -match "SHA256 for data and names:\s+([0-9a-fA-F\-]+)") {
            $nameDataHash = $matches[1]
        }
    }

    return [PSCustomObject]@{
        DataHash     = $dataHash
        NameDataHash = $nameDataHash
    }
}

function Compare-Integrity {
    param($Src, $Dst)

    Write-Log "Comparando integridad..." "INFO" "Yellow"

    if ($Src.DataHash -eq $Dst.DataHash) {
        Write-Log "OK: SHA256 for data -> MATCH" "INFO" "Green"
    } else {
        Write-Log "ERROR: SHA256 for data -> MISMATCH" "INFO" "Red"
    }

    if ($Src.NameDataHash -eq $Dst.NameDataHash) {
        Write-Log "OK: SHA256 for data and names -> MATCH" "INFO" "Green"
    } else {
        Write-Log "ERROR: SHA256 for data and names -> MISMATCH" "INFO" "Red"
    }
}

# MENU PROFESIONAL
function Show-Menu {
    Clear-Host
    Write-Host "=======================================================" -ForegroundColor Cyan
    Write-Host "                 COPIA SEGURA - SUITE 4.2              " -ForegroundColor Yellow
    Write-Host "=======================================================" -ForegroundColor Cyan
    Write-Host ""
    Write-Host "  1. Proceso completo (hash origen -> copia -> hash destino -> comparacion)" -ForegroundColor Green
    Write-Host "  2. Calcular hashes del origen" -ForegroundColor White
    Write-Host "  3. Copiar solo con Robocopy" -ForegroundColor White
    Write-Host "  4. Calcular hashes del destino" -ForegroundColor White
    Write-Host "  5. Comparar integridad" -ForegroundColor White
    Write-Host "  6. Salir" -ForegroundColor Red
    Write-Host ""
}

function Start-Menu {
    do {
        Show-Menu
        $choice = Read-Host "Seleccione una opcion"

        switch ($choice) {

            "1" {
                $src = Get-7ZipHash -Folder $SourceFolder -Label "SOURCE CHECKSUMS"
                Invoke-Robocopy -Source $SourceFolder -Destination $DestinationFolder
                $dst = Get-7ZipHash -Folder $DestinationFolder -Label "DESTINATION CHECKSUMS"

                $srcGlobal = Get-GlobalHashesFrom7zOutput -Lines $src
                $dstGlobal = Get-GlobalHashesFrom7zOutput -Lines $dst

                Compare-Integrity -Src $srcGlobal -Dst $dstGlobal
                Pause
            }

            "2" {
                Get-7ZipHash -Folder $SourceFolder -Label "SOURCE CHECKSUMS"
                Pause
            }

            "3" {
                Invoke-Robocopy -Source $SourceFolder -Destination $DestinationFolder
                Pause
            }

            "4" {
                Get-7ZipHash -Folder $DestinationFolder -Label "DESTINATION CHECKSUMS"
                Pause
            }

            "5" {
                $src = Get-7ZipHash -Folder $SourceFolder -Label "SOURCE CHECKSUMS"
                $dst = Get-7ZipHash -Folder $DestinationFolder -Label "DESTINATION CHECKSUMS"

                $srcGlobal = Get-GlobalHashesFrom7zOutput -Lines $src
                $dstGlobal = Get-GlobalHashesFrom7zOutput -Lines $dst

                Compare-Integrity -Src $srcGlobal -Dst $dstGlobal
                Pause
            }

            "6" {
                Write-Host "Cerrando..." -ForegroundColor Red
                return
            }

            default {
                Write-Host "Opcion no valida." -ForegroundColor Red
                Start-Sleep -Seconds 1
            }
        }

    } while ($true)
}

# EJECUCION DEL MENU
Start-Menu
```

## copia-segura_v3.3.ps1
```md
param(
    [Parameter(Mandatory = $true)]
    [string]$Path,

    [Parameter(Mandatory = $true)]
    [string]$DestinationName,

    [Parameter(Mandatory = $false)]
    [string]$SevenZipPath = "C:\Program Files\7-Zip\7z.exe"
)

# ============================
#   PROCESS PRIORITY
# ============================
(Get-Process -Id $PID).PriorityClass = "High"

# ============================
#   PREPARE PATHS
# ============================
$SourceFolder      = $Path
$DestinationFolder = $DestinationName

New-Item -ItemType Directory -Path $DestinationFolder -Force | Out-Null

$TimeStamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"

$ParentFolder = Split-Path $DestinationFolder -Parent
$LogsFolder   = Join-Path $ParentFolder "logs"
New-Item -ItemType Directory -Path $LogsFolder -Force | Out-Null

$LogProcess   = Join-Path $LogsFolder "process_$TimeStamp.log"
$LogChecksum  = Join-Path $LogsFolder "checksum_$TimeStamp.log"
$LogRobocopy  = Join-Path $LogsFolder "robocopy_$TimeStamp.log"

# ============================
#   FUNCTIONS
# ============================
function Write-Log {
    param([string]$Message, [ConsoleColor]$Color = [ConsoleColor]::Gray)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $line = "[$timestamp] [INFO] $Message"
    $old = $Host.UI.RawUI.ForegroundColor
    $Host.UI.RawUI.ForegroundColor = $Color
    Write-Host $line
    $Host.UI.RawUI.ForegroundColor = $old
    Add-Content -Path $LogProcess -Value $line
}

function Write-ColoredLine {
    param(
        [string]$Prefix,
        [string]$Text,
        [ConsoleColor]$Color = "Blue"
    )
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $old = $Host.UI.RawUI.ForegroundColor
    $Host.UI.RawUI.ForegroundColor = $Color
    Write-Host "[$timestamp] [$Prefix] $Text"
    $Host.UI.RawUI.ForegroundColor = $old
}

function Show-GlobalProgress {
    param(
        [int]$Step,
        [int]$TotalSteps,
        [string]$Activity
    )
    $percent = [math]::Round(($Step / $TotalSteps) * 100)
    Write-Progress -Activity $Activity -Status "$percent% completed" -PercentComplete $percent
}

function Get-GlobalHashesFrom7zOutput {
    param([string[]]$Lines)

    $dataHash      = $null
    $nameDataHash  = $null

    foreach ($line in $Lines) {

        if ($line -match 'SHA256 for data:\s+([0-9a-fA-F\-]+)') {
            $dataHash = $matches[1]
        }

        if ($line -match 'SHA256 for data and names:\s+([0-9a-fA-F\-]+)') {
            $nameDataHash = $matches[1]
        }
    }

    return [PSCustomObject]@{
        DataHash     = $dataHash
        NameDataHash = $nameDataHash
    }
}

# ============================
#   START
# ============================
$globalStopwatch = [System.Diagnostics.Stopwatch]::StartNew()

Write-Log "===== PROCESS STARTED =====" "Cyan"
Write-Log "Source: $SourceFolder"
Write-Log "Destination: $DestinationFolder"
Write-Log "7-Zip: $SevenZipPath"
Write-Log "-----------------------------------------------"

$totalSteps = 4
$currentStep = 0

# ============================
# 1. SOURCE CHECKSUMS
# ============================
$currentStep++
Show-GlobalProgress -Step $currentStep -TotalSteps $totalSteps -Activity "Processing..."

Write-Log "Calculating SOURCE checksums..." "Yellow"
$sw1 = [System.Diagnostics.Stopwatch]::StartNew()

"===== SOURCE CHECKSUMS =====" | Out-File -FilePath $LogChecksum -Encoding UTF8

$hashSrc = & "$SevenZipPath" h -scrcSHA256 -r "$SourceFolder" 2>&1

$hashSrc | ForEach-Object {
    Write-ColoredLine -Prefix "7z" -Text $_
}

$hashSrc | Out-File -FilePath $LogChecksum -Encoding UTF8 -Append

$sw1.Stop()
Write-Log ("Source hashing time: {0:N1} seconds" -f $sw1.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 2. ROBOCOPY
# ============================
$currentStep++
Show-GlobalProgress -Step $currentStep -TotalSteps $totalSteps -Activity "Processing..."

Write-Log "Copying with ROBOCOPY..." "Yellow"
$sw2 = [System.Diagnostics.Stopwatch]::StartNew()

$cmd = "robocopy `"$SourceFolder`" `"$DestinationFolder`" /MIR /MT:32 /R:1 /W:1 /NP /TEE /LOG:`"$LogRobocopy`""

Write-Log "Executing: $cmd" "Cyan"

cmd.exe /c $cmd | ForEach-Object {
    Write-ColoredLine -Prefix "robocopy" -Text $_
}

$sw2.Stop()
Write-Log ("Copy time: {0:N1} seconds" -f $sw2.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 3. DESTINATION CHECKSUMS
# ============================
$currentStep++
Show-GlobalProgress -Step $currentStep -TotalSteps $totalSteps -Activity "Processing..."

Write-Log "Calculating DESTINATION checksums..." "Yellow"
$sw3 = [System.Diagnostics.Stopwatch]::StartNew()

"===== DESTINATION CHECKSUMS =====" | Out-File -FilePath $LogChecksum -Encoding UTF8 -Append

$hashDst = & "$SevenZipPath" h -scrcSHA256 -r "$DestinationFolder" 2>&1

$hashDst | ForEach-Object {
    Write-ColoredLine -Prefix "7z" -Text $_
}

$hashDst | Out-File -FilePath $LogChecksum -Encoding UTF8 -Append

$sw3.Stop()
Write-Log ("Destination hashing time: {0:N1} seconds" -f $sw3.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
# 4. INTEGRITY COMPARISON
# ============================
$currentStep++
Show-GlobalProgress -Step $currentStep -TotalSteps $totalSteps -Activity "Processing..."

Write-Log "Comparing integrity SOURCE → DESTINATION..." "Yellow"
$sw4 = [System.Diagnostics.Stopwatch]::StartNew()

$srcGlobal = Get-GlobalHashesFrom7zOutput -Lines $hashSrc
$dstGlobal = Get-GlobalHashesFrom7zOutput -Lines $hashDst

if ($srcGlobal.DataHash -eq $dstGlobal.DataHash) {
    Write-Log "OK: Global SHA256 for data → MATCH" "Green"
} else {
    Write-Log "ERROR: Global SHA256 for data → MISMATCH" "Red"
}

if ($srcGlobal.NameDataHash -eq $dstGlobal.NameDataHash) {
    Write-Log "OK: Global SHA256 for data and names → MATCH" "Green"
} else {
    Write-Log "ERROR: Global SHA256 for data and names → MISMATCH" "Red"
}

$sw4.Stop()
Write-Log ("Integrity check time: {0:N1} seconds" -f $sw4.Elapsed.TotalSeconds)
Write-Log "-----------------------------------------------"

# ============================
#   FINAL SUMMARY
# ============================
$globalStopwatch.Stop()

# Intelligent time formatting
$elapsed = $globalStopwatch.Elapsed

if ($elapsed.TotalSeconds -lt 60) {
    $formatted = "{0:N1} seconds" -f $elapsed.TotalSeconds
}
elseif ($elapsed.TotalMinutes -lt 60) {
    $formatted = "{0} minutes {1} seconds" -f $elapsed.Minutes, $elapsed.Seconds
}
else {
    $formatted = "{0} hours {1} minutes {2} seconds" -f $elapsed.Hours, $elapsed.Minutes, $elapsed.Seconds
}

Write-Log "========== FINAL SUMMARY ==========" "Cyan"
Write-Log "Total time: $formatted"
Write-Log "Process completed successfully."
Write-Log "==================================" "Cyan"

Write-Progress -Activity "Processing..." -Completed
```

## copia-segura_v3.2.ps1
```md
param(
    [Parameter(Mandatory = $true)]
    [string]$Path,

    [Parameter(Mandatory = $true)]
    [string]$DestinationName,

    [Parameter(Mandatory = $false)]
    [string]$SevenZipPath = "C:\Program Files\7-Zip\7z.exe"
)

# ============================
#   PREPARE PATHS
# ============================
$SourceFolder      = $Path
$DestinationFolder = $DestinationName

New-Item -ItemType Directory -Path $DestinationFolder -Force | Out-Null

$TimeStamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"

$ParentFolder = Split-Path $DestinationFolder -Parent
$LogsFolder   = Join-Path $ParentFolder "logs"
New-Item -ItemType Directory -Path $LogsFolder -Force | Out-Null

$LogProcess   = Join-Path $LogsFolder "process_$TimeStamp.log"
$LogChecksum  = Join-Path $LogsFolder "checksum_$TimeStamp.log"
$LogRobocopy  = Join-Path $LogsFolder "robocopy_$TimeStamp.log"

# ============================
#   FUNCTIONS
# ============================
function Write-Log {
    param([string]$Message, [ConsoleColor]$Color = [ConsoleColor]::Gray)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $line = "[$timestamp] [INFO] $Message"
    $old = $Host.UI.RawUI.ForegroundColor
    $Host.UI.RawUI.ForegroundColor = $Color
    Write-Host $line
    $Host.UI.RawUI.ForegroundColor = $old
    Add-Content -Path $LogProcess -Value $line
}

function Write-ColoredLine {
    param(
        [string]$Prefix,
        [string]$Text,
        [ConsoleColor]$Color = "Blue"
    )
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $old = $Host.UI.RawUI.ForegroundColor
    $Host.UI.RawUI.ForegroundColor = $Color
    Write-Host "[$timestamp] [$Prefix] $Text"
    $Host.UI.RawUI.ForegroundColor = $old
}

# ============================
#   VALIDATE 7-ZIP
# ============================
if (-not (Test-Path $SevenZipPath)) {
    Write-Log "ERROR: 7z.exe not found at: $SevenZipPath" "Red"
    exit 1
}

# ============================
#   START
# ============================
Write-Log "===== PROCESS STARTED =====" "Cyan"
Write-Log "Source: $SourceFolder"
Write-Log "Destination: $DestinationFolder"
Write-Log "7-Zip: $SevenZipPath"
Write-Log "-----------------------------------------------"

# ============================
# 1. SOURCE CHECKSUMS
# ============================
Write-Log "Calculating SOURCE checksums..." "Yellow"

"===== SOURCE CHECKSUMS =====" | Out-File -FilePath $LogChecksum -Encoding UTF8

$hashSrc = & "$SevenZipPath" h -scrcSHA256 -r "$SourceFolder" 2>&1

$hashSrc | ForEach-Object {
    Write-ColoredLine -Prefix "7z" -Text $_
}

$hashSrc | Out-File -FilePath $LogChecksum -Encoding UTF8 -Append

Write-Log "Source checksums saved to checksum.log" "Green"
Write-Log "-----------------------------------------------"

# ============================
# 2. ROBOCOPY
# ============================
Write-Log "Copying with ROBOCOPY..." "Yellow"

$cmd = "robocopy `"$SourceFolder`" `"$DestinationFolder`" /MIR /MT:32 /R:1 /W:1 /NP /TEE /LOG:`"$LogRobocopy`""

Write-Log "Executing: $cmd" "Cyan"

cmd.exe /c $cmd | ForEach-Object {
    Write-ColoredLine -Prefix "robocopy" -Text $_
}

Write-Log "Robocopy completed." "Green"
Write-Log "-----------------------------------------------"

# ============================
# 3. DESTINATION CHECKSUMS
# ============================
Write-Log "Calculating DESTINATION checksums..." "Yellow"

"===== DESTINATION CHECKSUMS =====" | Out-File -FilePath $LogChecksum -Encoding UTF8 -Append

$hashDst = & "$SevenZipPath" h -scrcSHA256 -r "$DestinationFolder" 2>&1

$hashDst | ForEach-Object {
    Write-ColoredLine -Prefix "7z" -Text $_
}

$hashDst | Out-File -FilePath $LogChecksum -Encoding UTF8 -Append

Write-Log "Destination checksums saved to checksum.log" "Green"
Write-Log "-----------------------------------------------"

# ============================
#   END
# ============================
Write-Log "===== PROCESS COMPLETED =====" "Cyan"
```

## targets.json
```md
{
  "OCM": [
    { "Name": "Servidor Web OCM",   "Host": "192.168.1.10", "Ports": [80, 443] },
    { "Name": "Control OCM 01",     "Host": "srv-ocm01",    "Ports": [3389] }
  ],
  "CORE": [
    { "Name": "Core Principal",     "Host": "10.0.0.5",     "Ports": [22, 443, 8080] }
  ],
  "KIOSK": [
    { "Name": "Kiosko 01",          "Host": "kiosk-01",     "Ports": [80] },
    { "Name": "Kiosko 02",          "Host": "kiosk-02",     "Ports": [80, 443] }
  ],
  "OTROS": [
    { "Name": "Google Public",      "Host": "google.com",   "Ports": [443] }
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
Write-Host "                 Version 1.3 - by Juan                 " -ForegroundColor Yellow
Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
Write-Host ""

# ============================
#   RUTA DEL SCRIPT + LOGS
# ============================
$ScriptPath = Split-Path -Parent $MyInvocation.MyCommand.Path
$LogFolder  = Join-Path $ScriptPath "logs"
New-Item -ItemType Directory -Path $LogFolder -Force | Out-Null

$Hostname  = $env:COMPUTERNAME
$TimeStamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"
$LogFile   = Join-Path $LogFolder "connectivity_${Hostname}_${TimeStamp}.log"

# ============================
#   LOG ENTERPRISE
# ============================
function Write-Log {
    param(
        [string]$Message,
        [string]$Tag = "INFO",
        [ConsoleColor]$Color = "Gray"
    )

    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $tagPadded = $Tag.ToUpper().PadRight(10)
    $line = "[$timestamp] [$tagPadded] $Message"

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

Write-Log "IP local detectada: $MyIP" "INFO" "Cyan"
Write-Host ""

# ============================
#   CARGAR CONFIGURACIÓN JSON
# ============================
if (-not (Test-Path $ConfigPath)) {
    Write-Log "No se encuentra el archivo JSON: $ConfigPath" "ERROR" "Red"
    exit 1
}

try {
    $jsonRaw = Get-Content -Path $ConfigPath -Raw
    $Targets = $jsonRaw | ConvertFrom-Json
}
catch {
    Write-Log "ERROR al leer o parsear el JSON: $($_.Exception.Message)" "ERROR" "Red"
    exit 1
}

Write-Log "Cargando configuración desde: $ConfigPath" "INFO" "Cyan"
Write-Log "Grupos definidos: $($Targets.PSObject.Properties.Name -join ', ')" "INFO"
Write-Log "------------------------------------------------------------" "INFO"

# ============================
#   LISTAR OBJETIVOS
# ============================
function Show-Targets {
    Write-Host ""
    Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
    Write-Host "                 LISTA DE OBJETIVOS (JSON)             " -ForegroundColor Yellow
    Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

    Write-Log "Mostrando lista de objetivos desde JSON." "INFO"

    foreach ($groupProp in $Targets.PSObject.Properties) {
        $groupName = $groupProp.Name
        $items     = $groupProp.Value

        Write-Host ""
        Write-Host "Grupo: $groupName" -ForegroundColor Cyan
        Write-Log  "Grupo: $groupName" "INFO"

        foreach ($item in $items) {
            $ports = ($item.Ports -join ",")
            Write-Host " - Equipo: $($item.Name)  Host: $($item.Host)  Puertos: $ports"
            Write-Log  " - Equipo: $($item.Name)  Host: $($item.Host)  Puertos: $ports" "INFO"
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
                Name  = $item.Name
                Host  = $item.Host
                Ports = $item.Ports
            }
        }
    }

    if ($allItems.Count -eq 0) {
        Write-Log "No hay objetivos definidos en el JSON." "WARN" "Yellow"
        return
    }

    foreach ($entry in $allItems) {

        Write-Host ""
        Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
        Write-Host ("GRUPO: {0}  •  EQUIPO: {1}  •  HOST: {2}" -f $entry.Group, $entry.Name, $entry.Host) -ForegroundColor Yellow
        Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan

        Write-Log "Grupo: $($entry.Group)  •  Equipo: $($entry.Name)  •  Host: $($entry.Host)" "NETWORK" "Blue"

        # PING
        $pingResult = Test-NetConnection -ComputerName $entry.Host -WarningAction SilentlyContinue
        if ($pingResult.PingSucceeded) {
            Write-Log "PING OK → $($entry.Host)" "OK" "Green"
        }
        else {
            Write-Log "PING FALLÓ → $($entry.Host)" "ERROR" "Red"
        }

        # PUERTOS
        foreach ($port in $entry.Ports) {
            $tcpResult = Test-NetConnection -ComputerName $entry.Host -Port $port -WarningAction SilentlyContinue
            if ($tcpResult.TcpTestSucceeded) {
                Write-Log "TCP OK (puerto $port)" "OK" "Green"
            }
            else {
                Write-Log "TCP FALLÓ (puerto $port)" "ERROR" "Red"
            }
        }

        Write-Log "------------------------------------------------------------" "INFO"
    }

    Write-Log "Pruebas de red completadas." "SUCCESS" "DarkGreen"
}

# ============================
#   MENÚ PROFESIONAL
# ============================
function Show-Menu {
    Clear-Host
    Write-Host "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" -ForegroundColor Cyan
    Write-Host "                     NET TEST SUITE                    " -ForegroundColor Yellow
    Write-Host "                 Version 1.3 - by Juan                 " -ForegroundColor Yellow
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
                Write-Log "Cierre de la herramienta solicitado por el usuario." "INFO" "Yellow"
                Write-Host "Cerrando Net Test Suite..." -ForegroundColor Red
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

```md
param(
    [Parameter(Mandatory = $true)]
    [string]$RutasTXT
)

# Carpeta donde está el script
$ScriptFolder = Split-Path -Parent $MyInvocation.MyCommand.Path

# Nombre del log
$HostName  = $env:COMPUTERNAME
$TimeStamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"
$LogFile   = Join-Path $ScriptFolder "Reparse_${HostName}_$TimeStamp.log"

Write-Host "Log generado en: $LogFile"
Write-Host ""

# Función para escribir en log
function Write-Log {
    param([string]$Message)
    Add-Content -Path $LogFile -Value $Message
}

# Validar TXT
if (-not (Test-Path $RutasTXT)) {
    Write-Host "ERROR: No se encuentra el archivo TXT."
    exit 1
}

$Rutas = Get-Content -Path $RutasTXT | Where-Object { $_.Trim() -ne "" }

# Procesar cada ruta
foreach ($Ruta in $Rutas) {

    Write-Host "Consultando: $Ruta"
    Write-Log  "Consultando: $Ruta"

    # Ejecutar comando
    $Salida = fsutil reparsepoint query "$Ruta" 2>&1

    # Mostrar por pantalla
    $Salida | ForEach-Object { Write-Host $_ }

    # Guardar en log
    $Salida | ForEach-Object { Write-Log $_ }

    Write-Log "------------------------------------------------------------"
    Write-Host ""
}

Write-Host "Proceso completado."

```




