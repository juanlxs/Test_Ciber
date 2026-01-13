# Test_Ciber
```md
<#
.SYNOPSIS
    Extrae TODAS las propiedades posibles de un fichero:
    - Generales (Get-Item)
    - Seguridad (Get-Acl)
    - Detalles extendidos (Shell.Application)
    Devuelve un objeto unificado por fichero.
#>

param(
    [Parameter(Mandatory=$true)]
    [string]$Path
)

# Validación
if (-not (Test-Path $Path)) {
    Write-Error "La ruta '$Path' no existe."
    exit
}

# --- 1. Propiedades generales ---
$general = Get-Item $Path

# --- 2. Propiedades de seguridad ---
$acl = Get-Acl $Path

# --- 3. Propiedades extendidas (Detalles) ---
$shell = New-Object -ComObject Shell.Application
$folder = $shell.Namespace((Split-Path $Path))
$file   = $folder.ParseName((Split-Path $Path -Leaf))

$extendedProps = @{}

0..400 | ForEach-Object {
    $name  = $folder.GetDetailsOf($folder.Items, $_)
    $value = $folder.GetDetailsOf($file, $_)

    if ($name -and $value) {
        $extendedProps[$name] = $value
    }
}

# --- 4. Construcción del objeto final ---
$result = [PSCustomObject]@{
    Ruta                = $general.FullName
    Nombre              = $general.Name
    Extension           = $general.Extension
    TamanoBytes         = $general.Length
    FechaCreacion       = $general.CreationTime
    FechaModificacion   = $general.LastWriteTime
    FechaAcceso         = $general.LastAccessTime
    Atributos           = $general.Attributes
    Seguridad_ACL       = $acl.Access
    Seguridad_Propietario = $acl.Owner
    Seguridad_Grupo       = $acl.Group
    PropiedadesExtendidas = $extendedProps
}

# --- 5. Salida ---
$result
