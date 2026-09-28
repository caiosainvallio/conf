# AI workspaces: MacBook Air, MacBook Pro e servidor

Última validação: 2026-09-28.

Este documento descreve a arquitetura de workspaces de IA usada entre os dois Macs e o servidor, com foco em isolamento de contextos, túneis SSH, portas e recuperação após reboot.

> Este repositório é público. Não registrar aqui credenciais, tokens, e-mails de clientes, chaves SSH, IPs privados/Tailscale ou outros segredos.

## Objetivo

Permitir trabalhar no MacBook Air e no MacBook Pro ao mesmo tempo, usando os mesmos serviços do servidor, sem colisão de portas e com recuperação automática após reinicializações normais ou quedas abruptas.

## Arquitetura

### MacBook Pro

O Pro possui workspaces locais para os perfis e usa um ControlMaster SSH compartilhado para abrir o painel remoto no servidor.

- launcher Metropolis: `metropolis` -> `metropolis-tmux`
- launcher Hapvida: `hapvida` -> workspace Hapvida
- ControlMaster: `~/.ssh/cm/ai-server-pro`
- túnel reverso: `server:49390 -> pro:49390`
- `ai-server metropolis` abre o contexto Metropolis no servidor
- `ai-server hapvida` abre o contexto Hapvida no servidor

O `ai-server` reutiliza o ControlMaster quando ele já existe e o recria quando necessário.

### MacBook Air

O Air usa launchers próprios.

- `metropolis` -> `metropolis-air`
- `hapvida` -> `hapvida-air`

Para Metropolis:

- ControlMaster dedicado: `~/.ssh/cm/metropolis-cursor`
- túnel reverso: `server:49391 -> air:49390`
- o servidor recebe `CURSOR_BRIDGE_PORT=49391`

Assim, Pro e Air podem ficar ligados simultaneamente:

| Origem | Porta no servidor | Porta local |
| --- | ---: | ---: |
| MacBook Pro | 49390 | 49390 |
| MacBook Air | 49391 | 49390 |

As portas no servidor são diferentes, portanto não há disputa entre os dois Macs.

### Dependência específica do Hapvida no Air

O launcher `hapvida-air` atualmente usa:

```text
Air -> mac-tail -> MacBook Pro
```

Portanto, o perfil Hapvida no Air depende do MacBook Pro estar acessível. Isso é diferente do Metropolis no Air, que possui seu próprio túnel para o servidor.

Essa dependência é intencional no estado atual da configuração e deve ser lembrada ao diagnosticar indisponibilidade do Hapvida no Air.

## Perfis no servidor

Os contextos remotos permanecem isolados.

| Perfil | ai-memory |
| --- | ---: |
| Metropolis | 49374 |
| Hapvida | 49375 |

Os launchers do servidor carregam as configurações específicas de cada contexto, incluindo diretórios de dados e perfis das ferramentas de IA.

## Recuperação após reboot

### MacBook Pro

As sessões tmux e o socket SSH não sobrevivem ao reboot, o que é esperado.

Após login:

1. executar `metropolis` ou `hapvida`;
2. o launcher recria a sessão tmux se ela não existir;
3. o painel `server` executa o `REMOTE_CMD`;
4. o `ai-server` verifica o ControlMaster;
5. se necessário, recria `~/.ssh/cm/ai-server-pro`;
6. recria o reverse tunnel `49390`;
7. abre o contexto remoto correspondente.

O `ai-server` possui retry automático quando a porta remota ainda está ocupada por uma sessão SSH stale:

- 7 tentativas;
- intervalo de 20 segundos;
- `ExitOnForwardFailure=yes`;
- `ServerAliveInterval=60`;
- `ServerAliveCountMax=3`.

### MacBook Air

O `metropolis-air` possui a mesma proteção de retry para a porta remota `49391`:

- 7 tentativas;
- intervalo de 20 segundos;
- `ExitOnForwardFailure=yes`;
- `ServerAliveInterval=60`;
- `ServerAliveCountMax=3`.

O launcher recria o ControlMaster e o túnel quando necessário.

O `hapvida-air` não cria esse reverse tunnel; ele depende da conexão `mac-tail` com o Pro.

### Servidor

O SSH do servidor está configurado para remover sessões mortas:

```text
ClientAliveInterval 60
ClientAliveCountMax 2
```

Isso evita que uma sessão SSH antiga mantenha `49390` ou `49391` ocupada indefinidamente após um reboot abrupto de um Mac.

Na prática, uma sessão morta deve ser removida aproximadamente dentro da janela de dois minutos. Os retries dos launchers cobrem essa janela.

## O que significa "voltar automaticamente"

A configuração é autocurável depois que o sistema operacional e a rede/Tailscale voltam a ficar disponíveis.

Ela não significa que as sessões tmux permanecem vivas durante um reboot. Elas são recriadas pelos launchers.

Também existem duas distinções importantes:

1. **login do usuário no macOS**: componentes em espaço de usuário dependem da sessão do usuário estar disponível;
2. **energia física do servidor**: serviços podem voltar automaticamente depois que o Ubuntu inicia, mas ligar fisicamente o servidor após retorno de energia é uma responsabilidade separada de BIOS/AC Recovery/UPS.

## Testes realizados

Em 2026-09-28 foi feito reboot real do MacBook Pro.

O teste confirmou:

- diretório `~/.ssh/cm` vazio imediatamente após reboot;
- recriação das sessões Metropolis e Hapvida;
- recriação automática de `ai-server-pro`;
- restauração de `server:49390 -> pro:49390`;
- manutenção simultânea de `server:49391 -> air:49390`;
- funcionamento dos contextos remotos após a limpeza de uma sessão SSH stale;
- configuração do servidor para detectar sessões mortas;
- retry automático adicionado ao Pro e ao Air.

Também foi validado que os dois listeners podem coexistir no servidor:

```text
127.0.0.1:49390  # MacBook Pro
127.0.0.1:49391  # MacBook Air
```

## Smoke tests

### No Pro

```bash
metropolis
hapvida
ls -la ~/.ssh/cm
```

O socket esperado é:

```text
ai-server-pro
```

Para validar o contexto:

```bash
ctx
```

### No Air

```bash
metropolis
hapvida
```

Para validar o túnel do Metropolis no servidor:

```bash
ssh server-tail 'ss -ltn | grep -E ":4939[01] "'
```

### No servidor

Os dois listeners devem poder coexistir:

```text
127.0.0.1:49390
127.0.0.1:49391
```

Para conferir a política de sessões mortas:

```bash
sudo sshd -T | grep -E 'clientaliveinterval|clientalivecountmax'
```

Esperado:

```text
clientaliveinterval 60
clientalivecountmax 2
```

## Diagnóstico rápido

### Erro: `remote port forwarding failed for listen port 49390`

Normalmente indica uma sessão SSH stale do Pro ainda segurando a porta no servidor.

O launcher deve aguardar e tentar novamente. O `sshd` do servidor deve remover a sessão morta automaticamente.

### Erro equivalente em `49391`

Aplica-se ao túnel Metropolis do Air.

### Hapvida no Air não abre

Verificar primeiro se o MacBook Pro está ligado e acessível por `mac-tail`, porque o launcher do Air depende dele.

## Convenção de comandos

Nos Macs:

```text
metropolis -> abre o workspace/tmux de IA
hapvida    -> abre o workspace/tmux de IA
```

No servidor:

```text
metropolis -> Metropolis CLI
```

A Metropolis CLI não deve ser instalada nos Macs apenas para fornecer o comando `metropolis`; nos Macs esse nome é reservado ao launcher do workspace.
