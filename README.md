# Instalação do Claude Code (Windows → VPS)

## Objetivo

Disponibilizar o Claude Code (CLI de agente de codificação da Anthropic) para uso em estudos, documentando tanto a tentativa fracassada no PC pessoal (Windows 10, hardware de 2010) quanto a instalação bem-sucedida na VPS, incluindo o hardening de segurança feito antes de rodar qualquer ferramenta nova em uma máquina de produção.

---

## Motivação

O objetivo original era instalar o Claude Code no PC pessoal (Dell Inspiron 560s, Core 2 Duo E7500, 8GB RAM, Windows 10). Após horas de diagnóstico, o processo do Claude Code travava indefinidamente logo na inicialização, sem erro, sem uso de CPU/rede, de forma reproduzível em múltiplos métodos de instalação. Diante da impossibilidade de resolver localmente, a decisão foi migrar a ferramenta para a VPS já existente — mas isso exigiu, antes de qualquer instalação, corrigir lacunas de proteção reais que foram descobertas durante o processo (backup quebrado havia 7 dias, ausência de swap).

---

## Parte 1 — Tentativa no Windows (não resolvida)

### Sintoma

O comando `claude --version` (e qualquer outro comando do Claude Code) travava indefinidamente na mensagem `Setting up Claude Code...`, sem avançar, sem erro, com uso de CPU próximo de zero e nenhuma conexão de rede aberta pelo processo.

### Hipóteses testadas e descartadas

| Hipótese | Teste realizado | Resultado |
|---|---|---|
| Redirect HTTP não seguido | `curl.exe -L` no lugar de `irm` | Resolveu o download, não a execução |
| IPv6 mal configurado | Desabilitado IPv6 em todas as interfaces (Wi-Fi, Ethernet, vEthernet) + registro do Windows (`DisabledComponents`) + reboot | Sem efeito |
| Windows Defender (scan em tempo real) | `Add-MpPreference -ExclusionPath`, depois `Set-MpPreference -DisableRealtimeMonitoring $true` | Sem efeito |
| SmartScreen | Verificado já desativado | Sem efeito |
| Terminal legado (console antigo) | Testado em Windows Terminal (ConPTY) | Sem efeito |
| Caminho de execução (wrapper) | Testado binário nativo baixado manualmente, instalação via npm, e execução direta do `.js`/`.exe` sem wrapper | Todos convergem para o mesmo binário nativo, todos travam identicamente |
| Arquivo corrompido | Binário baixado novamente do zero | Sem efeito |
| Lock de processo/estado sujo | Reboot completo do sistema | Sem efeito |
| Incompatibilidade de CPU (AVX) | Descartada — `node --version`/`npm --version` respondem instantaneamente, provando que binários modernos rodam bem no processador | N/A |
| WSL2 como alternativa | `wsl --install` falhou: `HCS_E_HYPERV_NOT_INSTALLED`. Investigado via `systeminfo`: **"Conversão de Endereços de Segundo Nível: Não"** (SLAT/EPT ausente no hardware) | **Bloqueador definitivo de hardware** — Core 2 Duo E7500 (2010) não possui SLAT, WSL2 é inviável nesta máquina, sem solução por software |

### Conclusão

O travamento nunca foi explicado com certeza — permanece um comportamento anômalo, reproduzível, sem causa raiz identificada dentro do conjunto de ferramentas de diagnóstico disponíveis no Windows. Foi reportado ao suporte da Anthropic (support.claude.com) para investigação adicional. Decisão tomada: abandonar a instalação local e migrar para a VPS.

---

## Parte 2 — Hardening da VPS antes da instalação

Antes de instalar qualquer software novo em uma máquina hospedando produção (n8n, Evolution API, PostgreSQL, Redis para os clientes `dudu`, `nexflow` e `ultragas`), foi feita uma auditoria e correção de proteções.

### 2.1 Configuração de swap

A VPS (Hostinger KVM 1, 4GB RAM) não possuía swap configurada — risco real de o kernel encerrar processos de produção (OOM killer) em caso de pico de memória.

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Validado com `free -h`: `Swap: 2.0Gi` disponível.

### 2.2 Correção do backup (incidente documentado separadamente)

Auditoria revelou que a rotina de backup diário (rclone → Google Drive), implementada e validada em 02/08/2026 no repositório de infraestrutura da VPS, estava **falhando silenciosamente havia 7 dias** devido a um token OAuth expirado combinado com uma alteração incorreta de posse (`chown`) do arquivo de configuração do rclone. Corrigido via `chown` + `rclone config reconnect gdrive:` + execução manual de validação (documentado separadamente, no repositório de infraestrutura da VPS).

### 2.3 Usuário isolado para o Claude Code

Em vez de rodar o Claude Code sob o usuário administrativo (`edujr`, com sudo e acesso de leitura ao `rclone.conf` e ao histórico do shell), foi criado um usuário dedicado, sem privilégios administrativos:

```bash
sudo adduser claudecode
sudo passwd claudecode
```

**Validação do isolamento:**

```bash
groups claudecode
# claudecode : claudecode users   (sem "sudo" no grupo — confirmado)

sudo -u claudecode ls /etc/easypanel/projects/
# dudu  nexflow  ultragas   → consegue LISTAR (permissão 755 da pasta, não é possível restringir sem
# arriscar quebrar o EasyPanel)

sudo -u claudecode touch /etc/easypanel/projects/dudu/teste-permissao.txt
# touch: Permission denied   → confirmado: NÃO consegue escrever/apagar
```

**Limitação aceita e registrada:** o usuário `claudecode` consegue enxergar os *nomes* de pastas/arquivos dos clientes (por causa da permissão `755` padrão do EasyPanel em `/etc/easypanel/projects/`), mas não tem capacidade alguma de leitura de conteúdo, escrita ou exclusão. O risco residual foi considerado aceitável diante do ganho de isolamento.

---

## Parte 3 — Instalação do Claude Code na VPS

### 3.1 Node.js (instalado como `edujr`, requer sudo)

```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install nodejs -y
```

Validado: `node --version` → `v24.19.0`, `npm --version` → `11.17.0`.

### 3.2 Claude Code (instalado como `claudecode`, sem sudo)

Como o usuário `claudecode` não possui permissão de escrita no diretório global do npm (`/usr/lib/node_modules`), foi configurado um prefixo próprio dentro da home do usuário:

```bash
sudo -iu claudecode

mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

npm install -g --allow-scripts=@anthropic-ai/claude-code @anthropic-ai/claude-code
```

Validado: `claude --version` → `2.1.241 (Claude Code)`, sem nenhum travamento (confirmando que o problema no Windows era específico daquele ambiente, não do Claude Code em si).

### 3.3 Autenticação

Testada a opção de assinatura (Pro/Max) — não aplicável, pois a conta é do plano gratuito do Claude.ai, que não inclui acesso ao Claude Code.

Optado pela opção **API Console (billing por uso)**:

1. Conta criada em console.anthropic.com
2. Créditos pré-pagos adicionados ($5) — **sem** auto-reload ativado, garantindo que o gasto nunca excede o valor carregado manualmente
3. API Key gerada em Console → API Keys
4. `claude` executado na VPS → login via fluxo OAuth no navegador (não via colar direto da API key) → código de autorização copiado de volta ao terminal
5. Confirmação de confiança na pasta de trabalho (`/home/claudecode`)

### Ferramentas de "proxy gratuito" avaliadas e descartadas

Durante a pesquisa por alternativas de custo zero (9Router, OmniRoute, OpenRouter "free tier", DeepClaude e ferramentas similares), todas foram descartadas pelos mesmos motivos:

- Não entregam o modelo Claude de fato — roteiam para modelos de terceiros (DeepSeek, GLM, Kimi) com o nome "Claude Code" mantido na interface
- Uma delas (OmniRoute) possui CVE de bypass de autenticação documentado
- Rodar um proxy não-oficial de terceiros na mesma VPS que hospeda dados de clientes reais foi julgado um risco desnecessário
- Uso desse tipo de ferramenta pode violar os Termos de Serviço da Anthropic

Decisão: usar exclusivamente o canal oficial (API Console), aceitando o custo de poucos dólares por mês para uso de estudo.

---

## Parte 4 — Otimização de custo

Boas práticas adotadas para reduzir consumo de tokens:

- **Modelo padrão trocado de Opus 5 para Sonnet** via `/model` — sessão inicial em Opus custou $0,08 (majoritariamente cache write do contexto inicial); Sonnet é significativamente mais barato para o mesmo tipo de tarefa
- `/clear` entre tarefas não relacionadas, para não reprocessar histórico irrelevante
- `/compact` em sessões longas e contínuas
- Preferência por sessões longas em vez de múltiplas sessões curtas (cada sessão nova paga um "custo de entrada" de carregamento de contexto)
- `/cost` usado para acompanhar gasto em tempo real dentro da sessão

---

## Como validar a configuração

- ✅ `free -h` confirma 2GB de swap ativos
- ✅ `sudo crontab -l` + `sudo tail /var/log/nexflow-backup.log` confirmam backup diário funcionando novamente
- ✅ `groups claudecode` confirma ausência de privilégio sudo
- ✅ Teste de escrita (`touch`) confirma que `claudecode` não consegue alterar dados de clientes
- ✅ `claude --version` executa sem erro
- ✅ Sessão autenticada com sucesso via Console/API billing

---

## Lições aprendidas

- **Um travamento sem erro, sem log, sem uso de recursos é o mais difícil de diagnosticar** — depois de esgotadas todas as hipóteses de rede, terminal, antivírus e caminho de execução no Windows, a causa raiz permaneceu desconhecida. Nem todo problema tem solução viável dentro do tempo razoável de diagnóstico; reportar ao suporte oficial e mudar de ambiente foi a decisão certa, não uma desistência.
- **`Test-NetConnection` e `Resolve-DnsName` podem "funcionar" e ainda assim não refletirem o comportamento real de um processo Node.js** — a lógica de resolução de rede do Node é própria e pode divergir do que o sistema operacional reporta.
- **Hardware antigo tem limitações de baixo nível que nenhuma configuração resolve** — a ausência de SLAT/EPT no processador é um limite físico do silício, não algo ajustável via BIOS ou software (diferente de "virtualização desabilitada", que é comum e resolvível).
- **"Rodar em outra máquina" (a VPS) resolveu em minutos o que não foi resolvido em horas no ambiente problemático** — uma vez identificado que o problema era ambiental (não do software em si, confirmado pelo funcionamento imediato do mesmo pacote no Linux), migrar foi mais eficiente do que continuar depurando o Windows.
- **Antes de instalar qualquer ferramenta nova em uma máquina de produção, auditar o que já existe.** A auditoria pré-instalação revelou um backup quebrado havia 7 dias — um problema sério, encontrado por acaso, não porque estava sendo procurado.
- **Isolamento de usuário reduz risco mas não elimina 100%** — permissões herdadas de outras ferramentas (neste caso, o EasyPanel) podem manter uma brecha de leitura mesmo com um usuário corretamente configurado sem sudo. Vale documentar a limitação residual em vez de assumir isolamento perfeito.
- **Ofertas de "Claude Code de graça" via proxies de terceiros são, na prática, ofertas de "outro modelo com o nome Claude Code"** — vale desconfiar sistematicamente desse tipo de ferramenta, especialmente em máquinas com dados sensíveis.
- **Cobrança de API é genuinamente pré-paga** (confirmado na documentação oficial): sem auto-reload ativado, o gasto nunca ultrapassa o crédito comprado manualmente.
- **O modelo padrão (Opus) pode não ser a melhor escolha para uso de estudo** — o custo por token é significativamente maior que o Sonnet, sem necessariamente entregar benefício proporcional para tarefas simples.

---

*Documentação registrada em 23/08/2026.*
