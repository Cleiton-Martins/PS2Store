POST-MORTEM: INDISPONIBILIDADE PARCIAL E DEGRADAÇÃO DE PERFORMANCE NA RELEASE v2.0.0 (BLACK FRIDAY)

INFORMAÇÕES GERAIS
Serviço / Aplicação: Retrô 128 (https://cleiton-martins.github.io/PS2Store/)
Repositório: https://github.com/Cleiton-Martins/PS2Store
Versão Afetada: v2.0.0 (merge PR #1 - release/2.0)
Versão de Mitigação: v1.0.0 (commit e802eb2 via Rollback)
Versão de Resolução: v2.0.1 (hotfix/2.0.1)
Severidade: Alta (S1 - Catálogo incompleto, ativos quebrados e esgotamento de recursos no cliente)
Status: Resolvido e Estabilizado

RESUMO EXECUTIVO

Durante o lançamento da campanha de Black Friday (v2.0.0), a loja Retrô 128 apresentou três falhas simultâneas em produção: quebra de 100% das imagens de capa do catálogo, interrupção da renderização da vitrine (exibindo apenas 3 dos 12 jogos disponíveis) e consumo severo de CPU e memória RAM no navegador dos usuários.

Seguindo os procedimentos de resposta a incidentes (SRE), o serviço foi mitigado rapidamente por meio de um rollback para a tag estável v1.0.0 via pipeline parametrizado no GitHub Actions. Após a restauração da operação, as causas raiz foram investigadas em ambiente local, corrigidas na branch hotfix/2.0.1 e publicadas com sucesso em produção.

LINHA DO TEMPO E MÉTRICAS SRE

Início do Incidente (14:51): Conclusão do deploy automático da v2.0.0 após o merge da branch release/2.0 (#1).

Detecção (14:55): Constatação visual de capas quebradas (404), vitrine truncada (3/12 jogos) e lentidão na aba.

Mitigação via Rollback (~14:56 / 20s de pipeline): Execução manual do workflow Deploy no GitHub Pages apontando para a tag v1.0.0 (e802eb2).

Investigação (Etapa 9): Reprodução local da v2.0.0, inspeção via DevTools (Network, Console, Task Manager) e análise de logs.

Resolução Definitiva (v2.0.1): Merge da branch hotfix/2.0.1, publicação da tag v2.0.1 e validação completa em produção.

Métricas:

MTTD (Tempo Médio para Detectar): ~4 minutos

MTTR (Tempo Médio para Restaurar): < 5 minutos (a utilização de tags imutáveis permitiu restaurar a produção em 20s de execução de pipeline antes mesmo da análise de código).

COMPARATIVO DE TELEMETRIA

Jogos exibidos na vitrine:

v1.0.0 (Estável): Mostrando 12 de 12 jogos

v2.0.0 (Incidente): Mostrando 3 de 12 jogos

v2.0.1 (Hotfix): Mostrando 12 de 12 jogos

Carregamento de capas:

v1.0.0 (Estável): 12 de 12 (HTTP 200)

v2.0.0 (Incidente): 0 de 12 (HTTP 404)

v2.0.1 (Hotfix): 12 de 12 (HTTP 200)

Erros no Console:

v1.0.0 (Estável): 0

v2.0.0 (Incidente): TypeError contínuo

v2.0.1 (Hotfix): 0

Consumo de Memória (RAM):

v1.0.0 (Estável): 61.908K (~60,4 MB)

v2.0.0 (Incidente): 616.124K (~601,7 MB - aumento de 900%)

v2.0.1 (Hotfix): Estável (~60 MB)

Consumo de CPU (aba parada):

v1.0.0 (Estável): 0,1%

v2.0.0 (Incidente): 44,3% (443x maior)

v2.0.1 (Hotfix): 0,1%

Evento RECURSOS (Logs):

v1.0.0 (Estável): heap=2MB dom=188 atraso_loop=16ms

v2.0.0 (Incidente): atraso_event_loop_ms > 500ms e vazamento contínuo

v2.0.1 (Hotfix): Estável (historico_contador <= 60)

ANÁLISE DAS CAUSAS RAIZ E CORREÇÕES APLICADAS (v2.0.1)

4.1. Falha de Configuração - Erro HTTP 404 nas Capas

Origem: Commit 7b7df9a (RET-201) em config/app-config.json.

Causa Raiz: A propriedade pastaCapas foi alterada de "img/capas/" para "assets/capas/", caminho inexistente na estrutura do repositório, fazendo com que todas as requisições de imagem retornassem 404 Not Found.

Correção: Restauração do caminho correto "pastaCapas": "img/capas/" e atualização de "versao" para "2.0.1".

4.2. Exceção Não Tratada - Truncamento do Catálogo

Origem: Commit 30376ef (RET-202) em js/precos.js.

Causa Raiz: A função precoComDesconto(jogo, config) acessava promocao.percentual diretamente sem verificar se campanha.promocoes[jogo.id] estava definido. Ao renderizar o 4º item do catálogo (Silêncio em Morrow Hill, que não participava da promoção), ocorria TypeError: Cannot read properties of undefined (reading 'percentual'), interrompendo o loop de renderização.

Correção: Inclusão de verificação defensiva (if (!promocao) return jogo.preco;).

4.3. Esgotamento de CPU e Vazamento de Memória

Origem: Commit 0587aca (RET-203) ativando js/contador.js.

Causa Raiz:

A função iniciarContador() repassava campanha.intervaloContadorSegundos (1) diretamente para setInterval(), que espera milissegundos, disparando tick() centenas de vezes por segundo e elevando a CPU para 44,3%.

A cada execução de tick(), o código clonava toda a árvore DOM da vitrine (vitrine.cloneNode(true)) para dentro do array global historicoContador sem política de descarte, causando crescimento ilimitado de RAM (616.124K).

Correção: Multiplicação do intervalo por 1000 (1 execução por segundo) e implementação de limite circular de 60 entradas no buffer (if (historicoContador.length > 60) historicoContador.shift();).

LIÇÕES APRENDIDAS E PLANO DE AÇÃO PREVENTIVO

O que funcionou bem:

A marcação de releases com tags Git imutáveis (v1.0.0, v2.0.0, v2.0.1) permitiu isolamento claro entre versões.

O workflow parametrizado no GitHub Actions viabilizou o rollback imediato sem necessidade de reverter commits na main sob pressão.

A telemetria embarcada (observabilidade.js e logs.html) forneceu evidências objetivas de atraso_event_loop_ms, historico_contador e erros HTTP/JS.

Ações Preventivas:

Automação de Testes no CI (GitHub Actions): Adicionar etapa de validação pré-deploy para verificar sintaxe JSON, existência física dos caminhos configurados em app-config.json e testes unitários em precoComDesconto() (cobrindo jogos com e sem promoção).

Checklist de Revisão de Código (Code Review): Exigir validação explícita de unidades de tempo em temporizadores (setInterval / setTimeout), limites de memória em estruturas globais e tratamento de valores undefined antes do merge em main.

Ambiente de Pré-Produção (Staging): Validar branches release/* em ambiente de homologação antes da promoção para produção.