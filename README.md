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

# 2. Seguridad
$acl = Get-Acl $Path

# 3. Propiedades extendidas básicas
$extended = Get-ItemProperty -Path $Path -ErrorAction SilentlyContinue

# 4. Construcción del objeto final
$result = [PSCustomObject]@{
    Ruta                  = $general.FullName
    Nombre                = $general.Name
    Extension             = $general.Extension
    TamanoBytes           = $general.Length
    FechaCreacion         = $general.CreationTime
    FechaModificacion     = $general.LastWriteTime
    FechaAcceso           = $general.LastAccessTime
    Atributos             = $general.Attributes
    Seguridad_ACL         = $acl.Access
    Seguridad_Propietario = $acl.Owner
    Seguridad_Grupo       = $acl.Group
    PropiedadesExtendidas = $extended
}

# 5. Salida
$result
