# ☁️ Atividade 1: Serverless

Infraestrutura de Software · ADS · CESAR School · 2026.2

Função serverless que recebe uma requisição do Rota Vital e devolve a guia de transporte, acionada por evento (`sam local invoke`) e por HTTP (`sam local start-api`).

**Caminho escolhido:** Caminho A (AWS SAM CLI local), sem conta em provedor de nuvem.

## 🏗️ Estrutura

```
serverless-rota-vital/
├── evidencias/              # prints dos comandos e das saídas
├── rota-vital-guias/
│   ├── events/pedido.json   # evento do API Gateway usado no sam local invoke
│   ├── hello_world/app.py   # função (lambda_handler)
│   ├── samconfig.toml
│   └── template.yaml        # rota POST /guias, Timeout em Globals
└── README.md
```

## ▶️ Como rodar

**Pré-requisitos:** Docker Desktop aberto, AWS SAM CLI (`sam --version`) e Python 3.12 (o `sam build` exige o 3.12 no PATH).

Todos os comandos rodam dentro de [`rota-vital-guias/`](./rota-vital-guias) (testado no PowerShell, Windows).

**1. Build**

```powershell
sam build
```

Se o build reclamar que não achou o Python 3.12, coloque-o na frente do PATH da janela atual (ajuste a pasta):

```powershell
$env:Path = "C:\Users\andre\AppData\Local\Programs\Python\Python312;" + $env:Path
```

**2. Invocar a função com um evento**

O [`events/pedido.json`](./rota-vital-guias/events/pedido.json) foi gerado com:

```powershell
$body = '{\"requisicaoId\":\"R-001\",\"hospital\":\"H-Recife\",\"bolsas\":[\"O-NEG-0142\"]}'
sam local generate-event apigateway aws-proxy --method POST --path guias --body $body | Set-Content -Encoding utf8NoBOM events/pedido.json
```

Esta versão do SAM CLI gera o `body` em base64. Como o `app.py` lê o JSON direto, editei o arquivo para `"isBase64Encoded": false` e `"body"` com o JSON puro (como o API Gateway envia).

```powershell
sam local invoke -e events/pedido.json
```

**3. Acionar por HTTP**

Terminal 1:

```powershell
sam local start-api
```

Terminal 2 (no PowerShell, use `curl.exe`, porque `curl` é alias de outro comando):

```powershell
curl.exe -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{\"requisicaoId\":\"R-002\",\"hospital\":\"H-Caruaru\"}'
```

Com reaproveitamento do ambiente: `sam local start-api --warm-containers LAZY`.

**4. Limite de tempo**

```powershell
curl.exe -i -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{\"requisicaoId\":\"R-003\",\"demorar\":true}'
```

O `Timeout` fica em `Globals` do `template.yaml`. Rodei com 3 s (erro) e depois com 10 s (sucesso). O valor final no repositório é **10**. Depois de mudar o `template.yaml`, rode `sam build` de novo.

## 🔎 Observações do ambiente

- O `sam init` gerou o template com `python3.14`; troquei para `python3.12`, como pede a atividade.
- O `samconfig.toml` gerado trazia `warm_containers = "EAGER"` em `local_start_api`, o que mudava o resultado do teste "sem flag". Removi esse bloco (ver Pergunta 2).
- Ao parar o `start-api` com `Ctrl+C`, o SAM CLI exibiu um `Fatal Python error` na limpeza interna (parada do observador de arquivos). Ocorreu depois das chamadas terminarem com 201 e não afeta a função.

## 🗨️ Perguntas

### 1. Por que a primeira invocação demorou mais que a segunda? O que acontece na fase de Init?

A primeira invocação demora mais porque o ambiente de execução ainda precisa ser criado. Essa criação é a fase de Init: o código é carregado, o runtime é iniciado e o que está fora do `lambda_handler` (como o `contador = 0`) é executado. Nas chamadas seguintes, se o ambiente for reaproveitado, essa etapa é pulada e a resposta sai mais rápido.

No meu teste local, a 1ª invocação levou 389,36 ms e a 2ª, 196,34 ms. O `Init Duration` ficou perto de zero nas duas (0,19 ms e 0,07 ms), então localmente a diferença vem provavelmente do Docker, e o cold start de verdade só apareceria na nuvem.

### 2. O que o campo `invocacaoNesteAmbiente` mostrou com e sem `--warm-containers`? O que isso diz sobre guardar estado numa variável global da função?

Sem `--warm-containers`, o `invocacaoNesteAmbiente` deu 1, 1, 1, porque cada chamada criou um container novo. Com `--warm-containers LAZY`, deu 1, 2, 3: o container é criado na primeira chamada e reaproveitado nas seguintes.

Percebi também que o `samconfig.toml` tinha `warm_containers = "EAGER"`, que já subia o container na partida e fazia o resultado "sem flag" dar 1, 2, 3. Isso mostra que uma variável global só mantém o valor se o ambiente for reaproveitado, e isso não é garantido. Por isso não se deve guardar estado importante nela, e sim em um serviço externo, como um banco de dados.

### 3. O que aconteceu com `demorar = true`? Qual é o tempo máximo de uma execução no Lambda, e o que fazer com um trabalho mais longo?

O limite de tempo é definido no `template.yaml` (`Timeout`). Ele estava em 3 s e, com `demorar = true`, a função dorme 5 s, então a execução foi cortada (`timed out after 3 seconds`) e a API respondeu 502. Ao aumentar para 10 s, a mesma requisição terminou em cerca de 5,5 s (`Duration` de 5465,53 ms) com 201.

O tempo máximo de uma execução no Lambda é de 15 minutos. Para um trabalho mais longo, o certo é dividir em partes menores, usar uma fila ou um fluxo de etapas, ou rodar em outro serviço (um container ou servidor) em vez de uma função só.

### 4. Quanto custa um milhão de execuções desta função com 128 MB e 100 ms?

Preços de us-east-1 da aula: US$ 0,20 por milhão de requisições e US$ 0,0000166667 por GB-s.

**Requisições:** o preço já é por milhão, e a função roda 1 milhão de vezes, então 1 × 0,20 = **US$ 0,20**.

**Computação (GB-s):**
- Memória: 128 MB ÷ 1024 = 0,125 GB
- Tempo: 100 ms ÷ 1000 = 0,1 s
- Por execução: 0,125 × 0,1 = 0,0125 GB-s
- Total: 0,0125 × 1.000.000 = 12.500 GB-s
- Custo: 12.500 × 0,0000166667 = **US$ 0,20833375**

**Total:** 0,20 + 0,20833375 = **US$ 0,40833375**, cerca de **US$ 0,41** por milhão de execuções.

### 5. Em que parte do Rota Vital uma função assim faria sentido, e em qual não faria?

**Onde faria sentido:** em partes da aplicação que quase não recebem chamadas ou que recebem muitas chamadas de repente (picos), e que não precisam guardar estado entre as chamadas. Por exemplo, o cálculo das estatísticas por hospital, que roda sob demanda, e a geração da guia de transporte.

**Onde não faria sentido:** em partes que recebem chamadas o dia todo, como a API principal usada pelos hospitais, em que um servidor ligado costuma sair mais barato. Também não serve para o que depende de estado em memória (estoque, fila de requisições, histórico), porque o serverless não garante estado entre chamadas.

## 📸 Evidências

Prints na pasta `evidencias/`:

| Arquivo | O que mostra |
|---|---|
| [`sam-build-sucesso.png`](./evidencias/01-sam-build-sucesso.png) | `sam build` concluído (Build Succeeded) |
| [`invoke-1.png`](./evidencias/02-invoke-1.png) | 1ª invocação com evento: 201, Duration 389,36 ms |
| [`invoke-2.png`](./evidencias/03-invoke-2.png)| 2ª invocação com evento: 201, Duration 196,34 ms |
| [`start-api-eager-curl.png`](./evidencias/04-start-api-eager-curl.png) | `start-api` com `EAGER` do `samconfig.toml`: contador 1, 2, 3 |
| [`start-api-eager-logs.png`](./evidencias/05-start-api-eager-logs.png) | Logs: container já de pé na partida |
| [`start-api-lazy-curl.png`](./evidencias/06-start-api-lazy-curl.png) | `--warm-containers LAZY`: contador 1, 2, 3 |
| [`start-api-lazy-logs.png`](./evidencias/07-start-api-lazy-logs.png) | Logs: container criado na 1ª chamada |
| [`start-api-semflag-curl.png`](./evidencias/08-start-api-semflag-curl.png) | Sem flag: contador 1, 1, 1 |
| [`start-api-semflag-logs(1).png`](./evidencias/09-start-api-semflag-logs(1).png) e [`start-api-semflag-logs(2).png`](./evidencias/09-start-api-semflag-logs(2).png) | Logs: um container novo por chamada |
| [`timeout-3s-curl.png`](./evidencias/10-timeout-3s-curl.png) | Resposta 502 BAD GATEWAY |
| [`timeout-3s-logs.png`](./evidencias/11-timeout-3s-logs.png) | `timed out after 3 seconds` | 
| [`timeout-10s-curl.png`](./evidencias/12-timeout-10s-curl.png) |  Resposta 201 CREATED, guia `G-R-003` |
| [`timeout-10s-logs.png`](./evidencias/13-timeout-10s-logs.png) | `REPORT` com Duration 5465,53 ms | 

## 🤖 Uso de IA

**Ferramenta:** Claude (Anthropic).

**Finalidade:** apoio na configuração do ambiente (SAM CLI, Docker e Python 3.12 no Windows), na depuração de erros (PATH, formato do evento, [`samconfig.toml`](./rota-vital-guias/samconfig.toml)), na explicação dos conceitos de serverless e na revisão das minhas respostas, que escrevi em rascunho e a IA ajudou a corrigir.

**Como validei:** executei todos os comandos e conferi as saídas nos logs e nos prints de `evidencias/`, comparei o comportamento com o PDF e com a documentação do SAM, e refiz a conta de custo passo a passo.
