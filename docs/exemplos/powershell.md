# Exemplos PowerShell — API BDesk

Funcoes helper e exemplos de uso da API BDesk em PowerShell 5.1 e 7+.

> **Compatibilidade:** PowerShell 5.1 (Windows) e PowerShell 7+ (cross-platform). Todos os exemplos usam `Invoke-RestMethod` e `Invoke-WebRequest` — sem dependencias externas.

---

## 1. Funcoes Helper

Cole o bloco abaixo em um arquivo `.ps1` ou diretamente no perfil do PowerShell para reutilizar nas sessoes.

```powershell
# bdesk-api.ps1 — Funcoes helper para a API BDesk
# Uso: . .\bdesk-api.ps1

#region Autenticacao

function Connect-BDesk {
    <#
    .SYNOPSIS
        Autentica na API BDesk e retorna um objeto de sessao.

    .DESCRIPTION
        Realiza login na API e retorna uma hashtable com BaseUrl, Token e Headers
        prontos para uso nas demais funcoes. O campo Dados da resposta de login
        e uma string JSON escapada — este cmdlet realiza os dois niveis de parse
        automaticamente.

    .PARAMETER BaseUrl
        URL base da API (ex: https://sua-empresa.bdesk.com.br/askrest).

    .PARAMETER Login
        Nome de usuario BDesk.

    .PARAMETER Senha
        Senha do usuario.

    .EXAMPLE
        $sessao = Connect-BDesk -BaseUrl "https://sua-empresa.bdesk.com.br/askrest" `
                                -Login "usuario" -Senha "senha"

    .OUTPUTS
        Hashtable com chaves: BaseUrl, Token, Headers
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][string]$BaseUrl,
        [Parameter(Mandatory)][string]$Login,
        [Parameter(Mandatory)][string]$Senha
    )

    $url = "$($BaseUrl.TrimEnd('/'))/v1/login/entrar"
    $corpo = @{ Login = $Login; Senha = $Senha } | ConvertTo-Json

    try {
        $resposta = Invoke-RestMethod -Uri $url -Method POST `
            -ContentType "application/json" -Body $corpo -ErrorAction Stop
    }
    catch {
        $statusCode = $_.Exception.Response.StatusCode.value__
        if ($statusCode -eq 406) {
            $stream   = $_.Exception.Response.GetResponseStream()
            $reader   = [System.IO.StreamReader]::new($stream)
            $corpoErr = $reader.ReadToEnd() | ConvertFrom-Json
            $msgs     = $corpoErr.MensagensErro -join "; "
            throw "Falha no login (406): $msgs"
        }
        throw "Erro ao conectar a API: $_"
    }

    # Dados e uma string JSON escapada — requer dois niveis de parse
    if (-not $resposta.Dados) {
        throw "Resposta de login invalida: campo Dados ausente."
    }
    $dados = $resposta.Dados | ConvertFrom-Json
    $token = $dados.access_token

    $sessao = @{
        BaseUrl = $BaseUrl.TrimEnd('/')
        Token   = $token
        Headers = @{ Authorization = "Bearer $token" }
    }

    Write-Verbose "Autenticado. Token: $($token.Substring(0, [Math]::Min(20, $token.Length)))..."
    return $sessao
}

#endregion

#region Requisicoes

function Get-BDeskRequisicoes {
    <#
    .SYNOPSIS
        Lista requisicoes abertas do usuario autenticado.

    .PARAMETER Session
        Objeto de sessao retornado por Connect-BDesk.

    .PARAMETER PageSize
        Itens por pagina (padrao 20, maximo 100).

    .PARAMETER PageNumber
        Numero da pagina (base 1, padrao 1).

    .EXAMPLE
        $abertas = Get-BDeskRequisicoes -Session $sessao -PageSize 10
        $abertas.records | ForEach-Object { Write-Host "#$($_.RequisicaoId) - $($_.Assunto)" }
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][hashtable]$Session,
        [int]$PageSize   = 20,
        [int]$PageNumber = 1
    )

    $url = "$($Session.BaseUrl)/v1/requisicoes/abertas?pageSize=$PageSize&pageNumber=$PageNumber"

    try {
        $resposta = Invoke-RestMethod -Uri $url -Method GET `
            -Headers $Session.Headers -ErrorAction Stop
    }
    catch {
        _Handle-BDeskError $_
    }

    _Check-BDeskErrors $resposta
    return $resposta
}

function Get-BDeskRequisicao {
    <#
    .SYNOPSIS
        Retorna detalhes completos de uma requisicao pelo ID.

    .PARAMETER Session
        Objeto de sessao retornado por Connect-BDesk.

    .PARAMETER RequisicaoId
        ID numerico da requisicao.

    .EXAMPLE
        $det = Get-BDeskRequisicao -Session $sessao -RequisicaoId 35174
        $det.Conjuntos.'Detalhes Do Pedido'.Assunto
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][hashtable]$Session,
        [Parameter(Mandatory)][int]$RequisicaoId
    )

    $url = "$($Session.BaseUrl)/v1/requisicoes/$RequisicaoId"

    try {
        $resposta = Invoke-RestMethod -Uri $url -Method GET `
            -Headers $Session.Headers -ErrorAction Stop
    }
    catch {
        _Handle-BDeskError $_
    }

    if ($resposta.MensagensErro) {
        throw "Erro da API: $($resposta.MensagensErro -join '; ')"
    }

    return $resposta
}

function New-BDeskRequisicao {
    <#
    .SYNOPSIS
        Abre uma nova requisicao no BDesk.

    .PARAMETER Session
        Objeto de sessao retornado por Connect-BDesk.

    .PARAMETER FormularioId
        ID do formulario (obtido via Get-BDeskCatalogo).

    .PARAMETER Assunto
        Titulo/assunto da requisicao.

    .PARAMETER Descricao
        Descricao detalhada do problema ou solicitacao.

    .PARAMETER Conjuntos
        Hashtable com conjuntos adicionais do formulario.
        A chave e o nome do conjunto (Chave do conjunto); o valor e uma hashtable
        com os campos. Para conjuntos com multiplas linhas (Multiplo: true),
        passe um array de hashtables.

    .EXAMPLE
        $reqId = New-BDeskRequisicao -Session $sessao -FormularioId 101 `
                                     -Assunto "Impressora nao liga" `
                                     -Descricao "Nao liga desde esta manha."
        Write-Host "Requisicao criada: #$reqId"

    .OUTPUTS
        ID numerico (int) da requisicao criada.
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][hashtable]$Session,
        [Parameter(Mandatory)][int]$FormularioId,
        [Parameter(Mandatory)][string]$Assunto,
        [Parameter(Mandatory)][string]$Descricao,
        [hashtable]$Conjuntos = @{}
    )

    $dadosBasicos = @{
        Assunto   = $Assunto
        Descricao = $Descricao
    }
    $todosConjuntos = @{ DadosBasicos = $dadosBasicos }
    foreach ($chave in $Conjuntos.Keys) {
        $todosConjuntos[$chave] = $Conjuntos[$chave]
    }

    $payload = @{
        Formulario = $FormularioId
        Conjuntos  = $todosConjuntos
    } | ConvertTo-Json -Depth 10

    $url = "$($Session.BaseUrl)/v1/requisicoes/abrir"

    try {
        # /abrir retorna o ID como string simples (ex: "12345")
        $resposta = Invoke-RestMethod -Uri $url -Method POST `
            -Headers $Session.Headers -ContentType "application/json" `
            -Body $payload -ErrorAction Stop
    }
    catch {
        _Handle-BDeskError $_
    }

    return [int]$resposta
}

function Invoke-BDeskAcao {
    <#
    .SYNOPSIS
        Executa uma acao de workflow em uma requisicao.

    .PARAMETER Session
        Objeto de sessao retornado por Connect-BDesk.

    .PARAMETER RequisicaoId
        ID da requisicao.

    .PARAMETER AcaoId
        Identificador da acao no formato "Nome [CODIGO]"
        (ex: "Encerrar [ENC]", "Direcionar [DIR]").
        Use Get-BDeskAcoes para listar as acoes disponiveis.

    .PARAMETER Descricao
        Comentario/descricao da acao (pode ser obrigatorio dependendo da configuracao).

    .PARAMETER Params
        Hashtable com parametros adicionais da acao:
          Motivo       (int)    — ID do motivo (ENC)
          prioridade   (int)    — Nova prioridade (ALTPRI)
          NovoSolicitado (str)  — ID do novo solicitado (DIR)
          GrupoId      (int)    — Novo grupo (DIR)
          IdRequisicaoAVincular (int) — ID a vincular (VINC)

    .EXAMPLE
        Invoke-BDeskAcao -Session $sessao -RequisicaoId 35174 `
                         -AcaoId "Encerrar [ENC]" -Descricao "Resolvido." `
                         -Params @{ Motivo = 1 }
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][hashtable]$Session,
        [Parameter(Mandatory)][int]$RequisicaoId,
        [Parameter(Mandatory)][string]$AcaoId,
        [string]$Descricao = "",
        [hashtable]$Params = @{}
    )

    $payload = @{ Id = $AcaoId; Descricao = $Descricao }
    foreach ($chave in $Params.Keys) {
        $payload[$chave] = $Params[$chave]
    }

    $corpo = $payload | ConvertTo-Json -Depth 5
    $url   = "$($Session.BaseUrl)/v1/requisicoes/$RequisicaoId/acoes"

    try {
        $resposta = Invoke-RestMethod -Uri $url -Method POST `
            -Headers $Session.Headers -ContentType "application/json" `
            -Body $corpo -ErrorAction Stop
    }
    catch {
        _Handle-BDeskError $_
    }

    _Check-BDeskErrors $resposta
    return $resposta
}

function Get-BDeskAcoes {
    <#
    .SYNOPSIS
        Lista as acoes disponiveis para o usuario na requisicao.

    .PARAMETER Session
        Objeto de sessao retornado por Connect-BDesk.

    .PARAMETER RequisicaoId
        ID da requisicao.

    .EXAMPLE
        $acoes = Get-BDeskAcoes -Session $sessao -RequisicaoId 35174
        $acoes | ForEach-Object { Write-Host $_.Id }
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][hashtable]$Session,
        [Parameter(Mandatory)][int]$RequisicaoId
    )

    $url = "$($Session.BaseUrl)/v1/requisicoes/$RequisicaoId/acoes"

    try {
        $resposta = Invoke-RestMethod -Uri $url -Method GET `
            -Headers $Session.Headers -ErrorAction Stop
    }
    catch {
        _Handle-BDeskError $_
    }

    _Check-BDeskErrors $resposta
    return $resposta.records
}

function Send-BDeskAnexo {
    <#
    .SYNOPSIS
        Faz upload de um arquivo como anexo de uma requisicao.

    .PARAMETER Session
        Objeto de sessao retornado por Connect-BDesk.

    .PARAMETER RequisicaoId
        ID da requisicao.

    .PARAMETER CaminhoArquivo
        Caminho completo do arquivo a enviar.

    .EXAMPLE
        $result = Send-BDeskAnexo -Session $sessao -RequisicaoId 35174 `
                                  -CaminhoArquivo "C:\relatorios\relatorio.pdf"
        Write-Host "Anexo enviado. ID: $($result.Id)"

    .OUTPUTS
        Objeto com Id (ID do anexo) e MensagensErro.
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][hashtable]$Session,
        [Parameter(Mandatory)][int]$RequisicaoId,
        [Parameter(Mandatory)][string]$CaminhoArquivo
    )

    if (-not (Test-Path $CaminhoArquivo -PathType Leaf)) {
        throw "Arquivo nao encontrado: $CaminhoArquivo"
    }

    $url          = "$($Session.BaseUrl)/v1/requisicoes/$RequisicaoId/anexo"
    $nomeArquivo  = Split-Path $CaminhoArquivo -Leaf
    $boundary     = [System.Guid]::NewGuid().ToString()
    $bytes        = [System.IO.File]::ReadAllBytes($CaminhoArquivo)
    $encoding     = [System.Text.Encoding]::UTF8

    # Montar payload multipart/form-data manualmente
    $lf     = "`r`n"
    $inicio = $encoding.GetBytes(
        "--$boundary$lf" +
        "Content-Disposition: form-data; name=`"file`"; filename=`"$nomeArquivo`"$lf" +
        "Content-Type: application/octet-stream$lf$lf"
    )
    $fim    = $encoding.GetBytes("$lf--$boundary--$lf")

    $stream = [System.IO.MemoryStream]::new()
    $stream.Write($inicio, 0, $inicio.Length)
    $stream.Write($bytes,  0, $bytes.Length)
    $stream.Write($fim,    0, $fim.Length)
    $corpoBytes = $stream.ToArray()
    $stream.Dispose()

    $headersUpload = @{
        Authorization  = "Bearer $($Session.Token)"
        "Content-Type" = "multipart/form-data; boundary=$boundary"
    }

    try {
        $resposta = Invoke-RestMethod -Uri $url -Method POST `
            -Headers $headersUpload -Body $corpoBytes -ErrorAction Stop
    }
    catch {
        _Handle-BDeskError $_
    }

    if ($resposta.MensagensErro) {
        throw "Erro ao enviar anexo: $($resposta.MensagensErro -join '; ')"
    }

    return $resposta
}

#endregion

#region Catalogo

function Get-BDeskCatalogo {
    <#
    .SYNOPSIS
        Retorna todas as areas e formularios do catalogo de servicos.

    .PARAMETER Session
        Objeto de sessao retornado por Connect-BDesk.

    .EXAMPLE
        $catalogo = Get-BDeskCatalogo -Session $sessao
        $catalogo | ForEach-Object {
            Write-Host "[$($_.Id)] $($_.Nome)"
            $_.Formularios | ForEach-Object { Write-Host "  Formulario $($_.Id): $($_.Nome)" }
        }
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][hashtable]$Session
    )

    $url = "$($Session.BaseUrl)/v1/cardapio"

    try {
        $resposta = Invoke-RestMethod -Uri $url -Method GET `
            -Headers $Session.Headers -ErrorAction Stop
    }
    catch {
        _Handle-BDeskError $_
    }

    return $resposta.records
}

#endregion

#region Funcoes Internas

function _Handle-BDeskError {
    param([System.Management.Automation.ErrorRecord]$Err)

    $statusCode = $Err.Exception.Response.StatusCode.value__
    if ($statusCode -eq 406) {
        try {
            $stream   = $Err.Exception.Response.GetResponseStream()
            $reader   = [System.IO.StreamReader]::new($stream)
            $corpoErr = $reader.ReadToEnd() | ConvertFrom-Json
            $msgs     = (
                $corpoErr._metadata.MensagensErro `
                ?? $corpoErr.MensagensErro `
                ?? @("Erro de negocio (406).")
            ) -join "; "
            throw "Erro de negocio (406): $msgs"
        }
        catch [System.Management.Automation.RuntimeException] {
            throw
        }
        catch {
            throw "Erro HTTP 406: $($Err.Exception.Message)"
        }
    }
    throw "Erro HTTP $statusCode`: $($Err.Exception.Message)"
}

function _Check-BDeskErrors {
    param($Resposta)

    $msgs = $Resposta._metadata.MensagensErro ?? $Resposta.MensagensErro
    if ($msgs) {
        throw "Erro da API: $($msgs -join '; ')"
    }
}

#endregion
```

---

## 2. Exemplos de Uso

### Carregar as Funcoes

```powershell
# Carregar o arquivo de funcoes na sessao atual
. .\bdesk-api.ps1
```

### Autenticar

```powershell
$sessao = Connect-BDesk `
    -BaseUrl "https://sua-empresa.bdesk.com.br/askrest" `
    -Login "seu-usuario" `
    -Senha "sua-senha"

Write-Host "Conectado. Token: $($sessao.Token.Substring(0,20))..."
```

### Listar Requisicoes Abertas

```powershell
$abertas = Get-BDeskRequisicoes -Session $sessao -PageSize 10

$paginacao = $abertas._metadata.Pagination
Write-Host "Total: $($paginacao.TotalRecords) requisicoes | Pagina $($paginacao.CurrentPage)/$($paginacao.TotalPages)"
Write-Host ""

$abertas.records | ForEach-Object {
    Write-Host "#$($_.RequisicaoId) - $($_.Assunto)"
    Write-Host "  Status: $($_.Status) | Responsavel: $($_.Responsavel)"
    Write-Host "  Abertura: $($_.DataAbertura.Substring(0,10))"
}
```

### Buscar Detalhes de uma Requisicao

```powershell
$detalhes = Get-BDeskRequisicao -Session $sessao -RequisicaoId 35174

# Conjuntos e um dicionario — acesse por chave de secao
$info = $detalhes.Conjuntos.'Detalhes Do Pedido'
Write-Host "Assunto  : $($info.Assunto)"
Write-Host "Status   : $($info.Status)"
Write-Host "Descricao: $($info.Descricao)"

$participantes = $detalhes.Conjuntos.Participantes
$participantes.PSObject.Properties | ForEach-Object {
    Write-Host "  $($_.Name): $($_.Value)"
}
```

### Criar Requisicao

```powershell
try {
    $reqId = New-BDeskRequisicao `
        -Session      $sessao `
        -FormularioId 101 `
        -Assunto      "Impressora nao liga" `
        -Descricao    "A impressora do setor financeiro nao liga desde esta manha."

    Write-Host "Requisicao criada: #$reqId"
}
catch {
    Write-Error "Falha ao criar requisicao: $_"
}
```

**Com conjuntos adicionais:**

```powershell
# Conjuntos com linhas multiplas (formularios com Multiplo = true)
$itens = @(
    @{ Produto = "Teclado"; Quantidade = 2 },
    @{ Produto = "Mouse";   Quantidade = 5 }
)

$reqId = New-BDeskRequisicao `
    -Session      $sessao `
    -FormularioId 318 `
    -Assunto      "Compra de equipamentos" `
    -Descricao    "Solicitacao de compra para TI." `
    -Conjuntos    @{ Itens = $itens }

Write-Host "Requisicao de compra criada: #$reqId"
```

### Listar e Executar Acoes

```powershell
# Ver acoes disponiveis
$acoes = Get-BDeskAcoes -Session $sessao -RequisicaoId $reqId
Write-Host "Acoes disponiveis:"
$acoes | ForEach-Object { Write-Host "  $($_.Id)" }

# Encerrar a requisicao
try {
    Invoke-BDeskAcao `
        -Session      $sessao `
        -RequisicaoId $reqId `
        -AcaoId       "Encerrar [ENC]" `
        -Descricao    "Problema resolvido. Equipamento substituido." `
        -Params       @{ Motivo = 1 }

    Write-Host "Requisicao #$reqId encerrada com sucesso."
}
catch {
    Write-Error "Nao foi possivel encerrar: $_"
}
```

**Outros exemplos de acoes:**

```powershell
# Direcionar para outro grupo
Invoke-BDeskAcao -Session $sessao -RequisicaoId $reqId `
    -AcaoId "Direcionar [DIR]" `
    -Descricao "Direcionando para infra." `
    -Params @{ GrupoId = 10 }

# Alterar prioridade
Invoke-BDeskAcao -Session $sessao -RequisicaoId $reqId `
    -AcaoId "Alterar Prioridade [ALTPRI]" `
    -Descricao "Impacto em producao." `
    -Params @{ prioridade = 1 }

# Vincular requisicoes
Invoke-BDeskAcao -Session $sessao -RequisicaoId $reqId `
    -AcaoId "Vincular [VINC]" `
    -Params @{ IdRequisicaoAVincular = 12300 }
```

### Upload de Anexo

```powershell
try {
    $resultado = Send-BDeskAnexo `
        -Session       $sessao `
        -RequisicaoId  $reqId `
        -CaminhoArquivo "C:\relatorios\relatorio.pdf"

    Write-Host "Anexo enviado com sucesso. ID do anexo: $($resultado.Id)"
}
catch {
    Write-Error "Erro ao enviar anexo: $_"
}
```

### Explorar o Catalogo

```powershell
$areas = Get-BDeskCatalogo -Session $sessao

foreach ($area in $areas) {
    Write-Host "[$($area.Id)] $($area.Nome)"
    foreach ($frm in $area.Formularios) {
        Write-Host "  Formulario $($frm.Id): $($frm.Nome)"
    }
}
```

### Script de Relatorio: Requisicoes em Atraso

```powershell
#!/usr/bin/env pwsh
# relatorio-atraso.ps1 — Lista requisicoes com prazo vencido

. .\bdesk-api.ps1

$sessao = Connect-BDesk `
    -BaseUrl "https://sua-empresa.bdesk.com.br/askrest" `
    -Login   "seu-usuario" `
    -Senha   "sua-senha"

$agora    = Get-Date
$pagina   = 1
$total    = 0
$atrasadas = @()

do {
    $lote = Get-BDeskRequisicoes -Session $sessao -PageSize 100 -PageNumber $pagina
    $totalPaginas = $lote._metadata.Pagination.TotalPages

    foreach ($req in $lote.records) {
        if ($req.DataFimPrevisto) {
            $prazo = [datetime]$req.DataFimPrevisto
            if ($prazo -lt $agora) {
                $atrasadas += [PSCustomObject]@{
                    Id         = $req.RequisicaoId
                    Assunto    = $req.Assunto
                    Responsavel = $req.Responsavel
                    Prazo      = $prazo.ToString("dd/MM/yyyy HH:mm")
                    AtrasoDias = [math]::Round(($agora - $prazo).TotalDays, 1)
                }
                $total++
            }
        }
    }
    $pagina++
} while ($pagina -le $totalPaginas)

Write-Host "=== Requisicoes em Atraso ($total) ==="
$atrasadas | Sort-Object AtrasoDias -Descending |
    Format-Table Id, Assunto, Responsavel, Prazo, AtrasoDias -AutoSize
```

---

## Referencia Rapida

| Funcao | Endpoint | Descricao |
|--------|----------|-----------|
| `Connect-BDesk -BaseUrl -Login -Senha` | `POST /v1/login/entrar` | Autentica e retorna sessao |
| `Get-BDeskRequisicoes -Session [-PageSize] [-PageNumber]` | `GET /v1/requisicoes/abertas` | Lista requisicoes abertas |
| `Get-BDeskRequisicao -Session -RequisicaoId` | `GET /v1/requisicoes/{id}` | Detalhes de uma requisicao |
| `New-BDeskRequisicao -Session -FormularioId -Assunto -Descricao [-Conjuntos]` | `POST /v1/requisicoes/abrir` | Cria requisicao; retorna ID (int) |
| `Get-BDeskAcoes -Session -RequisicaoId` | `GET /v1/requisicoes/{id}/acoes` | Lista acoes disponiveis |
| `Invoke-BDeskAcao -Session -RequisicaoId -AcaoId [-Descricao] [-Params]` | `POST /v1/requisicoes/{id}/acoes` | Executa acao de workflow |
| `Send-BDeskAnexo -Session -RequisicaoId -CaminhoArquivo` | `POST /v1/requisicoes/{id}/anexo` | Upload de arquivo |
| `Get-BDeskCatalogo -Session` | `GET /v1/cardapio` | Areas e formularios |
