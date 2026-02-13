# Test_Ciber
```md

$LetrasUSB = @()   # Array donde guardaremos las letras USB

(
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

                # Guardar la letra en el array
                $LetrasUSB += $vol.DriveLetter

                [PSCustomObject]@{
                    Nombre        = if ($vol.FileSystemLabel) { $vol.FileSystemLabel } else { "(Sin nombre)" }
                    Letra         = $vol.DriveLetter
                    'Libre (GB)'  = "{0:N2}" -f ($vol.SizeRemaining / 1GB)
                    'Usado (GB)'  = "{0:N2}" -f (($vol.Size - $vol.SizeRemaining) / 1GB)
                    'Total (GB)'  = "{0:N2}" -f ($vol.Size / 1GB)
                    'Nº Serie'    = if ($serial) { $serial.Trim() } else { "(No disponible)" }
                }
            }
    }
) | Format-Table -AutoSize


# Mostrar contenido de cada letra USB
foreach ($letra in $LetrasUSB) {
    Write-Host "`n=== Contenido de $letra`: ===" -ForegroundColor Cyan
    Get-ChildItem "$letra`:\" -Directory | Select-Object Name
}
