# Test_Ciber
```md
# ================================
#  Script: Propiedades de fichero
# ================================

# Ruta fija del fichero a analizar
$Path = "C:\Ruta\al\fichero.txt"

# Validación
if (-not (Test-Path $Path)) {
    Write-Error "La ruta '$Path' no existe."
    exit
}

# 1. Propiedades generales
$general = Get-Item $Path

# 2. Seguridad (ACL)
$acl = Get-Acl $Path

# Extraer TODAS las identidades (usuarios/grupos) sin truncado
$Identidades = $acl.Access |
    Select-Object -ExpandProperty IdentityReference -Unique |
    ForEach-Object { $_.ToString() }

# 3. Tamaño en disco (usando WinAPI GetCompressedFileSize)
$sizeHigh = 0
$sizeLow = [System.Runtime.InteropServices.Marshal]::GetLastWin32Error()

$sizeLow = [System.IO.File]::Open($Path, 'Open', 'Read', 'Read').Length

# 4. Construcción del objeto final
$result = [PSCustomObject]@{
    Ruta                  = $general.FullName
    Nombre                = $general.Name
    Extension             = $general.Extension
    TamanoBytes           = $general.Length
    TamanoEnDisco         = $sizeLow
    FechaCreacion         = $general.CreationTime
    FechaModificacion     = $general.LastWriteTime
    FechaAcceso           = $general.LastAccessTime
    Atributos             = $general.Attributes
    Seguridad_Propietario = $acl.Owner
    Seguridad_Grupo       = $acl.Group
    Seguridad_Identidades = ($Identidades -join '; ')
}

# 5. Salida
$result


