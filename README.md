# ⚙️ Estudo de Caso: Simulação da Rotina do Operador de Produção (TI)

Este repositório documenta uma simulação prática e realista do dia a dia de um **Operador de Produção de TI**, focado em engenharia de sustentação de sistemas, conformidade de janelas operacionais (SLAs) e resolução de incidentes críticos em ambiente Linux corporativo.

O projeto demonstra a capacidade analítica de monitorar ecossistemas em lote (Batch), realizar manutenções preventivas manuais e aplicar técnicas forenses de terminal para solucionar travamentos e retenções de barramento de servidores.

---

## 📋 Cenário A: Resolução de Atrasos na Esteira (Carga Urgente Manual)

Quando o monitoramento centralizado aponta ausência de dados gerando alarmes sucessivos em lote, cabe ao operador intervir se houver um plano de contingência documentado para não estourar a janela operacional da companhia.

1. **Configuração da Automação Contínua:** O gatilho temporal foi configurado no daemon do Cron para processar em intervalos de um minuto, garantindo escuta permanente no ambiente:

![Configuração do Agendador](assets/cron_config_minuto.png)

2. **Monitoramento de Rotina Ativo:** O monitoramento contínuo via `tail -f` exibe os ciclos varrendo a pasta sem encontrar novos lotes, gerando o status controlado de `[WARNING]`:

![Esteira em Estado de Espera Contínua](assets/monitoramento_continuo.png)

3. **Injeção e Resolução de SLA:** Uma carga emergencial (`carga_urgente.txt`) foi injetada manualmente no diretório de entrada. O robô em lote pescou o lote de forma autônoma, processou-o e gerou o log de auditoria `[SUCCESS]`, limpando a área de entrada:

![Mitigação de Incidente de Carga Atrasada](assets/injecao_sucesso_urgente.png)

---

## 🚨 Cenário B: Caça e Depuração de Processos Travados (Loops Operacionais)

Em cenários severos, queries pesadas de banco de dados ou loops internos podem congelar scripts em execução. O operador de produção precisa identificar e derrubar os gargalos para liberar o barramento do servidor.

1. **Simulação de Gargalo de Rede/Banco:** O script foi alterado via editor de terminal para reter o processo por 300 segundos (`sleep 300`), represando a fila de processamento intencionalmente:

![Configuração do Loop de Travamento no Script](assets/script_parametrizado_trava.png)

2. **Engenharia Forense de Processos (Derrubada com Kill):** Ao identificar o represamento e o acúmulo de instâncias concorrentes devido ao tempo de trava, executou-se uma busca profunda utilizando o utilitário de sistema `ps aux | grep`. Localizados os identificadores exclusivos, aplicou-se o sinal de encerramento controlado `kill -15` para purgar a fila de execução com segurança:

![Localização e Exclusão Cirúrgica de Processos Travados](assets/caca_processo_kill.png)

---

## 🧹 Encerramento do Turno & Governança de Infraestrutura

Uma das responsabilidades críticas de um Operador de TI na passagem e finalização de turnos é garantir a governança e o "decommissioning" (desativação) de rotinas e tarefas temporárias. Isso impede o desperdício de memória e o acúmulo de lixo técnico nos servidores de produção.

Ao concluir todas as validações práticas e testes de estresse das janelas batch, o ambiente foi purgado de forma segura através dos seguintes procedimentos:

1. **Auditoria Forense de Processos Ativos:** Foi realizada uma varredura profunda com o comando `ps aux | grep` para certificar que nenhuma instância zumbi ou travada ficou retida na memória do sistema operacional. O barramento de processos retornou totalmente limpo:

![Fila de Execução Sem Processos Retidos](assets/auditoria_processos_limpos.png)

2. **Desativação Total do Agendador Crontab:** Para garantir que a esteira batch não executasse fora da janela permitida, limpou-se permanentemente o arquivo de tarefas do daemon com o comando `crontab -r`. A listagem via `crontab -l` ratificou o desligamento seguro:

![Confirmação de Remoção do Crontab](assets/remocao_cron_sucesso.png)

Com estes procedimentos executados e devidamente evidenciados nos registros de auditoria, o turno de sustentação foi encerrado com sucesso, mantendo o ecossistema e o servidor em estado perfeitamente estável e limpo.





## 📸 Informações extras - Olhando para a Segurança da Informação

Existem vulnerabilidades e riscos operacionais graves nesse modelo de projeto que um agente de ameaça (ou até mesmo um funcionário insatisfeito com acesso ao servidor) poderia explorar para paralisar ou prejudicar gravemente o ambiente de produção da empresa.

Como o papel de um Operador de Produção envolve garantir a disponibilidade e a segurança do ambiente, entender esses pontos fracos ajuda a proteger o servidor. Abaixo estão os principais vetores de ataque e falhas de segurança presentes nesse cenário:


🚨 1. Ataque de Negação de Serviço Local (Inundação de Disco e Processos)

No laboratório, o script varre a pasta input e processa qualquer arquivo .txt que encontrar.

• A Ameaça: Um agente malicioso com acesso ao diretório de entrada poderia criar um script simples para gerar milhões de arquivos falsos de 0 bytes ou criar um único arquivo gigante de vários gigabytes de forma contínua.

• O Impacto: Isso causaria um "travamento por exaustão". O script em Bash abriria loops infinitos tentando ler os arquivos, consumindo 100% da CPU. Além disso, o arquivo de log (execucao_batch.log) cresceria descontroladamente até lotar completamente o disco do servidor (df -h), o que faria o sistema operacional travar e derrubaria o banco de dados.


🛡️ 2. Manipulação de Arquivos e Permissões Fracas (chmod / chown)

Para que o operador ou o agendador consigam manipular os arquivos, as pastas precisam de permissões de leitura e escrita.

• A Ameaça: Se as permissões das pastas input ou output forem configuradas de forma negligente (como um permissivo chmod 777, onde qualquer usuário do sistema pode ler, escrever e apagar), um agente de ameaça lateral poderia interceptar a carga de dados.

• O Impacto: O atacante poderia deletar arquivos legítimos de cobrança antes que o script os processasse, ou pior: modificar o conteúdo dos arquivos de transações para alterar valores financeiros, injetar dados falsos ou corromper a estrutura do texto, fazendo o processador batch quebrar por erro de sintaxe.


⚙️ 3. Injeção de Comandos (Command Injection)

Embora o script execute comandos simples como mv e basename, a manipulação direta de variáveis baseadas em nomes de arquivos externos esconde um risco.

• A Ameaça: Se o script em Bash usasse o conteúdo interno do arquivo de texto para executar comandos diretamente no terminal (como um comando eval ou ler um caminho de diretório de dentro do arquivo do cliente), teríamos uma falha de injeção.

• O Impacto: O atacante poderia batizar um arquivo ou inserir uma linha de texto contendo comandos maliciosos escondidos (ex: ; rm -rf /home/thiago-pr/output). Ao ler o arquivo, o script executaria o comando destrutivo com os privilégios do dono da aplicação, apagando todo o histórico de dados.


⏰ 4. Sequestro ou Sabotagem do Agendador (Crontab)

O arquivo do crontab controla o batimento cardíaco da produção da empresa.

• A Ameaça: Se o atacante conseguir acesso à conta do usuário que gerencia o agendador, ele não precisa alterar o script principal para causar dano.

• O Impacto: Ele pode simplesmente rodar um crontab -r (exatamente o comando de limpeza que usamos) no meio da janela de processamento para desativar toda a automação da empresa silenciosamente. Sem o operador perceber imediatamente, os arquivos se acumulariam na entrada, estourando o SLA de entrega dos clientes sem gerar alertas explícitos de erro no log.

🔒 Como mitigar esses riscos no mundo real?

Para evitar esses ataques, aplicam-se camadas de segurança chamadas de Hardening:

1. Princípio do Menor Privilégio: O operador e o script rodam com usuários separados e sem acesso de administrador (sudo).

2. Validação de Assinatura: O sistema só processa arquivos se eles vierem acompanhados de uma chave de segurança (Hash MD5/SHA256) que garante que o arquivo não foi alterado no caminho.

3. Cotas de Disco: Limita-se o tamanho máximo que a pasta input e a pasta de logs podem atingir, impedindo que um ataque lote o HD do servidor.

