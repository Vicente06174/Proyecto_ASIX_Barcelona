Import-Module ActiveDirectory

$Dominio = "barcelona.lan"
$CSVPath = "C:\Scripts\usuaris.csv"
$PasswordPlano = "barcelona@1"
$PasswordSegura = ConvertTo-SecureString $PasswordPlano -AsPlainText -Force

$OUUsuarios = "OU=Usuarios,DC=barcelona,DC=lan"

function Limpiar-Texto {
    param([string]$Texto)

    $Texto = $Texto.ToLower()
    $Texto = $Texto.Normalize([Text.NormalizationForm]::FormD)
    $Texto = [regex]::Replace($Texto, '\p{Mn}', '')
    $Texto = $Texto -replace '[^a-z0-9]', ''

    return $Texto
}

$Usuarios = Import-Csv $CSVPath

foreach ($Usuario in $Usuarios) {

    if ($Usuario.sede -ne "Barcelona") {
        continue
    }

    $NombreLimpio = Limpiar-Texto $Usuario.nombre
    $ApellidoLimpio = Limpiar-Texto $Usuario.cognoms.Split(" ")[0]

    $LoginBase = "$NombreLimpio.$ApellidoLimpio"
    $Login = $LoginBase
    $Contador = 1

    while (Get-ADUser -Filter "SamAccountName -eq '$Login'" -ErrorAction SilentlyContinue) {
        $Login = "$LoginBase$Contador"
        $Contador++
    }

    New-ADUser `
        -Name "$($Usuario.nombre) $($Usuario.cognoms)" `
        -GivenName $Usuario.nombre `
        -Surname $Usuario.cognoms `
        -SamAccountName $Login `
        -UserPrincipalName "$Login@$Dominio" `
        -DisplayName "$($Usuario.nombre) $($Usuario.cognoms)" `
        -Path $OUUsuarios `
        -Department $Usuario.departament `
        -Description $Usuario.descrip `
        -AccountPassword $PasswordSegura `
        -Enabled $true `
        -ChangePasswordAtLogon $false `
        -PasswordNeverExpires $false

    Write-Host "Usuario creado: $Login"

    $GrupoDept = $Usuario.departament -replace " ", ""

    if (Get-ADGroup -Filter "Name -eq '$GrupoDept'" -ErrorAction SilentlyContinue) {
        Add-ADGroupMember -Identity $GrupoDept -Members $Login
        Write-Host "  Añadido a grupo: $GrupoDept"
    }

    if ($Usuario.descrip -eq "Jefe") {
        Add-ADGroupMember -Identity "JefesDepartamento" -Members $Login
        Write-Host "  Añadido a grupo: JefesDepartamento"
    }
}
