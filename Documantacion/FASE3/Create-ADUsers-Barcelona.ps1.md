Import-Module ActiveDirectory

$Dominio = "barcelona.lan"
$CSVPath = "$PSScriptRoot\usuaris.csv"
$PasswordPlano = "barcelona@1"
$PasswordSegura = ConvertTo-SecureString $PasswordPlano -AsPlainText -Force

$OUUsuarios = "OU=Usuarios,DC=barcelona,DC=lan"
$GrupoJefes = "JefesDepartamento"

function Limpiar-Texto {
    param([string]$Texto)

    if ([string]::IsNullOrWhiteSpace($Texto)) {
        return ""
    }

    $Texto = $Texto.ToLower()
    $Texto = $Texto.Normalize([Text.NormalizationForm]::FormD)
    $Texto = [regex]::Replace($Texto, '\p{Mn}', '')
    $Texto = $Texto -replace 'ñ', 'n'
    $Texto = $Texto -replace 'ç', 'c'
    $Texto = $Texto -replace '[^a-z0-9]', ''

    return $Texto
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

    $NombreLimpio = Limpiar-Texto $Usuario.nom
    $ApellidoLimpio = Limpiar-Texto $Usuario.cognom1

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

    $NombreCompleto = "$($Usuario.nom) $($Usuario.cognom1) $($Usuario.cognom2)"
    $UPN = "$Login@$Dominio"

    New-ADUser `
        -Name $NombreCompleto `
        -GivenName $Usuario.nom `
        -Surname "$($Usuario.cognom1) $($Usuario.cognom2)" `
        -SamAccountName $Login `
        -UserPrincipalName $UPN `
        -DisplayName $NombreCompleto `
        -Path $OUUsuarios `
        -Department $Usuario.dept `
        -Description $Usuario.descrip `
        -EmployeeID $Usuario.dni `
        -Office $Usuario.sede `
        -AccountPassword $PasswordSegura `
        -Enabled $true `
        -ChangePasswordAtLogon $false `
        -PasswordNeverExpires $false

    Write-Host "Usuario creado: $Login" -ForegroundColor Green

    $GrupoDept = Limpiar-Texto $Usuario.dept

    if (Get-ADGroup -Filter "Name -eq '$GrupoDept'" -ErrorAction SilentlyContinue) {
        Add-ADGroupMember -Identity $GrupoDept -Members $Login
        Write-Host "  Añadido al grupo: $GrupoDept"
    }
    else {
        Write-Host "  AVISO: No existe el grupo $GrupoDept" -ForegroundColor Yellow
    }

    if ($Usuario.descrip -eq "Jefe") {
        if (Get-ADGroup -Filter "Name -eq '$GrupoJefes'" -ErrorAction SilentlyContinue) {
            Add-ADGroupMember -Identity $GrupoJefes -Members $Login
            Write-Host "  Añadido al grupo: $GrupoJefes"
        }
        else {
            Write-Host "  AVISO: No existe el grupo $GrupoJefes" -ForegroundColor Yellow
        }
    }
}
