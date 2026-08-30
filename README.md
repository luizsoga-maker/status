# status

Monitor de uptime da Outlet das Caixas. Um workflow do GitHub Actions checa os
serviços públicos e avisa por e-mail quando algo sai do ar. O agendamento pede
uma checagem a cada 5 minutos, mas o que o GitHub entrega é bem menos — veja
[Agendamento: o que o GitHub realmente entrega](#agendamento-o-que-o-github-realmente-entrega).

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


## Agendamento: o que o GitHub realmente entrega

O `cron` pede `4-59/5`, ou seja **288 execuções por dia**. Medido em 30/08/2026
sobre os 872 runs agendados que o repositório acumulou desde 02/08:

| Janela | Runs | Do programado | Intervalo médio | Maior buraco |
| --- | --- | --- | --- | --- |
| 24 h | 7 | 2,4 % | 3 h 36 | 5 h 42 |
| 48 h | 11 | 1,9 % | 4 h 02 | 6 h 43 |
| 7 dias | 140 | 6,9 % | 1 h 12 | 12 h 15 |

A queda tem data. Até 25/08 o monitor rodava de 38 a 60 vezes por dia; em 26/08
foram 25, e a partir de **27/08** despencou para 2 a 7 por dia. Os runs de
30/08 saíram às 01:46, 07:27, 13:05 e 17:22 UTC — espaçamento de cerca de seis
horas, sem nenhuma relação com os minutos que o `cron` pede.

**Consequência prática:** a loja pode ficar fora do ar por várias horas antes de
o monitor perceber. O alerta continua correto quando dispara; o que não existe é
a promessa de cinco minutos.

O comportamento é conhecido e documentado pelo GitHub: `schedule` é
melhor-esforço, executado numa fila compartilhada, e runs podem ser atrasados ou
descartados em períodos de carga alta — sem aviso e sem erro. Não há nada a
consertar dentro deste repositório: quem quiser detecção em minutos precisa de um
agendador fora do GitHub Actions.

## Manutenção: este repositório precisa receber commits

O GitHub **desabilita workflows agendados em repositório público após 60 dias sem
atividade no repositório**, silenciosamente. Um workflow desabilitado não roda e,
por consequência, não alerta — a falha é indistinguível de "está tudo no ar".

- Último commit antes deste: `f8a5a76`, de **02/08/2026**.
- Prazo que ele criava: por volta de **01/10/2026**.
- Este commit reinicia a contagem. **Próximo prazo: por volta de 29/10/2026.**

Qualquer commit serve, inclusive uma linha nesta seção. Alterar o `uptime.yml`
também reinicia a contagem, mas mexer no `cron` re-registra o agendamento e já
custou dois commits de conserto em 02/08 — para manutenção de rotina, prefira
editar apenas este README.

## Notas

- Roda em `ubuntu-latest` só com `bash`, `curl` e o `gh` CLI. Sem `npm` e sem
  actions de terceiros — nada de cadeia de suprimentos para auditar.
- Usa apenas o `GITHUB_TOKEN` do próprio run, com permissão de escrita em issues
  e leitura de conteúdo. Não há segredo configurado no repositório.
- O agendamento do GitHub Actions é melhor-esforço. A nota anterior aqui dizia
  "pode atrasar alguns minutos"; a medição de 30/08/2026 mostrou que o atraso é
  de **horas**. Ver a seção de agendamento acima.
