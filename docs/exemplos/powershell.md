# Exemplos PowerShell — API BDesk

Funcoes helper e exemplos de uso da API BDesk em PowerShell 5.1 e 7+.

> **Compatibilidade:** PowerShell 5.1 (Windows) e PowerShell 7+ (cross-platform). Todos os exemplos usam `Invoke-RestMethod` — sem dependencias externas.

> **Sobre erros:** a API sinaliza falhas de duas formas: HTTP 406 com a mensagem em **texto puro**, ou HTTP 200 com a mensagem em `MensagensErro` (no envelope `_metadata`, ou na raiz da resposta no login e em `GET /v1/requisicoes/{id}`). As funcoes abaixo tratam os dois casos. Veja [Tratamento de Erros](../referencia/erros.md).

---

## 1. Funcoes Helper

Cole o bloco abaixo em um arquivo `.ps1` ou diretamente no perfil do PowerShell para reutilizar nas sessoes.

```powershell
# bdesk-api.ps1 — Funcoes helper para a API BDesk
# Uso: . .\bdesk-api.ps1

#region Funcoes Internas

function Get-BDeskCorpoErro {
    # Le o corpo de uma resposta HTTP de erro (no PS 5.1 e no PS 7+).
    param([System.Management.Automation.ErrorRecord]$Err)

    if ($Err.ErrorDetails -and $Err.ErrorDetails.Message) {
        return $Err.ErrorDetails.Message
    }
    try {
        $stream = $Err.Exception.Response.GetResponseStream()
        $reader = [System.IO.StreamReader]::new($stream)
        return $reader.ReadToEnd()
    }
    catch {
        return $Err.Exception.Message
    }
}

function Stop-BDeskComErro {
    # Converte um erro HTTP em excecao com a mensagem da API.
    #  - 406: o corpo e TEXTO PURO (nao e JSON) com a mensagem de negocio.
    #  - 401: token invalido ou usuario desativado (refaca o login).
    #  - 404: rota inexistente ou parametro de query obrigatorio ausente.
    param([System.Management.Automation.ErrorRecord]$Err)

    $statusCode = 0
    if ($Err.Exception.Response) { $statusCode = [int]$Err.Exception.Response.StatusCode }

    if ($statusCode -eq 406) {
        throw "Erro de negocio (406): $(Get-BDeskCorpoErro $Err)"
    }
    if ($statusCode -eq 401) {
        throw "Nao autenticado (401): token invalido ou usuario desativado. Conecte-se novamente."
    }
    throw "Erro HTTP ${statusCode}: $($Err.Exception.Message)"
}

function Test-BDeskMensagensErro {
    # Padrao (b): a API pode responder 200 com a mensagem em MensagensErro.
    # O campo fica em _metadata.MensagensErro (envelope padrao) ou na raiz.
    param($Resposta)

    if ($null -eq $Resposta -or $Resposta -is [string]) { return }

    $msgs = $null
    if ($Resposta._metadata -and $Resposta._metadata.MensagensErro) {
        $msgs = $Resposta._metadata.MensagensErro
    }
    elseif ($Resposta.MensagensErro) {
        $msgs = $Resposta.MensagensErro
    }
    if ($msgs) {
        throw "Erro da API: $($msgs -join '; ')"
    }
}

#endregion

#region Autenticacao

function Connect-BDesk {
    <#
    .SYNOPSIS
        Autentica na API BDesk e retorna um objeto de sessao.

    .DESCRIPTION
        Realiza login na API e retorna uma hashtable com BaseUrl, Token e Headers
        prontos para uso nas demais funcoes. O login responde HTTP 200 mesmo quando
        falha (Dados nulo e MensagensErro preenchido). Quando funciona, o campo Dados
        e uma string JSON escapada — este cmdlet realiza os dois niveis de parse.

        O token nao expira por tempo no servidor (expires_in e apenas informativo):
        reutilize-o. Se uma chamada responder 401, conecte-se novamente.

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
        Stop-BDeskComErro $_
    }

    # O login responde 200 mesmo em falha: confira Dados e MensagensErro
    if (-not $resposta.Dados -or $resposta.MensagensErro) {
        throw "Falha no login: $($resposta.MensagensErro -join '; ')"
    }

    # Dados e uma string JSON escapada — requer dois niveis de parse
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

    .DESCRIPTION
        A API NAO pagina: a resposta traz as requisicoes ate o limite informado
        (LimiteRequisicoes, padrao 500). Para reduzir o volume, use -Filtros.

    .PARAMETER Session
        Objeto de sessao retornado por Connect-BDesk.

    .PARAMETER Limite
        Maximo de requisicoes devolvidas (LimiteRequisicoes, padrao 500).

    .PARAMETER Filtros
        Hashtable opcional de filtros, enviada no corpo de um POST. Ex.:
        @{ DescricoesStatus = @("Aberta");
           AbertoEntre = @{ Inicio = "2026-01-01T00:00:00"; Fim = "2026-01-31T23:59:59" } }

    .EXAMPLE
        $abertas = Get-BDeskRequisicoes -Session $sessao -Limite 10
        $abertas.records | ForEach-Object { Write-Host "#$($_.RequisicaoId) - $($_.Assunto)" }
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][hashtable]$Session,
        [int]$Limite = 500,
        [hashtable]$Filtros
    )

    $url = "$($Session.BaseUrl)/v1/requisicoes/abertas"

    try {
        if ($Filtros) {
            $Filtros["LimiteRequisicoes"] = $Limite
            $corpo = $Filtros | ConvertTo-Json -Depth 10
            $resposta = Invoke-RestMethod -Uri $url -Method POST `
                -Headers $Session.Headers -ContentType "application/json" `
                -Body $corpo -ErrorAction Stop
        }
        else {
            $resposta = Invoke-RestMethod -Uri "${url}?LimiteRequisicoes=$Limite" -Method GET `
                -Headers $Session.Headers -ErrorAction Stop
        }
    }
    catch {
        Stop-BDeskComErro $_
    }

    Test-BDeskMensagensErro $resposta
    return $resposta
}

function Get-BDeskRequisicao {
    <#
    .SYNOPSIS
        Retorna detalhes completos de uma requisicao pelo ID.

    .DESCRIPTION
        Requisicao inexistente ou sem acesso volta com HTTP 200, Conjuntos nulo e
        MensagensErro preenchido na raiz da resposta; esta funcao converte isso em erro.

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
        Stop-BDeskComErro $_
    }

    Test-BDeskMensagensErro $resposta
    if (-not $resposta.Conjuntos) {
        throw "Requisicao nao encontrada ou sem acesso."
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
        Numero (int) da requisicao criada.
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
        # /abrir retorna o numero como texto JSON simples (ex: "12345")
        $resposta = Invoke-RestMethod -Uri $url -Method POST `
            -Headers $Session.Headers -ContentType "application/json" `
            -Body $payload -ErrorAction Stop
    }
    catch {
        Stop-BDeskComErro $_
    }

    return [int]$resposta
}

function Invoke-BDeskAcao {
    <#
    .SYNOPSIS
        Executa uma acao de workflow em uma requisicao.

    .DESCRIPTION
        A maioria dos erros desta rota volta com HTTP 200 e a mensagem em
        _metadata.MensagensErro (acao nao encontrada, usuario sem permissao neste
        status). Esta funcao converte esses casos em excecao.

    .PARAMETER Session
        Objeto de sessao retornado por Connect-BDesk.

    .PARAMETER RequisicaoId
        ID da requisicao.

    .PARAMETER AcaoId
        Identificador EXATO da acao, como devolvido por Get-BDeskAcoes, com o
        codigo entre colchetes (ex: "Encerrar [ENC]", "Direcionar [DIR]").
        "ENC" sozinho resulta em "Acao nao encontrada".

    .PARAMETER Descricao
        Comentario/descricao da acao (pode ser obrigatorio dependendo da configuracao).

    .PARAMETER Params
        Hashtable com parametros adicionais da acao:
          tipoAvaliacao  (int)  — 1, 2 ou 3 (ENC e AVAL)
          prioridade     (int)  — Nova prioridade (ALTPRI)
          NovoSolicitado (str)  — Id do destino, copiado de acoes/DIR/grupos (DIR)
          usuResponsavelId (int) — ID do usuario (ATR e ATRR)
          IdRequisicaoAVincular (int) — ID a vincular (VINC)

    .EXAMPLE
        Invoke-BDeskAcao -Session $sessao -RequisicaoId 35174 `
                         -AcaoId "Encerrar [ENC]" -Descricao "Resolvido." `
                         -Params @{ tipoAvaliacao = 2 }
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
        Stop-BDeskComErro $_
    }

    Test-BDeskMensagensErro $resposta
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
        Stop-BDeskComErro $_
    }

    Test-BDeskMensagensErro $resposta
    return $resposta.records
}

function Get-BDeskDestinosDirecionar {
    <#
    .SYNOPSIS
        Lista os grupos/usuarios que podem receber a requisicao na acao DIR.

    .OUTPUTS
        Itens com Id (texto pronto, use sem alterar em NovoSolicitado) e Texto (nome).
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][hashtable]$Session,
        [Parameter(Mandatory)][int]$RequisicaoId
    )

    $url = "$($Session.BaseUrl)/v1/requisicoes/$RequisicaoId/acoes/DIR/grupos"

    try {
        $resposta = Invoke-RestMethod -Uri $url -Method GET `
            -Headers $Session.Headers -ErrorAction Stop
    }
    catch {
        Stop-BDeskComErro $_
    }

    Test-BDeskMensagensErro $resposta
    return $resposta.records
}

function Send-BDeskArquivo {
    <#
    .SYNOPSIS
        ETAPA 1 do anexo: faz upload do arquivo para a area temporaria.

    .DESCRIPTION
        ATENCAO: isto NAO anexa o arquivo a requisicao. A resposta e 200 mesmo assim.
        Depois do upload, chame Submit-BDeskAnexos (etapa 2) com o GUID devolvido.

    .PARAMETER Session
        Objeto de sessao retornado por Connect-BDesk.

    .PARAMETER RequisicaoId
        ID da requisicao.

    .PARAMETER CaminhoArquivo
        Caminho completo do arquivo a enviar.

    .OUTPUTS
        GUID (texto) que identifica o arquivo na area temporaria.
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

    # Montar payload multipart/form-data manualmente (funciona no PS 5.1 e no 7+)
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

    $headersUpload = @{ Authorization = "Bearer $($Session.Token)" }

    try {
        $resposta = Invoke-RestMethod -Uri $url -Method POST `
            -Headers $headersUpload `
            -ContentType "multipart/form-data; boundary=$boundary" `
            -Body $corpoBytes -ErrorAction Stop
    }
    catch {
        Stop-BDeskComErro $_
    }

    # Falha ao gravar: HTTP 200 com Id vazio e MensagensErro preenchido
    if ($resposta.MensagensErro -or -not $resposta.Id) {
        throw "Erro no upload: $($resposta.MensagensErro -join '; ')"
    }

    return $resposta.Id
}

function Submit-BDeskAnexos {
    <#
    .SYNOPSIS
        ETAPA 2 do anexo: vincula os arquivos ja enviados a requisicao.

    .PARAMETER Anexos
        Array de hashtables @{ Id = <GUID da etapa 1>; NomeDuranteUpload = "relatorio.pdf"
        (COM extensao); Titulo = "Relatorio de marco" }.

    .PARAMETER CodigoAcao
        Codigo da acao "anexar documento" (normalmente ANDOC).

    .NOTES
        Erros comuns (HTTP 406, texto puro): extensao nao permitida, caractere nao
        permitido no nome, usuario sem permissao de anexar neste status.
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][hashtable]$Session,
        [Parameter(Mandatory)][int]$RequisicaoId,
        [Parameter(Mandatory)][array]$Anexos,
        [string]$CodigoAcao = "ANDOC"
    )

    $corpo = @{ CodigoAcao = $CodigoAcao; Anexos = $Anexos } | ConvertTo-Json -Depth 5
    $url   = "$($Session.BaseUrl)/v1/requisicoes/$RequisicaoId/anexos/submeter"

    try {
        $resposta = Invoke-RestMethod -Uri $url -Method POST `
            -Headers $Session.Headers -ContentType "application/json" `
            -Body $corpo -ErrorAction Stop
    }
    catch {
        Stop-BDeskComErro $_
    }

    Test-BDeskMensagensErro $resposta
}

function Add-BDeskAnexo {
    <#
    .SYNOPSIS
        Anexa um arquivo a uma requisicao (upload + submissao, as 2 etapas).

    .EXAMPLE
        Add-BDeskAnexo -Session $sessao -RequisicaoId 35174 -CaminhoArquivo "C:\relatorios\relatorio.pdf"
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][hashtable]$Session,
        [Parameter(Mandatory)][int]$RequisicaoId,
        [Parameter(Mandatory)][string]$CaminhoArquivo,
        [string]$Titulo
    )

    $nome = Split-Path $CaminhoArquivo -Leaf
    if (-not $Titulo) { $Titulo = $nome }

    $guid = Send-BDeskArquivo -Session $Session -RequisicaoId $RequisicaoId -CaminhoArquivo $CaminhoArquivo
    Submit-BDeskAnexos -Session $Session -RequisicaoId $RequisicaoId `
        -Anexos @(@{ Id = $guid; NomeDuranteUpload = $nome; Titulo = $Titulo })
}

function Get-BDeskAnexos {
    <#
    .SYNOPSIS
        Lista os anexos ja vinculados a requisicao (rota no plural: /anexos).
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][hashtable]$Session,
        [Parameter(Mandatory)][int]$RequisicaoId
    )

    $url = "$($Session.BaseUrl)/v1/requisicoes/$RequisicaoId/anexos"

    try {
        $resposta = Invoke-RestMethod -Uri $url -Method GET `
            -Headers $Session.Headers -ErrorAction Stop
    }
    catch {
        Stop-BDeskComErro $_
    }

    Test-BDeskMensagensErro $resposta
    return $resposta.records
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
        Stop-BDeskComErro $_
    }

    return $resposta.records
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
# A API nao pagina: informe um limite (padrao 500) e use filtros para reduzir o volume
$abertas = Get-BDeskRequisicoes -Session $sessao -Limite 10

Write-Host "$($abertas.records.Count) requisicoes abertas"
if ($abertas.records.Count -ge 10) {
    Write-Warning "O limite foi atingido; pode haver mais requisicoes."
}
Write-Host ""

$abertas.records | ForEach-Object {
    Write-Host "#$($_.RequisicaoId) - $($_.Assunto)"
    Write-Host "  Status: $($_.Status) | Responsavel: $($_.Responsavel)"
    Write-Host "  Abertura: $($_.DataAbertura.Substring(0,10))"
}
```

**Com filtros (periodo de abertura e status):**

```powershell
$janeiro = Get-BDeskRequisicoes -Session $sessao -Limite 200 -Filtros @{
    DescricoesStatus = @("Aberta")
    AbertoEntre      = @{ Inicio = "2026-01-01T00:00:00"; Fim = "2026-01-31T23:59:59" }
}
Write-Host "$($janeiro.records.Count) requisicoes em janeiro"
```

### Buscar Detalhes de uma Requisicao

```powershell
try {
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
}
catch {
    # Inclui o caso "requisicao inexistente ou sem acesso" (HTTP 200 com MensagensErro)
    Write-Error "Nao foi possivel consultar: $_"
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
# Ver acoes disponiveis (o Id inclui o codigo entre colchetes)
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
        -Params       @{ tipoAvaliacao = 2 }

    Write-Host "Requisicao #$reqId encerrada com sucesso."
}
catch {
    # Cobre HTTP 406 e tambem HTTP 200 com MensagensErro
    Write-Error "Nao foi possivel encerrar: $_"
}
```

**Outros exemplos de acoes:**

```powershell
# Direcionar: o Id do destino vem de acoes/DIR/grupos e vai, sem alteracao, em NovoSolicitado
$destinos = Get-BDeskDestinosDirecionar -Session $sessao -RequisicaoId $reqId
Invoke-BDeskAcao -Session $sessao -RequisicaoId $reqId `
    -AcaoId "Direcionar [DIR]" `
    -Descricao "Direcionando para infra." `
    -Params @{ NovoSolicitado = $destinos[0].Id }

# Alterar prioridade (1 = Alta, 2 = Media, 3 = Baixa)
Invoke-BDeskAcao -Session $sessao -RequisicaoId $reqId `
    -AcaoId "Alterar Prioridade [ALTPRI]" `
    -Descricao "Impacto em producao." `
    -Params @{ prioridade = 1 }

# Vincular requisicoes
Invoke-BDeskAcao -Session $sessao -RequisicaoId $reqId `
    -AcaoId "Vincular [VINC]" `
    -Params @{ IdRequisicaoAVincular = 12300 }
```

### Anexar Arquivo (2 etapas)

O envio de um anexo tem duas chamadas: o **upload** apenas guarda o arquivo numa area temporaria; a **submissao** e que o vincula a requisicao. Parar no upload deixa a requisicao sem anexo, mesmo com resposta 200.

```powershell
try {
    # Atalho: faz as duas etapas
    Add-BDeskAnexo -Session $sessao -RequisicaoId $reqId `
        -CaminhoArquivo "C:\relatorios\relatorio.pdf" -Titulo "Relatorio de marco"

    # Confira que o documento apareceu na lista
    Get-BDeskAnexos -Session $sessao -RequisicaoId $reqId |
        ForEach-Object { Write-Host "$($_.Id): $($_.Titulo) ($($_.NomeDocumentoFisico))" }
}
catch {
    # Ex.: extensao nao permitida, usuario sem permissao de anexar neste status
    Write-Error "Anexo nao vinculado: $_"
}
```

**Passo a passo, com varios arquivos numa unica submissao:**

```powershell
$arquivos = @("C:\relatorios\a.pdf", "C:\relatorios\b.png")

$itens = foreach ($caminho in $arquivos) {
    $guid = Send-BDeskArquivo -Session $sessao -RequisicaoId $reqId -CaminhoArquivo $caminho   # etapa 1
    $nome = Split-Path $caminho -Leaf
    @{ Id = $guid; NomeDuranteUpload = $nome; Titulo = $nome }
}

Submit-BDeskAnexos -Session $sessao -RequisicaoId $reqId -Anexos @($itens)                     # etapa 2
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

$agora  = Get-Date
$limite = 500

# A API nao pagina: uma unica chamada traz ate $limite requisicoes
$lote = Get-BDeskRequisicoes -Session $sessao -Limite $limite

if ($lote.records.Count -ge $limite) {
    Write-Warning "O limite de $limite requisicoes foi atingido; o relatorio pode estar incompleto. Divida a consulta por periodo de abertura (-Filtros)."
}

$atrasadas = @()
foreach ($req in $lote.records) {
    if ($req.DataFimPrevisto) {
        $prazo = [datetime]$req.DataFimPrevisto
        if ($prazo -lt $agora) {
            $atrasadas += [PSCustomObject]@{
                Id          = $req.RequisicaoId
                Assunto     = $req.Assunto
                Responsavel = $req.Responsavel
                Prazo       = $prazo.ToString("dd/MM/yyyy HH:mm")
                AtrasoDias  = [math]::Round(($agora - $prazo).TotalDays, 1)
            }
        }
    }
}

Write-Host "=== Requisicoes em Atraso ($($atrasadas.Count)) ==="
$atrasadas | Sort-Object AtrasoDias -Descending |
    Format-Table Id, Assunto, Responsavel, Prazo, AtrasoDias -AutoSize
```

---

## Referencia Rapida

| Funcao | Endpoint | Descricao |
|--------|----------|-----------|
| `Connect-BDesk -BaseUrl -Login -Senha` | `POST /v1/login/entrar` | Autentica e retorna sessao |
| `Get-BDeskRequisicoes -Session [-Limite] [-Filtros]` | `GET` ou `POST /v1/requisicoes/abertas` | Lista requisicoes abertas (sem paginacao; limite padrao 500) |
| `Get-BDeskRequisicao -Session -RequisicaoId` | `GET /v1/requisicoes/{id}` | Detalhes de uma requisicao |
| `New-BDeskRequisicao -Session -FormularioId -Assunto -Descricao [-Conjuntos]` | `POST /v1/requisicoes/abrir` | Cria requisicao; retorna o numero (int) |
| `Get-BDeskAcoes -Session -RequisicaoId` | `GET /v1/requisicoes/{id}/acoes` | Lista acoes disponiveis |
| `Get-BDeskDestinosDirecionar -Session -RequisicaoId` | `GET /v1/requisicoes/{id}/acoes/DIR/grupos` | Destinos para a acao DIR |
| `Invoke-BDeskAcao -Session -RequisicaoId -AcaoId [-Descricao] [-Params]` | `POST /v1/requisicoes/{id}/acoes` | Executa acao de workflow |
| `Send-BDeskArquivo -Session -RequisicaoId -CaminhoArquivo` | `POST /v1/requisicoes/{id}/anexo` | Anexo, etapa 1: upload (area temporaria) |
| `Submit-BDeskAnexos -Session -RequisicaoId -Anexos` | `POST /v1/requisicoes/{id}/anexos/submeter` | Anexo, etapa 2: vincula a requisicao |
| `Add-BDeskAnexo -Session -RequisicaoId -CaminhoArquivo` | as duas rotas acima | Faz as 2 etapas |
| `Get-BDeskAnexos -Session -RequisicaoId` | `GET /v1/requisicoes/{id}/anexos` | Anexos ja vinculados |
| `Get-BDeskCatalogo -Session` | `GET /v1/cardapio` | Areas e formularios |
