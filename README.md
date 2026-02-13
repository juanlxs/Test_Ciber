# Test_Ciber
```md
# ================================
#   TÍTULO ESTILO CUADRO PRO
# ================================

$titulo = "CONFIGURACIÓN DEL PROCESO"
$max = $titulo.Length

Write-Host ""
Write-Host ("╔" + ("═" * ($max + 2)) + "╗")
Write-Host ("║ " + $titulo.PadLeft((($max + $titulo.Length)/2)).PadRight($max) + " ║")
Write-Host ("╚" + ("═" * ($max + 2)) + "╝")
Write-Host ""

# ================================
#   ENTRADAS PROFESIONALES
# ================================

$usuario  = Read-Host -Prompt ("{0,-25}" -f "Usuario")
$proyecto = Read-Host -Prompt ("{0,-25}" -f "Proyecto")
$version  = Read-Host -Prompt ("{0,-25}" -f "Versión")
$codigo   = Read-Host -Prompt ("{0,-25}" -f "Código interno")
