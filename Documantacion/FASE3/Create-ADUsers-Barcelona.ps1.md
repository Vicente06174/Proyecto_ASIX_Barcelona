\[CmdletBinding()\]  
param(  
    \[Parameter(Mandatory \= $false)\]  
    \[string\]$CsvPath \= ".\\usuaris.csv",

    \[Parameter(Mandatory \= $false)\]  
    \[string\]$DomainFqdn \= "barcelona.lan",

    \[Parameter(Mandatory \= $false)\]  
    \[string\]$SiteName \= "Barcelona",

    \[Parameter(Mandatory \= $false)\]  
    \[string\]$BaseOU \= "OU=Usuaris,DC=barcelona,DC=lan",

    \[Parameter(Mandatory \= $false)\]  
    \[string\]$DefaultPassword \= "barcelona@1",

    \[Parameter(Mandatory \= $false)\]  
    \[string\]$GroupPrefix \= "GG\_",

    \[Parameter(Mandatory \= $false)\]  
    \[string\]$HeadsGroupName \= "GG\_JefesDepartamento",

    \[Parameter(Mandatory \= $false)\]  
    \[switch\]$CreateMissingOUs \= $true,

    \[Parameter(Mandatory \= $false)\]  
    \[switch\]$CreateMissingGroups \= $true,

    \[Parameter(Mandatory \= $false)\]  
    \[bool\]$EnableUsers \= $true,

    \[Parameter(Mandatory \= $false)\]  
    \[bool\]$ForcePasswordChangeAtLogon \= $false,

    \[Parameter(Mandatory \= $false)\]  
    \[string\]$UpnSuffix \= "barcelona.lan",

    \[Parameter(Mandatory \= $false)\]  
    \[string\]$MailDomain \= "barcelona.lan",

    \[Parameter(Mandatory \= $false)\]  
    \[switch\]$WhatIfMode,

    \[Parameter(Mandatory \= $false)\]  
    \[string\]$LogPath \= ".\\Create-ADUsers-Barcelona-log.csv"  
)

Set-StrictMode \-Version Latest  
$ErrorActionPreference \= 'Stop'

function Remove-Diacritics {  
    param(\[string\]$Text)

    if (\[string\]::IsNullOrWhiteSpace($Text)) { return "" }

    $normalized \= $Text.Normalize(\[Text.NormalizationForm\]::FormD)  
    $sb \= New-Object System.Text.StringBuilder

    foreach ($char in $normalized.ToCharArray()) {  
        $unicodeCategory \= \[Globalization.CharUnicodeInfo\]::GetUnicodeCategory($char)  
        if ($unicodeCategory \-ne \[Globalization.UnicodeCategory\]::NonSpacingMark) {  
            \[void\]$sb.Append($char)  
        }  
    }

    return $sb.ToString().Normalize(\[Text.NormalizationForm\]::FormC)  
}

function Normalize-Token {  
    param(\[string\]$Text)

    $clean \= Remove-Diacritics \-Text ($Text ?? "")  
    $clean \= $clean.ToLowerInvariant().Trim()  
    $clean \= $clean \-replace '\[^a-z0-9 \]', ' '  
    $clean \= $clean \-replace '\\s+', ' '  
    return $clean.Trim()  
}

function Normalize-NameNoSpaces {  
    param(\[string\]$Text)  
    return (Normalize-Token \-Text $Text) \-replace ' ', ''  
}

function Normalize-GroupPart {  
    param(\[string\]$Text)

    $value \= Remove-Diacritics \-Text ($Text ?? "")  
    $value \= $value.Trim()  
    $value \= $value \-replace '\[^A-Za-z0-9 \]', ' '  
    $value \= $value \-replace '\\s+', '\_'  
    return $value.Trim('\_')  
}

function Ensure-OU {  
    param(\[string\]$DistinguishedName)

    try {  
        Get-ADOrganizationalUnit \-Identity $DistinguishedName \-ErrorAction Stop | Out-Null  
        return  
    }  
    catch {  
        if (-not $CreateMissingOUs) {  
            throw "La OU no existe y CreateMissingOUs está desactivado: $DistinguishedName"  
        }  
    }

    $parts \= $DistinguishedName \-split ','  
    $dcParts \= @($parts | Where-Object { $\_ \-match '^DC=' })  
    $ouParts \= @($parts | Where-Object { $\_ \-match '^OU=' })

    if ($dcParts.Count \-eq 0\) {  
        throw "DN inválido para OU: $DistinguishedName"  
    }

    $currentPath \= ($dcParts \-join ',')

    for ($i \= $ouParts.Count \- 1; $i \-ge 0; $i--) {  
        $ouName \= ($ouParts\[$i\] \-replace '^OU=', '')  
        $candidateDn \= "OU=$ouName,$currentPath"

        try {  
            Get-ADOrganizationalUnit \-Identity $candidateDn \-ErrorAction Stop | Out-Null  
        }  
        catch {  
            Write-Host "\[INFO\] Creando OU: $candidateDn"  
            if (-not $WhatIfMode) {  
                New-ADOrganizationalUnit \-Name $ouName \-Path $currentPath \-ProtectedFromAccidentalDeletion $false | Out-Null  
            }  
        }

        $currentPath \= $candidateDn  
    }  
}

function Ensure-Group {  
    param(  
        \[string\]$Name,  
        \[string\]$Path  
    )

    $group \= Get-ADGroup \-Filter "Name \-eq '$Name'" \-ErrorAction SilentlyContinue  
    if ($group) { return $group }

    if (-not $CreateMissingGroups) {  
        throw "El grupo no existe y CreateMissingGroups está desactivado: $Name"  
    }

    Write-Host "\[INFO\] Creando grupo: $Name"  
    if (-not $WhatIfMode) {  
        return New-ADGroup \-Name $Name \-SamAccountName $Name \-GroupCategory Security \-GroupScope Global \-Path $Path \-DisplayName $Name \-Description "Grupo creado automáticamente para importación de usuarios"  
    }

    return \[pscustomobject\]@{ Name \= $Name; DistinguishedName \= "CN=$Name,$Path" }  
}

function Get-UniqueSamAccountName {  
    param(  
        \[string\]$GivenName,  
        \[string\]$Surname,  
        \[string\]$Dni  
    )

    $baseGiven \= Normalize-NameNoSpaces \-Text $GivenName  
    $baseSurname \= Normalize-NameNoSpaces \-Text $Surname

    if (\[string\]::IsNullOrWhiteSpace($baseGiven) \-or \[string\]::IsNullOrWhiteSpace($baseSurname)) {  
        throw "No se puede generar el samAccountName sin nombre y primer apellido"  
    }

    $candidate \= "$baseGiven.$baseSurname"  
    if ($candidate.Length \-gt 20\) {  
        $candidate \= $candidate.Substring(0, 20\)  
    }

    $exists \= Get-ADUser \-Filter "SamAccountName \-eq '$candidate'" \-ErrorAction SilentlyContinue  
    if (-not $exists) {  
        return $candidate  
    }

    $dniClean \= (Normalize-NameNoSpaces \-Text $Dni)  
    if ($dniClean.Length \-ge 4\) {  
        $suffix \= $dniClean.Substring($dniClean.Length \- 4\)  
    }  
    elseif ($dniClean.Length \-gt 0\) {  
        $suffix \= $dniClean  
    }  
    else {  
        $suffix \= '0001'  
    }

    $baseLength \= \[Math\]::Min(20 \- $suffix.Length, $candidate.Length)  
    $candidate2 \= $candidate.Substring(0, $baseLength) \+ $suffix  
    $exists2 \= Get-ADUser \-Filter "SamAccountName \-eq '$candidate2'" \-ErrorAction SilentlyContinue  
    if (-not $exists2) {  
        return $candidate2  
    }

    for ($i \= 2; $i \-le 999; $i++) {  
        $num \= "$i"  
        $baseLength \= \[Math\]::Min(20 \- $num.Length, $candidate.Length)  
        $candidateN \= $candidate.Substring(0, $baseLength) \+ $num  
        $existsN \= Get-ADUser \-Filter "SamAccountName \-eq '$candidateN'" \-ErrorAction SilentlyContinue  
        if (-not $existsN) {  
            return $candidateN  
        }  
    }

    throw "No se pudo generar un samAccountName único para $GivenName $Surname"  
}

function Get-DepartmentGroupName {  
    param(\[string\]$Department)  
    return "$GroupPrefix$(Normalize-GroupPart \-Text $Department)"  
}

try {  
    Import-Module ActiveDirectory \-ErrorAction Stop  
}  
catch {  
    throw "No se pudo cargar el módulo ActiveDirectory. Ejecuta el script en el DC principal o en un servidor/equipo con RSAT instalado."  
}

if (-not (Test-Path \-LiteralPath $CsvPath)) {  
    throw "No existe el CSV: $CsvPath"  
}

$securePassword \= ConvertTo-SecureString $DefaultPassword \-AsPlainText \-Force  
$domainDn \= ($DomainFqdn \-split '\\.' | ForEach-Object { "DC=$\_" }) \-join ','

if ($BaseOU \-notmatch '^OU=.\*?,DC=.\*') {  
    throw "BaseOU debe ser un DN válido, por ejemplo: OU=Usuaris,$domainDn"  
}

Ensure-OU \-DistinguishedName $BaseOU  
$null \= Ensure-Group \-Name $HeadsGroupName \-Path $BaseOU

$rows \= Import-Csv \-LiteralPath $CsvPath  
$siteRows \= $rows | Where-Object { $\_.sede \-eq $SiteName }

if (-not $siteRows \-or $siteRows.Count \-eq 0\) {  
    throw "No se han encontrado usuarios para la sede '$SiteName' en el CSV."  
}

$results \= New-Object System.Collections.Generic.List\[object\]

foreach ($row in $siteRows) {  
    $fullName \= @($row.nom, $row.cognom1, $row.cognom2) | Where-Object { \-not \[string\]::IsNullOrWhiteSpace($\_) }  
    $displayName \= ($fullName \-join ' ').Trim()

    try {  
        $givenName \= (Normalize-Token \-Text $row.nom)  
        $givenName \= (Get-Culture).TextInfo.ToTitleCase($givenName)

        $surnameParts \= @()  
        if (-not \[string\]::IsNullOrWhiteSpace($row.cognom1)) { $surnameParts \+= ((Get-Culture).TextInfo.ToTitleCase((Normalize-Token \-Text $row.cognom1))) }  
        if (-not \[string\]::IsNullOrWhiteSpace($row.cognom2)) { $surnameParts \+= ((Get-Culture).TextInfo.ToTitleCase((Normalize-Token \-Text $row.cognom2))) }  
        $surname \= ($surnameParts \-join ' ').Trim()

        $sam \= Get-UniqueSamAccountName \-GivenName $row.nom \-Surname $row.cognom1 \-Dni $row.dni  
        $upn \= "$sam@$UpnSuffix"  
        $mail \= "$sam@$MailDomain"  
        $department \= ($row.dept ?? '').Trim()  
        $title \= ($row.descrip ?? '').Trim()  
        $office \= ($row.sede ?? '').Trim()  
        $employeeId \= ($row.dni ?? '').Trim()  
        $departmentGroup \= Get-DepartmentGroupName \-Department $department

        $null \= Ensure-Group \-Name $departmentGroup \-Path $BaseOU

        $existing \= Get-ADUser \-Filter "SamAccountName \-eq '$sam'" \-ErrorAction SilentlyContinue  
        if ($existing) {  
            $results.Add(\[pscustomobject\]@{  
                DisplayName     \= $displayName  
                SamAccountName  \= $sam  
                Department      \= $department  
                Status          \= 'Omitido'  
                Details         \= 'El usuario ya existe en Active Directory'  
            })  
            continue  
        }

        $params \= @{  
            Name                  \= $displayName  
            DisplayName           \= $displayName  
            GivenName             \= $givenName  
            Surname               \= $surname  
            SamAccountName        \= $sam  
            UserPrincipalName     \= $upn  
            EmailAddress          \= $mail  
            AccountPassword       \= $securePassword  
            Enabled               \= $EnableUsers  
            ChangePasswordAtLogon \= $ForcePasswordChangeAtLogon  
            Path                  \= $BaseOU  
            Department            \= $department  
            Title                 \= $title  
            Office                \= $office  
            EmployeeID            \= $employeeId  
            OtherAttributes       \= @{ info \= "Importado automáticamente desde CSV para la sede $SiteName" }  
        }

        Write-Host "\[INFO\] Creando usuario: $displayName ($sam)"  
        if (-not $WhatIfMode) {  
            New-ADUser @params  
            Add-ADGroupMember \-Identity $departmentGroup \-Members $sam

            if ($title \-eq 'Jefe') {  
                Add-ADGroupMember \-Identity $HeadsGroupName \-Members $sam  
            }  
        }

        $details \= "Grupo depto: $departmentGroup"  
        if ($title \-eq 'Jefe') {  
            $details \+= "; Grupo jefes: $HeadsGroupName"  
        }

        $results.Add(\[pscustomobject\]@{  
            DisplayName     \= $displayName  
            SamAccountName  \= $sam  
            Department      \= $department  
            Status          \= $(if ($WhatIfMode) { 'Simulado' } else { 'Creado' })  
            Details         \= $details  
        })  
    }  
    catch {  
        $results.Add(\[pscustomobject\]@{  
            DisplayName     \= $displayName  
            SamAccountName  \= ''  
            Department      \= ($row.dept ?? '').Trim()  
            Status          \= 'Error'  
            Details         \= $\_.Exception.Message  
        })  
    }  
}

$results | Export-Csv \-LiteralPath $LogPath \-NoTypeInformation \-Encoding UTF8

Write-Host ""  
Write-Host "================ RESUMEN \================"  
$results | Group-Object Status | ForEach-Object {  
    Write-Host ("{0}: {1}" \-f $\_.Name, $\_.Count)  
}  
Write-Host "Log generado en: $LogPath"  
Write-Host "========================================="