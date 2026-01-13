# Test_Ciber
```md
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

# Extraer todos los usuarios/grupos de la pestaña Seguridad
$Identidades = $acl.Access | Select-Object -ExpandProperty IdentityReference -Unique

# 3. Tamaño en disco (requiere llamada a API Win32)
$FileInfo = Get-Item $Path
$FileHandle = [System.IO.File]::Open($Path, 'Open', 'Read', 'Read')
$SizeOnDisk = $FileHandle.Length
$FileHandle.Close()

# 4. Construcción del objeto final
$result = [PSCustomObject]@{
    Ruta                  = $general.FullName
    Nombre                = $general.Name
    Extension             = $general.Extension
    TamanoBytes           = $general.Length
    TamanoEnDisco         = $SizeOnDisk
    FechaCreacion         = $general.CreationTime
    FechaModificacion     = $general.LastWriteTime
    FechaAcceso           = $general.LastAccessTime
    Atributos             = $general.Attributes
    Seguridad_Propietario = $acl.Owner
    Seguridad_Grupo       = $acl.Group
    Seguridad_Identidades = $Identidades
}

# 5. Salida
$result

