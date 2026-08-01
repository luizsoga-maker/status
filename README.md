# status

Monitor de uptime da Outlet das Caixas. Um workflow do GitHub Actions checa os
serviços públicos a cada 5 minutos e avisa por e-mail quando algo sai do ar.

## O que é monitorado

| Serviço | Endereço | Critério |
| --- | --- | --- |
| Loja | `outletdascaixas.com.br/br` | fora do ar só em falha de rede/timeout ou HTTP >= 500 |
| Admin | `admin.outletdascaixas.com.br/health` | precisa responder HTTP 200 |
| API de frete | `frete.outletdascaixas.com.br/api/health` | precisa responder HTTP 200 |
| Painel de frete | `frete.outletdascaixas.com.br/` | precisa responder HTTP 200 |

A loja fica atrás do proxy da Cloudflare com Bot Fight Mode ligado. Um challenge
(403/503 vindo do edge) significa que a borda está viva e a loja atende gente de
verdade, então isso **não** é queda. Já os códigos 520-526 querem dizer "a origem
não respondeu" e entram no critério de HTTP >= 500.

Cada checagem usa timeout de 10 segundos e faz até 2 tentativas espaçadas em 5
segundos, para não abrir incidente por causa de uma oscilação de rede.

## Como funciona o alerta

O estado da queda mora nas issues deste repositório, com a label `incidente`:

- **Algo caiu e não há incidente aberto** → o monitor abre uma issue
  `🔴 Fora do ar: ...` com o detalhe de cada serviço, horário em UTC e em
  Brasília, e uma menção que dispara e-mail. O run também falha de propósito,
  porque a falha do workflow gera um segundo e-mail nativo do GitHub.
- **Algo caído e a issue já está aberta** → o run passa em verde. Sem isso
  chegaria um e-mail a cada 5 minutos durante toda a queda.
- **Tudo voltou e havia incidente aberto** → o monitor comenta `🟢 Recuperado`
  com a duração aproximada da queda e fecha a issue.
- **Tudo no ar e sem incidente** → run verde, silencioso.

Todo run escreve no resumo (*step summary*) uma tabela com serviço, status e
latência, esteja tudo no ar ou não.

## Como testar

O workflow aceita execução manual com o input `simular_queda`. Quando marcado,
a URL da loja é trocada por um host inexistente, o que exercita o caminho de
alerta de ponta a ponta sem derrubar nada de verdade.

Pela interface: aba **Actions** → workflow **Uptime** → **Run workflow** →
marque `simular_queda`.

Pela linha de comando:

```bash
# checagem normal
gh workflow run uptime.yml

# simular queda da loja (deve abrir a issue de incidente e falhar o run)
gh workflow run uptime.yml -f simular_queda=true

# rodar normal de novo: o monitor comenta e fecha a issue
gh workflow run uptime.yml
```

## Notas

- Roda em `ubuntu-latest` só com `bash`, `curl` e o `gh` CLI. Sem `npm` e sem
  actions de terceiros — nada de cadeia de suprimentos para auditar.
- Usa apenas o `GITHUB_TOKEN` do próprio run, com permissão de escrita em issues
  e leitura de conteúdo. Não há segredo configurado no repositório.
- O agendamento do GitHub Actions é melhor-esforço: em horários de pico um run
  pode atrasar alguns minutos.
