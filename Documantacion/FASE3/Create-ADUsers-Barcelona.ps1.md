Import-Module ActiveDirectory
$ErrorActionPreference = "Stop"

# CONFIGURACIÓN
$Dominio = "barcelona.lan"
$BaseOU = "OU=Barcelona,DC=barcelona,DC=lan"
$CSVPath = "$PSScriptRoot\usuaris.csv"
$PasswordPlano = "barcelona@1"
$PasswordSegura = ConvertTo-SecureString $PasswordPlano -AsPlainText -Force

# Grupo de jefes existente en tu AD
$GrupoJefes = "GG Caps Departament"

# Mapeo CSV -> OU y grupo real de AD
$MapaDepartamentos = @{
    "Gerencia"       = @{ OU="Gerencia";        Grupo="GG_Seguridad_Gerencia" }
    "Administración" = @{ OU="Administrativo";  Grupo="GG_Seguridad_Administrativo" }
    "Administracio"  = @{ OU="Administrativo";  Grupo="GG_Seguridad_Administrativo" }
    "SAT"            = @{ OU="SAT";             Grupo="GG_Seguridad_SAT" }
    "Vigilancia"     = @{ OU="Vigilancia";      Grupo="GG_Seguridad_Vigilancia" }
    "Vigilància"     = @{ OU="Vigilancia";      Grupo="GG_Seguridad_Vigilancia" }
    "Comercial"      = @{ OU="Comerciales";     Grupo="GG_Seguridad_Comerciales" }
    "Comerciales"    = @{ OU="Comerciales";     Grupo="GG_Seguridad_Comerciales" }
    "Producción"     = @{ OU="Produccion";      Grupo="GG_Seguridad_Produccion" }
    "Produccion"     = @{ OU="Produccion";      Grupo="GG_Seguridad_Produccion" }
    "Almacén"        = @{ OU="Almacenamiento";  Grupo="GG_Seguridad_Almacenamiento" }
    "Almacen"        = @{ OU="Almacenamiento";  Grupo="GG_Seguridad_Almacenamiento" }
    "Magatzem"       = @{ OU="Almacenamiento";  Grupo="GG_Seguridad_Almacenamiento" }
    "Recepción"      = @{ OU="Recepcion";       Grupo="GG_Seguridad_Recepcion" }
    "Recepcion"      = @{ OU="Recepcion";       Grupo="GG_Seguridad_Recepcion" }
}

function Limpiar-Texto {
    param([string]$Texto)

    if ([string]::IsNullOrWhiteSpace($Texto)) {
        return ""
    }

    $Texto = $Texto.ToLower()
    $Texto = $Texto.Normalize([Text.NormalizationForm]::FormD)
    $Texto = [regex]::Replace($Texto, '\p{Mn}', '')
    $Texto = $Texto -replace 'ñ','n'
    $Texto = $Texto -replace 'ç','c'
    $Texto = $Texto -replace '[^a-z0-9]', ''

    return $Texto
}

function Get-LoginUnico {
    param(
        [string]$Nombre,
        [string]$Apellido
    )

    $NombreLimpio = Limpiar-Texto $Nombre
    $ApellidoLimpio = Limpiar-Texto $Apellido

    $LoginBase = "$NombreLimpio.$ApellidoLimpio"

    if ($LoginBase.Length -gt 18) {
        $LoginBase = $LoginBase.Substring(0,18)
    }

    $Login = $LoginBase
    $Contador = 1

    while (Get-ADUser -Filter "SamAccountName -eq '$Login'" -ErrorAction SilentlyContinue) {
        $Login = "$LoginBase$Contador"
        $Contador++
    }

    return $Login
}

function Add-UsuarioAGrupo {
    param(
        [string]$UsuarioSam,
        [string]$Grupo
    )

    $ExisteGrupo = Get-ADGroup -Filter "Name -eq '$Grupo'" -ErrorAction SilentlyContinue

    if ($ExisteGrupo) {
        Add-ADGroupMember -Identity $Grupo -Members $UsuarioSam -ErrorAction SilentlyContinue
        Write-Host "  Añadido al grupo: $Grupo" -ForegroundColor Cyan
    }
    else {
        Write-Host "  AVISO: No existe el grupo $Grupo" -ForegroundColor Yellow
    }
}

if (-not (Test-Path $CSVPath)) {
    Write-Host "ERROR: No se encuentra el CSV en $CSVPath" -ForegroundColor Red
    exit
}

$Usuarios = Import-Csv -Path $CSVPath

foreach ($Usuario in $Usuarios) {

    if ($Usuario.sede -ne "Barcelona") {
        continue
    }

    $DepartamentoCSV = $Usuario.dept.Trim()

    if (-not $MapaDepartamentos.ContainsKey($DepartamentoCSV)) {
        Write-Host "AVISO: Departamento no reconocido: $DepartamentoCSV. Usuario omitido: $($Usuario.nom) $($Usuario.cognom1)" -ForegroundColor Yellow
        continue
    }

    $OUDept = $MapaDepartamentos[$DepartamentoCSV].OU
    $GrupoDept = $MapaDepartamentos[$DepartamentoCSV].Grupo
    $RutaOU = "OU=$OUDept,$BaseOU"

    $ExisteOU = Get-ADOrganizationalUnit -LDAPFilter "(distinguishedName=$RutaOU)" -ErrorAction SilentlyContinue

    if (-not $ExisteOU) {
        Write-Host "ERROR: No existe la OU $RutaOU. Usuario omitido." -ForegroundColor Red
        continue
    }

    $NombreCompleto = "$($Usuario.nom) $($Usuario.cognom1) $($Usuario.cognom2)"
    $Login = Get-LoginUnico -Nombre $Usuario.nom -Apellido $Usuario.cognom1
    $UPN = "$Login@$Dominio"

    try {
        New-ADUser `
            -Name $NombreCompleto `
            -GivenName $Usuario.nom `
            -Surname "$($Usuario.cognom1) $($Usuario.cognom2)" `
            -SamAccountName $Login `
            -UserPrincipalName $UPN `
            -DisplayName $NombreCompleto `
            -Path $RutaOU `
            -Department $DepartamentoCSV `
            -Description $Usuario.descrip `
            -EmployeeID $Usuario.dni `
            -Office $Usuario.sede `
            -AccountPassword $PasswordSegura `
            -Enabled $true `
            -ChangePasswordAtLogon $false `
            -PasswordNeverExpires $false `
            -ErrorAction Stop

        Write-Host "Usuario creado: $Login en $OUDept" -ForegroundColor Green

        Add-UsuarioAGrupo -UsuarioSam $Login -Grupo $GrupoDept

        if ($Usuario.descrip -eq "Jefe") {
            Add-UsuarioAGrupo -UsuarioSam $Login -Grupo $GrupoJefes
        }
    }
    catch {
        Write-Host "ERROR creando usuario $NombreCompleto : $($_.Exception.Message)" -ForegroundColor Red
    }
}
