# подготовка
```bash
dir $HOME\.ssh # посмотреть есть ли ссш ключи
ssh-keygen -t ed25519 # создать если нет
notepad $PROFILE # отредактировать профиль повершела
New-Item -ItemType File -Path $PROFILE -Force # создать профиль если его нет

# в файл вставить код из блока ниже

. $PROFILE # перезагрузить профиль
```
# код для атвозагрузки ссш ключей на сервер
```bash
function Add-SshKeyToServer {
    param(
        [Parameter(Mandatory = $true)]
        [string]$Target,

        [string]$PubKeyPath = "$HOME\.ssh\id_ed25519.pub"
    )

    if (!(Test-Path $PubKeyPath)) {
        Write-Host "Public key not found: $PubKeyPath"
        return $false
    }

    $Key = (Get-Content $PubKeyPath -Raw).Trim()
    $RemoteCmd = "umask 077; mkdir -p ~/.ssh; touch ~/.ssh/authorized_keys; grep -qxF '$Key' ~/.ssh/authorized_keys || echo '$Key' >> ~/.ssh/authorized_keys"

    ssh $Target $RemoteCmd
    return ($LASTEXITCODE -eq 0)
}

function Add-SshHostConfig {
    param(
        [Parameter(Mandatory = $true)]
        [string]$Alias,

        [Parameter(Mandatory = $true)]
        [string]$Target,

        [string]$IdentityFile = "~/.ssh/id_ed25519"
    )

    $configPath = "$HOME\.ssh\config"
    $sshDir = "$HOME\.ssh"

    if (!(Test-Path $sshDir)) {
        New-Item -ItemType Directory -Path $sshDir -Force | Out-Null
    }

    if (!(Test-Path $configPath)) {
        New-Item -ItemType File -Path $configPath -Force | Out-Null
    }

    $user = $null
    $host = $null
    $port = $null

    if ($Target -match '^(?:(?<user>[^@]+)@)?(?<host>[^:]+?)(?::(?<port>\d+))?$') {
        $user = $Matches.user
        $host = $Matches.host
        $port = $Matches.port
    } else {
        Write-Host "Could not parse target: $Target"
        return $false
    }

    $existing = Get-Content $configPath -Raw
    if ($existing -match "(?ms)^\s*Host\s+$([regex]::Escape($Alias))\s*$") {
        Write-Host "Host alias already exists in SSH config: $Alias"
        return $false
    }

    $block = @()
    $block += ""
    $block += "Host $Alias"
    $block += "    HostName $host"
    if ($user) {
        $block += "    User $user"
    }
    if ($port) {
        $block += "    Port $port"
    }
    $block += "    IdentityFile $IdentityFile"
    $block += "    IdentitiesOnly yes"

    Add-Content -Path $configPath -Value ($block -join "`r`n")
    Write-Host "Added SSH config alias: $Alias"
    return $true
}

function sa {
    param(
        [Parameter(Mandatory = $true, Position = 0)]
        [string]$Target,

        [Parameter(Mandatory = $false)]
        [Alias("n")]
        [string]$Name
    )

    ssh -o BatchMode=yes -o ConnectTimeout=5 $Target exit 2>$null
    if ($LASTEXITCODE -ne 0) {
        Write-Host "SSH key not installed, adding it..."
        $ok = Add-SshKeyToServer -Target $Target
        if (-not $ok) {
            Write-Host "Failed to add SSH key."
            return
        }
    }

    if ($Name) {
        Add-SshHostConfig -Alias $Name -Target $Target | Out-Null
    }

    if ($Name) {
        ssh $Name
    } else {
        ssh $Target
    }
}```
