# Test_Ciber
```md
# Ruta del archivo con la lista
$listFile = "list.txt"

# Archivo de log para errores
$errorLog = "errores.log"

# Limpiar log anterior
Clear-Content $errorLog -ErrorAction SilentlyContinue

# Procesar cada línea
Get-Content $listFile | ForEach-Object {

    $ruta = $_.Trim()
    if ($ruta -eq "") { return }

    # Añadir unidad F:
    $origen = "F:$ruta"

    # Ejecutar robocopy en modo backup (/b)
    # Ejemplo copiando a un destino fijo (ajusta según necesites)
    $destino = "F:\destino"

    robocopy $origen $destino /b /r:0 /w:0 /np /njh /njs /ndl /nc /ns 2>> $errorLog

    if ($LASTEXITCODE -ge 8) {
        Add-Content $errorLog "Error copiando: $origen (Código $LASTEXITCODE)"
    }
}
