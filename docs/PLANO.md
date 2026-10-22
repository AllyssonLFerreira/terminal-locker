# Terminal Locker: Requisitos, Arquitetura e Plano de Ação

> App pessoal para Android que se disfarça de terminal e guarda fotos, vídeos e apps.
> Cada cofre só abre com o seu comando secreto.
>
> **Versão do documento:** 1.0 · **Data:** 2026-10-22 · **Aparelho piloto:** Samsung Galaxy S24 (256 GB)

---

## Sumário

1. [Visão geral](#1-visão-geral)
2. [Decisões fechadas](#2-decisões-fechadas)
3. [Requisitos funcionais](#3-requisitos-funcionais)
4. [Requisitos não funcionais](#4-requisitos-não-funcionais)
5. [Arquitetura](#5-arquitetura)
6. [Modelo criptográfico](#6-modelo-criptográfico)
7. [Chave de recuperação e modo root](#7-chave-de-recuperação-e-modo-root)
8. [Catálogo de comandos](#8-catálogo-de-comandos)
9. [Estética e design system](#9-estética-e-design-system)
10. [Ícone](#10-ícone)
11. [Estrutura do repositório](#11-estrutura-do-repositório)
12. [Plano de ação por fases](#12-plano-de-ação-por-fases)
13. [Estratégia de testes](#13-estratégia-de-testes)
14. [Riscos e mitigações](#14-riscos-e-mitigações)
15. [Pontos em aberto](#15-pontos-em-aberto)

---

## 1. Visão geral

O **Terminal Locker** abre como um terminal comum. Os comandos de disfarce (`ls`, `date`, `whoami`…) funcionam como esperado. Um comando específico de cada cofre revela o conteúdo escondido:

```
terminal ──► comando ──► correto? ──► sim ──► abre cofre (fotos | vídeos | apps)
                                   └─► não ──► "command not found" (sem nenhuma pista)
```

**Objetivos:**
- Esconder fotos e vídeos da galeria, guardando-os criptografados.
- Esconder apps do launcher e do sistema usando um **Work Profile**.
- Não dar nenhuma pista a terceiros: o app parece e se comporta como um terminal de verdade.

**Fora do escopo:**
- Proteção contra root, análise forense com o aparelho desbloqueado ou malware com privilégio de sistema.
- Sincronização com a nuvem.
- Publicação na Play Store. A distribuição será por APK, via sideload.

---

## 2. Decisões fechadas

| Tema | Decisão |
|---|---|
| Autenticação | **Somente o comando secreto** definido no setup. Sem PIN, biometria ou senha do sistema |
| Cofres | **3 cofres independentes**, cada um com seu comando: `fotos`, `videos`, `apps` |
| Ocultar apps | **Work Profile** (Managed Profile, com o app como *profile owner*) |
| Disfarce | Ícone e nome **"Terminal"** |
| Recuperação | **Par de chaves X25519** gerado em desenvolvimento. A chave privada (`KEY`) fica no repositório e é usada no **modo root** (`su`) |
| Estética | Terminal moderno, com cartões arredondados e temas claro e escuro sobre fundo em degradê (ver §9) |
| Aparelho piloto | Samsung Galaxy S24, One UI 7/8 (Android 15/16) |
| Stack | Kotlin + Jetpack Compose |

---

## 3. Requisitos funcionais

### RF01: Terminal (disfarce)
- **RF01.1** Prompt no formato `user@s24:~ ❯`, fonte monoespaçada, histórico rolável.
- **RF01.2** Comandos de disfarce funcionais: `help`, `ls`, `cd`, `pwd`, `date`, `echo`, `clear`, `whoami`, `uptime`, `uname`, `history`, `neofetch`.
- **RF01.3** Sistema de arquivos **fictício** e coerente para `ls` e `cd` (pastas `docs/`, `projects/`, `notes.txt`…). Nunca mostra dados reais.
- **RF01.4** Comando desconhecido responde `zsh: command not found: <cmd>`.
- **RF01.5** Histórico navegável (botões ↑/↓ na barra de teclas extras) com **autocomplete só dos comandos de disfarce**.
- **RF01.6** Comandos secretos **nunca** entram no histórico, nem na memória persistente, nem no log.
- **RF01.7** Barra de teclas extras acima do teclado: `Tab`, `↑`, `↓`, `Esc`, `Ctrl`, `|`, `-`, `/`, `~`.

### RF02: Comandos secretos
- **RF02.1** Cada cofre tem um comando livre definido no setup, por exemplo `sudo mount /dev/fotos -p <senha>`. O `-p <senha>` é só um marcador: a senha é você quem escolhe.
- **RF02.2** **Mínimo de 12 caracteres**, e o mesmo comando não pode servir para dois cofres.
- **RF02.3** O comando inteiro é a credencial. O app guarda apenas os dados para derivar a chave com Argon2id (salt e parâmetros), nunca o texto.
- **RF02.4** Toda linha digitada é testada contra os 3 cofres em **tempo constante**: o app sempre roda o mesmo cálculo, acerte ou erre, para que o tempo de resposta não revele nada.
- **RF02.5** Para validar sem deixar padrão óbvio, uma linha só é testada como segredo se tiver 12 caracteres ou mais. Linhas menores são tratadas direto como comando comum.
- **RF02.6** Proteção contra força bruta: depois de 5 erros seguidos de linhas candidatas, cada nova tentativa espera mais (1 s, 2 s, 4 s… até 5 min). A espera aparece como "processando", sem mensagem de erro.

### RF03: Setup inicial (primeira execução)
- **RF03.1** O wizard roda **dentro do terminal**, com perguntas no estilo `read -p`.
- **RF03.2** Define os 3 comandos, cada um digitado duas vezes para confirmar.
- **RF03.3** Cria o Work Profile. O app conduz o fluxo de provisionamento do sistema.
- **RF03.4** Gera as chaves (DEKs) dos cofres e as protege com o comando (Argon2id) e com a chave pública de recuperação embutida (§7).
- **RF03.5** No fim, mostra um resumo e limpa a tela. A partir daí, o app abre sempre como terminal comum.

### RF04: Cofre de fotos
- **RF04.1** Importar pelo seletor de fotos do sistema (Photo Picker), com vários itens de uma vez.
- **RF04.2** Criptografar com AES-256-GCM em blocos (streaming) e gravar em `filesDir/vault/fotos/<uuid>.bin`.
- **RF04.3** Remover o original da galeria com `MediaStore.createDeleteRequest`, que pede uma confirmação do sistema para cada lote.
- **RF04.4** Miniaturas também criptografadas, em cache próprio.
- **RF04.5** Grade de miniaturas, visualização em tela cheia com zoom e gestos, e álbuns.
- **RF04.6** Exportar: devolver um item para a galeria (`Pictures/Restored/`).
- **RF04.7** Capturar direto da câmera para o cofre, sem passar pela galeria.

### RF05: Cofre de vídeos
- **RF05.1** Mesmas funções do RF04, para vídeos.
- **RF05.2** Reprodução com Media3/ExoPlayer, usando um `DataSource` que descriptografa sob demanda. Nenhum arquivo descriptografado é gravado em disco.
- **RF05.3** Suporte a pular para qualquer trecho do vídeo, graças à criptografia em blocos com índice.

### RF06: Cofre de apps (Work Profile)
- **RF06.1** Listar os apps instalados **dentro do Work Profile**.
- **RF06.2** Instalar um app no perfil: clonar um app do perfil pessoal (`installExistingPackage`, quando aplicável) ou abrir a Play Store do perfil.
- **RF06.3** Ao abrir o cofre: tirar o perfil da pausa (`requestQuietModeEnabled(false)`), reexibir os apps (`setApplicationHidden(false)`) e listá-los.
- **RF06.4** Abrir o app escolhido (`run <app>` ou toque na lista).
- **RF06.5** Ao fechar (`exit`, timeout, tela apagada ou app em segundo plano): ocultar tudo de novo e **pausar o perfil**.
- **RF06.6** Marcar quais apps ficam "sempre ocultos" e quais podem ficar visíveis.

### RF07: Sessão e bloqueio
- **RF07.1** Auto-lock ao ir para segundo plano, ao apagar a tela ou depois de N minutos sem interação (configurável, padrão 2 min).
- **RF07.2** `exit` ou `lock` fecha o cofre na hora.
- **RF07.3** Ao fechar, as chaves são zeradas da memória e o terminal volta ao estado inicial, com um histórico de disfarce.
- **RF07.4** Um cofre aberto por vez.

### RF08: Modo root (recuperação)
Detalhado em §7.

### RF09: Recursos extras (v2)
- **RF09.1** Comando de pânico: fecha tudo e mostra um terminal limpo. Opcionalmente, abre um cofre falso (*decoy*).
- **RF09.2** Foto do intruso depois de X tentativas inválidas.
- **RF09.3** Backup manual criptografado (`.tlbak`) para armazenamento externo.

---

## 4. Requisitos não funcionais

| ID | Requisito | Critério de aceite |
|---|---|---|
| RNF01 | Criptografia em repouso | Nenhum byte de mídia em texto puro no disco (verificável via `adb`) |
| RNF02 | Sem vazamento visual | `FLAG_SECURE` em todas as telas: print bloqueado e prévia preta em "recentes" |
| RNF03 | Offline | **Sem** `android.permission.INTERNET` no manifest |
| RNF04 | Sem backup automático | `allowBackup="false"` e `dataExtractionRules` excluindo tudo |
| RNF05 | Desempenho do terminal | Resposta a comandos de disfarce em menos de 50 ms |
| RNF06 | Desempenho do KDF | Argon2id levando de 0,5 a 1 s no S24 (calibrado no setup) |
| RNF07 | Galeria | Rolagem a 60 fps com mais de 1.000 itens |
| RNF08 | Vídeo | Início da reprodução em menos de 1 s para um arquivo de 1 GB |
| RNF09 | Compatibilidade | `minSdk 30`, `targetSdk 36`; validado no S24 |
| RNF10 | Disfarce | Nenhum texto, notificação ou nome de componente revela "locker" ou "vault" (o pacote também é neutro) |
| RNF11 | Memória | Chaves em `ByteArray` zeradas depois do uso; nunca guardadas em `String` |

---

## 5. Arquitetura

### 5.1 Visão em dois perfis

```
┌──────────── Perfil pessoal ────────────┐      ┌──────────── Work Profile ─────────────┐
│ Terminal (UI principal)                │      │ Terminal (cópia: profile owner)       │
│  ├─ TerminalEngine (parser/disfarce)   │      │  ├─ DeviceAdminReceiver               │
│  ├─ SecretMatcher (Argon2id)           │ IPC  │  ├─ AppHider (setApplicationHidden)   │
│  ├─ MediaVault (fotos/vídeos)          │ ───► │  ├─ ProfileInstaller                  │
│  └─ AppVaultClient ────────────────────┼──────┼─►└─ AppVaultService (bound, seguro)   │
└────────────────────────────────────────┘      └───────────────────────────────────────┘
```

- **Mesmo APK, dois papéis.** No perfil pessoal, o app é a interface. No Work Profile, ele é o *profile owner* que oculta e exibe os apps.
- **Comunicação entre os perfis:** um `Intent` explícito entre perfis (`CrossProfileIntentFilter` registrado pelo profile owner), com assinatura verificada. Uma alternativa é `CrossProfileApps.startActivity`.
- O perfil pessoal só envia **ordens** ("abrir cofre", "fechar cofre"). A autenticação acontece no perfil pessoal, e o pedido leva um token de sessão de curta duração.

### 5.2 Módulos Gradle

| Módulo | Responsabilidade |
|---|---|
| `:app` | Activity única, navegação, DI (Hilt), manifest |
| `:core:terminal` | Parser de linha, comandos de disfarce, sistema de arquivos fictício, histórico |
| `:core:crypto` | Argon2id, AES-GCM em streaming, sealed box (libsodium), wrapping de chaves |
| `:core:design` | Tema, tokens, componentes Compose (TerminalCard, Chip, Prompt…) |
| `:feature:media` | Importar, exportar, galeria, player |
| `:feature:apps` | Cliente e serviço do Work Profile |
| `:feature:setup` | Wizard de primeira execução |
| `:feature:root` | Modo root (recuperação) |

### 5.3 Bibliotecas

| Uso | Biblioteca |
|---|---|
| UI | Jetpack Compose (BOM atual), Material 3 só como base |
| Crypto | `lazysodium-android` (Argon2id, sealed box X25519) + `javax.crypto` AES-GCM |
| Metadados | Room + SQLCipher (chave derivada da DEK de cada cofre) |
| Vídeo | Media3 ExoPlayer, com um `DataSource` customizado |
| Imagens | Coil 3, com um `Fetcher` customizado para os arquivos criptografados |
| DI | Hilt |
| Testes | JUnit5, Turbine, Robolectric, Compose UI Test, Macrobenchmark |

---

## 6. Modelo criptográfico

```
Comando do cofre ──Argon2id(salt, m=64MiB, t=3, p=1)──► KEK (32 B)
                                                          │
DEK do cofre (32 B aleatória) ◄──AES-GCM unwrap───────────┘
      │
      ├── criptografa arquivos  (AES-256-GCM, blocos de 64 KiB, nonce = prefixo‖contador)
      ├── criptografa miniaturas
      └── deriva chave do SQLCipher (HKDF "db")

DEK do cofre ──sealed_box(recovery.pub)──► recovery_blob   (só a KEY privada abre)
```

**Arquivo `vault.meta`, um por cofre, sem segredo em texto puro:**

```json
{
  "v": 1,
  "kdf": { "alg": "argon2id", "salt": "<b64>", "m": 65536, "t": 3, "p": 1 },
  "wrapped_dek": "<b64 AES-GCM(KEK, DEK)>",
  "recovery_blob": "<b64 sealed_box(recovery.pub, DEK)>",
  "recovery_pub_id": "<8 primeiros bytes do SHA-256 da pub>"
}
```

- **Validar um comando:** derivar a KEK e tentar desembrulhar `wrapped_dek`. Se a tag GCM bater, o comando está certo. Não existe hash do comando guardado em lugar nenhum.
- **Tempo constante:** cada linha candidata faz **3 derivações** (uma por cofre), mesmo que a primeira já tenha acertado.
- **Android Keystore:** guarda uma chave extra (`device_binding_key`, AES, sem exigir autenticação) que criptografa os `vault.meta`. Assim, copiar os arquivos para outro aparelho não basta para atacá-los.

---

## 7. Chave de recuperação e modo root

### 7.1 Geração (uma vez, em desenvolvimento)

```bash
./tools/gen-recovery-key.sh
# gera:
#   app/src/main/assets/recovery.pub   → entra no APK (pública; só tranca)
#   keys/recovery.key                  → a sua KEY (privada; NUNCA entra no APK)
#   keys/recovery.key.age              → (recomendado) versão cifrada com senha para commit
```

- O formato da KEY é `TLK1-` seguido da chave privada X25519 em Base64url, com cerca de 48 caracteres. É fácil de colar.
- O build tem uma **trava no Gradle** (`verifyNoPrivateKey`): ele falha se encontrar `recovery.key` ou o prefixo `TLK1-` dentro do APK ou AAB.

### 7.2 Guarda no repositório
- O repositório deve ser **privado**.
- **Recomendado:** commitar só `keys/recovery.key.age`, cifrado com `age -p`, e colocar `keys/recovery.key` no `.gitignore`.
- **Se preferir em texto puro**, como foi decidido: tudo bem, mas qualquer pessoa com acesso ao repositório consegue abrir os cofres.

### 7.3 Fluxo do modo root

```
user@s24:~ ❯ su
Password: ••••••••••••••••••••••••••••••••••••••••••••••••
root@s24:~ #
```

1. `su` pede a KEY com a entrada mascarada e aceitando colar.
2. O app deriva a chave pública a partir da KEY, compara com `recovery.pub` e tenta abrir os `recovery_blob`.
3. Se falhar, responde `su: Authentication failure`, com a mesma espera crescente do RF02.6.
4. A sessão root dura **5 min** sem uso ou até `exit`, e a área de transferência é limpa assim que a KEY é colada.

| Comando root | Função |
|---|---|
| `ls /vaults` | Lista cofres, quantidade de itens e tamanho |
| `passwd <fotos\|videos\|apps>` | Redefine o comando do cofre (gera novo salt e embrulha a DEK de novo) |
| `mount <cofre>` | Abre o cofre sem o comando dele |
| `export <cofre>` | Exporta tudo de volta para a galeria |
| `rekey` | Embrulha de novo todas as DEKs com uma nova `recovery.pub` (depois de atualizar o APK) |
| `wipe` | Apaga todos os cofres. Pede `yes` e depois a KEY de novo |
| `exit` | Sai do modo root |

### 7.4 Rotação (caso a KEY vaze)
1. Gerar um novo par com `gen-recovery-key.sh --rotate`.
2. Instalar o APK novo, que traz a nova `recovery.pub`.
3. Rodar `su` com a KEY **antiga** e depois `rekey`. As DEKs passam a ser protegidas pela nova chave pública.

---

## 8. Catálogo de comandos

### 8.1 Disfarce (modo normal)
| Comando | Saída |
|---|---|
| `help` | Lista apenas os comandos de disfarce |
| `ls [-la]`, `cd`, `pwd` | Navegam no sistema de arquivos fictício |
| `cat <arquivo>` | Conteúdo fictício (notas, README de projeto) |
| `date`, `uptime`, `whoami`, `uname -a` | Dados reais e inofensivos do sistema |
| `echo`, `clear`, `history` | Comportamento padrão (o histórico nunca mostra segredos) |
| `neofetch` | Arte ASCII com informações do aparelho |
| `<qualquer outro>` | `zsh: command not found: <cmd>` |

### 8.2 Secretos
| Comando | Efeito |
|---|---|
| `<comando-fotos>` | Abre o cofre de fotos |
| `<comando-videos>` | Abre o cofre de vídeos |
| `<comando-apps>` | Tira o Work Profile da pausa, reexibe e lista os apps |
| `su` | Modo root (§7) |

### 8.3 Dentro de um cofre
| Comando | Efeito |
|---|---|
| `ls` | Lista os itens (ou abre a grade visual) |
| `import` | Abre o seletor do sistema |
| `cam` | Captura direto para o cofre |
| `open <n>` | Abre o item `n` |
| `export <n\|all>` | Restaura para a galeria |
| `rm <n>` | Apaga o item, com confirmação |
| `run <app>` | (cofre de apps) Abre o app |
| `hide <app>` / `unhide <app>` | (cofre de apps) Gerencia a lista de apps ocultos |
| `exit` / `lock` | Fecha e volta ao disfarce |

---

## 9. Estética e design system

### 9.1 Referência
Fonte: [Collect UI: terminal-aesthetic](https://collectui.com/designs/terminal-aesthetic-ui-design-inspiration/704dfd52-9c19-4c13-8e81-3ba93219459c)

O que define o estilo:
- **Cartões de terminal arredondados**, com cabeçalho mostrando ícone, título em negrito, caminho em cinza e um *chip* à direita (`claude`, `node`).
- **Fundo em degradê suave** atrás dos cartões: rosa e lavanda no tema claro, vinho e ameixa no escuro.
- **"Semáforo" estilo macOS** (vermelho, amarelo e verde) no canto da janela.
- **Barra lateral de sessões** com título, horário ("now") e ícone de status (✓, ◉ em âmbar, spinner).
- **Saída com hierarquia:** marcador `●` para cada ação, `⎿` para a saída aninhada, cinza para detalhes e `❯` como prompt.
- **Destaque de sintaxe discreto:** azul para comandos, roxo para flags e valores, âmbar para avisos, verde para sucesso.
- **Rodapé de status** com contadores (`◉ 1`, `◐ 1`, `✓ 1`).
- O vídeo alterna entre os temas claro e escuro em transição suave.

### 9.2 Tokens de cor
> Valores amostrados do vídeo de referência e ajustados para contraste. Devem ser refinados no protótipo.

| Token | Claro | Escuro | Uso |
|---|---|---|---|
| `bg.gradient.start` | `#F0C9E5` | `#6D2D46` | Topo do degradê de fundo |
| `bg.gradient.end` | `#D7C096` | `#260F1C` | Base do degradê de fundo |
| `surface.sidebar` | `#F5EDE2` (90% opaco) | `#1E161C` (90% opaco) | Painel lateral ou gaveta |
| `surface.card` | `#FDFDFD` | `#242328` | Cartão de terminal |
| `surface.selected` | `#FFFFFF` | `#2F2C33` | Item selecionado da lista |
| `border.card` | `#00000014` | `#FFFFFF14` | Contorno de 1 dp |
| `text.primary` | `#1F1D22` | `#ECEAF0` | Texto principal |
| `text.secondary` | `#8A8690` | `#8C8893` | Caminhos, horários, saída aninhada |
| `chip.bg` | `#EDEBEF` | `#47454D` | Chips |
| `syntax.command` | `#4A6FD0` | `#7AA2F7` | `git`, `&&`, nomes de comando |
| `syntax.flag` | `#9B4FC0` | `#C58AF9` | `--size`, valores, constantes |
| `syntax.warn` | `#C08A20` | `#E5B45A` | `--short`, "needs you", status pendente |
| `syntax.success` | `#2E9A4A` | `#5FD07A` | `ready`, ✓, setas `➜` |
| `syntax.link` | `#4A6FD0` | `#7AA2F7` | URLs, sublinhado |
| `dot.red` | `#F05F51` | `#ED625F` | Semáforo: **lock** |
| `dot.yellow` | `#F7BB2E` | `#F6BD3A` | Semáforo: minimizar (vai para o launcher) |
| `dot.green` | `#26C93C` | `#36C24C` | Semáforo: nova aba |

### 9.3 Tipografia
| Papel | Fonte | Tamanho/peso |
|---|---|---|
| Terminal | **JetBrains Mono** (OFL, embutida) | 14 sp / 400. Negrito 700 para títulos `##` e verbos ("Ran") |
| UI (barra lateral, cabeçalhos, chips) | **Inter** (OFL, embutida) | 14 sp / 600 nos títulos, 12 sp / 400 nos metadados |
| Altura de linha do terminal | 1,45× | |

### 9.4 Forma, espaçamento e movimento
- **Raio:** 14 dp nos cartões, 10 dp nos itens da lista, 8 dp nos chips, 22 dp na "janela" externa.
- **Espaçamento:** grade de 4 dp. Margem interna de 16 dp nos cartões e 12 dp entre cartões.
- **Sombra:** suave e difusa (`elevation 2dp`, `ambient 8%`).
- **Cursor:** bloco `▌` piscando a cada 530 ms.
- **Saída:** efeito de digitação **só nos comandos de disfarce longos** (`neofetch`), a 4 ms por caractere. O resto aparece na hora.
- **Transição de tema:** crossfade de 300 ms, seguindo o tema do sistema por padrão.
- **Spinner:** `◐◓◑◒` durante a derivação do Argon2id, disfarçado de "processando".

### 9.5 Adaptação para o celular (S24, 6,2")

A referência é desktop. No celular, a adaptação fica assim:

```
┌────────────────────────────┐
│ ● ● ●   Terminal      ⊞  + │  ← barra de janela (semáforo funcional)
├────────────────────────────┤
│ ┌────────────────────────┐ │
│ │ ▤ zsh  ~/home    [zsh] │ │  ← cabeçalho do cartão (título, caminho, chip)
│ │                        │ │
│ │ ● Ran ls -la           │ │
│ │   ⎿ drwxr-xr-x docs/   │ │
│ │     -rw-r--r-- notes   │ │
│ │                        │ │
│ │ user@s24:~ ❯ ▌         │ │
│ └────────────────────────┘ │
│ [Tab][↑][↓][Esc][Ctrl][/]  │  ← barra de teclas extras
├────────────────────────────┤
│          teclado           │
└────────────────────────────┘
```

- **Barra lateral → gaveta** (arrastando da borda esquerda). Ela lista **"sessões"** de terminal fictícias (`~/projects`, `~/notes`) como disfarce. Quando um cofre está aberto, ele aparece ali como uma sessão (`/dev/fotos`).
- **Cofre de fotos/vídeos:** aparece como um cartão com cabeçalho `▤ mount /dev/fotos ~/vault` e chip `fs`. Dentro dele, a grade de miniaturas tem cantos de 8 dp e espaço de 4 dp entre elas.
- **Cofre de apps:** aparece como saída de terminal, com um app por linha e tocável:
  ```
  ● Listed 6 apps in /work
    ⎿ 1  WhatsApp        ✓ hidden
      2  Telegram        ✓ hidden
      3  Banco X         ◉ visible
  ```
- **Rodapé de status:** contadores no estilo da referência, todos fictícios no modo disfarce (`◉ 0  ◐ 1  ✓ 3`).

---

## 10. Ícone

**Conceito:** um terminal minimalista no mesmo idioma visual da referência.

```
┌─────────────────┐
│ ● ● ●           │   fundo: degradê #F0C9E5 → #C9B6F0 (claro) / cartão #242328
│                 │
│   ❯_            │   glifo: "❯_" em JetBrains Mono Bold, branco
│                 │
└─────────────────┘
```

- **Ícone adaptativo** (`mipmap-anydpi-v26`) em duas camadas. O *background* é o degradê do tema. O *foreground* é um cartão escuro arredondado com três pontinhos no topo e o glifo `❯_`.
- **Camada monocromática** (`<monochrome>`) para os ícones temáticos do Android 13+ e da One UI.
- **Nome no launcher:** `Terminal`.
- **Pacote neutro:** algo como `dev.alf.term`. Nada de "locker" ou "vault".
- Entregáveis: SVG mestre em `design/icon/`, mais o `VectorDrawable` gerado e o PNG de 512 px para referência.

---

## 11. Estrutura do repositório

```
terminal-locker/
├── docs/
│   └── PLANO.md                ← este documento
├── design/
│   ├── reference/              ← frames da referência estética
│   └── icon/                   ← SVG mestre do ícone
├── keys/
│   ├── .gitignore              ← ignora recovery.key (se optar pela versão .age)
│   └── recovery.key[.age]      ← a KEY
├── tools/
│   └── gen-recovery-key.sh
├── app/
├── core/{terminal,crypto,design}/
├── feature/{media,apps,setup,root}/
├── build-logic/                ← convention plugins + task verifyNoPrivateKey
└── settings.gradle.kts
```

---

## 12. Plano de ação por fases

> Estimativas para um desenvolvedor em ritmo de projeto pessoal (cerca de 10 h por semana).

### Fase 0: Fundação (semana 1)
| # | Tarefa | Entregável / critério de aceite |
|---|---|---|
| 0.1 | Criar o projeto Gradle multimódulo, version catalog e convention plugins | `./gradlew assembleDebug` OK |
| 0.2 | Configurar o manifest: sem INTERNET, `allowBackup=false`, `FLAG_SECURE` global | Verificado com `aapt dump permissions` |
| 0.3 | Script `gen-recovery-key.sh` e geração da KEY | `recovery.pub` em assets e `keys/recovery.key` criada |
| 0.4 | Task `verifyNoPrivateKey` ligada ao `assemble` | O build falha se a KEY for copiada para assets |
| 0.5 | CI local (lint, detekt, testes unitários) | `./gradlew check` verde |

### Fase 1: Terminal e design system (semanas 2–3)
| # | Tarefa | Critério |
|---|---|---|
| 1.1 | `:core:design`: tokens (§9.2), tipografia, tema claro e escuro, degradê | Tela de preview com todos os componentes |
| 1.2 | Componentes: `TerminalWindow`, `TerminalCard`, `Chip`, `TrafficLights`, `OutputLine (● / ⎿)`, `Prompt`, `ExtraKeysBar` | Previews no Compose nos 2 temas |
| 1.3 | `:core:terminal`: parser, registro de comandos, sistema de arquivos fictício, histórico | Testes unitários para cada comando de disfarce |
| 1.4 | Tela principal do terminal com teclado, rolagem e cursor | Uso fluido no S24 |
| 1.5 | Gaveta de sessões fictícias | Visual fiel à referência |
| 1.6 | Ícone adaptativo e monocromático | Aparece correto no launcher da One UI |

### Fase 2: Crypto e setup (semanas 4–5)
| # | Tarefa | Critério |
|---|---|---|
| 2.1 | `:core:crypto`: Argon2id, AES-GCM em streaming, sealed box, wrap e unwrap | Testes com vetores conhecidos e *round-trip* |
| 2.2 | Calibrar o Argon2id no S24 | De 0,5 a 1 s medido |
| 2.3 | `SecretMatcher` em tempo constante, mais a espera crescente | Teste: tempo de resposta igual para acerto e erro (±5%) |
| 2.4 | `vault.meta` e chave de *device binding* no Keystore | Os arquivos não abrem em outro aparelho |
| 2.5 | Wizard de setup no estilo terminal | Primeira execução completa e sem segredos no log |

### Fase 3: Cofre de fotos e vídeos (semanas 6–8)
| # | Tarefa | Critério |
|---|---|---|
| 3.1 | Importar via Photo Picker, criptografar em streaming e mostrar progresso no terminal | Lote de 100 fotos sem estouro de memória |
| 3.2 | Remover o original via `createDeleteRequest` | O item some da Galeria Samsung |
| 3.3 | Miniaturas criptografadas e `Fetcher` do Coil | Rolagem a 60 fps com mais de 1.000 itens |
| 3.4 | Visualização em tela cheia com zoom e gestos | |
| 3.5 | `DataSource` criptografado no Media3, com saltos no vídeo | Vídeo 4K de 1 GB começa em menos de 1 s |
| 3.6 | Exportar e restaurar | O arquivo volta para a galeria intacto (mesmo hash) |
| 3.7 | Captura pela câmera (`cam`) | Nada vai para o DCIM |
| 3.8 | Room + SQLCipher para metadados e álbuns | |

### Fase 4: Cofre de apps via Work Profile (semanas 9–11)
| # | Tarefa | Critério |
|---|---|---|
| 4.1 | **Spike:** provisionar o Work Profile no S24 ao lado da Pasta Segura | Os dois funcionando juntos (registrar o resultado aqui) |
| 4.2 | `DeviceAdminReceiver` e fluxo de provisionamento no setup | Perfil criado pelo próprio app |
| 4.3 | Comunicação entre perfis com assinatura verificada e token de sessão | Um pedido vindo de outro app é recusado |
| 4.4 | Ocultar/exibir (`setApplicationHidden`) e pausar o perfil (`requestQuietModeEnabled`) | Apps somem da gaveta da One UI |
| 4.5 | Instalar apps no perfil (clonar ou abrir a Play Store do perfil) | |
| 4.6 | Lista no terminal, `run`, `hide`, `unhide` | |
| 4.7 | Fechar automaticamente: ocultar e pausar | Ao apagar a tela, tudo volta a ficar oculto em menos de 1 s |

### Fase 5: Modo root e endurecimento (semanas 12–13)
| # | Tarefa | Critério |
|---|---|---|
| 5.1 | `su`: entrada mascarada, validação e sessão de 5 min | |
| 5.2 | `ls /vaults`, `passwd`, `mount`, `export`, `rekey`, `wipe` | Teste completo: esquecer o comando, recuperar e trocar |
| 5.3 | Limpeza da área de transferência e zeragem das chaves na memória | Conferido com um heap dump |
| 5.4 | Auditoria: logs, strings no APK, nomes de componentes | Nenhuma ocorrência de "vault" ou "locker" no APK de release |
| 5.5 | Build de release assinado e ofuscado (R8) | APK pronto para instalar |

### Fase 6: v2 (backlog)
- Comando de pânico e cofre falso.
- Foto do intruso.
- Backup criptografado `.tlbak`.
- Abas múltiplas no terminal (botão `+` e semáforo verde).
- Temas extras.

### Marcos
| Marco | Conteúdo | Semana |
|---|---|---|
| **M1** | Terminal de disfarce instalável com o visual final | 3 |
| **M2** | Setup e crypto prontos | 5 |
| **M3** | **MVP de mídia** (fotos e vídeos) | 8 |
| **M4** | Cofre de apps funcionando no S24 | 11 |
| **M5** | Release 1.0 com recuperação | 13 |

---

## 13. Estratégia de testes

| Camada | Ferramenta | Foco |
|---|---|---|
| Unitário | JUnit5 | Parser, comandos, crypto (vetores conhecidos e *round-trip*), lógica de espera |
| Propriedade | Kotest property | Criptografar e descriptografar arquivos de tamanhos e limites de bloco variados |
| UI | Compose UI Test | Fluxos: disfarce, comando certo, cofre aberto, lock |
| Instrumentado | AndroidJUnit + emulador com Work Profile | Ocultar/exibir apps e comunicação entre perfis |
| Desempenho | Macrobenchmark | Abertura do app, rolagem da galeria, início do vídeo |
| Segurança (manual) | `adb shell run-as`, `apktool`, `strings` | Nada em texto puro no disco, nenhuma chave privada ou string reveladora no APK |
| Aparelho real | S24 | Pasta Segura, Galeria Samsung, launcher da One UI, ícones temáticos |

---

## 14. Riscos e mitigações

| Risco | Impacto | Mitigação |
|---|---|---|
| KEY vazar pelo repositório | **Crítico**: todos os cofres expostos | Repositório privado, KEY cifrada com `age`, rotação via `rekey` (§7.4) |
| Esquecer o comando e perder a KEY | Perda total | Guardar a KEY em dois lugares: repositório e gerenciador de senhas |
| Desinstalar o app | Apaga os cofres | Aviso no setup e backup `.tlbak` (v2) |
| Conflito do Work Profile com a Pasta Segura ou o Knox | O cofre de apps não funciona | Spike 4.1 logo no início da fase. Plano B: Private Space ou modo launcher |
| A aba "Trabalho" aparece na gaveta | Revela que existe um perfil | Pausar e ocultar os apps quando o cofre fecha. Limite aceito sem root |
| Comando curto ou fácil de adivinhar | Ataque de força bruta | Mínimo de 12 caracteres, Argon2id pesado e espera crescente |
| Atualização da One UI mudar alguma API | Regressão | Testar em cada atualização grande. Sem APIs privadas da Samsung |
| Miniaturas antigas no cache da Galeria Samsung | Vazamento residual | Avisar o usuário. Limpar via Configurações → Galeria → Limpar cache |

---

## 15. Pontos em aberto

- [ ] **KEY em texto puro ou cifrada com `age`** no repositório? (Recomendado: `age`.)
- [ ] Nome do pacote (sugestão: `dev.alf.term`).
- [ ] O terminal deve seguir o tema do sistema ou ter um tema fixo?
- [ ] Resultado do spike 4.1 (Work Profile junto com a Pasta Segura no S24).
- [ ] Lista inicial de apps que vão para o cofre.
