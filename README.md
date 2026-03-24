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
        return
    }

    $Key = Get-Content $PubKeyPath -Raw
    $Key = $Key.Trim()

    $RemoteCmd = "umask 077; mkdir -p ~/.ssh; touch ~/.ssh/authorized_keys; grep -qxF '$Key' ~/.ssh/authorized_keys || echo '$Key' >> ~/.ssh/authorized_keys"

    ssh $Target $RemoteCmd
}

function sa {
    param(
        [Parameter(Mandatory = $true)]
        [string]$Target
    )

    ssh -o BatchMode=yes -o ConnectTimeout=5 $Target exit 2>$null
    if ($LASTEXITCODE -ne 0) {
        Write-Host "SSH key not installed, adding it..."
        Add-SshKeyToServer -Target $Target
    }

    ssh $Target
}
```
