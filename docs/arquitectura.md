# SMSForwarder — Documento de Arquitetura

## 1. Visão Geral

O SMSForwarder é uma aplicação Android nativa (Kotlin) responsável por interceptar mensagens SMS recebidas num dispositivo e encaminhá-las diretamente para um canal externo (Telegram, email, etc.), sem backend intermediário — arquitetura adequada para uso pessoal, num único dispositivo, com destino fixo.

**Objetivo principal:** garantir que nenhuma mensagem se perde, mesmo em condições adversas de conectividade, gestão agressiva de bateria dos fabricantes, ou falhas temporárias do canal de destino.

### 1.1 Princípios

Estes princípios decidem os conflitos: se uma proposta choca com um deles, o princípio ganha — ou é alterado aqui primeiro, com justificação.

1. **Nenhuma mensagem se perde** — na dúvida entre duplicar e perder, duplica-se.
2. **O conteúdo dos SMS é sensível** — sai do dispositivo só para o canal de destino, via HTTPS; não fica guardado mais do que o necessário; nunca aparece em logs.
3. **Sem backend** — o app fala diretamente com o canal de destino.
4. **Mínimo** — só as permissões, dependências e camadas que uma funcionalidade real exige.
5. **Configurar e esquecer** — a app funciona sem ser aberta; abri-la serve para ver o estado, não para operar.

### 1.2 Mapa da documentação

| Documento | Conteúdo |
|---|---|
| `arquitectura.md` (este) | Visão, princípios, arquitetura, bibliotecas, pontos críticos, registo de decisões |
| [`pipeline.md`](pipeline.md) | Caminho de uma SMS etapa a etapa, contrato de segurança, estados, retenção do conteúdo |
| [`permissoes.md`](permissoes.md) | Permissões runtime e acessos especiais, matriz, regras de pedido e revogação |
| [`roadmap.md`](roadmap.md) | Fases de desenvolvimento e o estado de cada uma |
| [`desenvolvimento.md`](desenvolvimento.md) | Requisitos, comandos, testes no emulador, testes manuais, release assinado, convenções |

Cada facto vive num só documento; os outros apontam para ele. Versões de ferramentas e bibliotecas não são escritas na documentação — a fonte é o código (ver `desenvolvimento.md`).

---

## 2. Arquitetura Geral

```
┌──────────────────────────────────────────────────────────────┐
│                     DISPOSITIVO ANDROID                       │
│                                                               │
│  ┌────────────────────┐  grava   ┌───────────┐                │
│  │ BroadcastReceiver  │─────────▶│ Room DB   │                │
│  │ estático (manifest)│          │ (fila/log)│                │
│  │ SMS_RECEIVED       │          └─────▲─────┘                │
│  └─────────┬──────────┘                │ lê / atualiza estado │
│            │ enfileira (trabalho único)│                      │
│            ▼                           │                      │
│  ┌────────────────────┐          ┌─────┴──────┐               │
│  │ WorkManager        │─────────▶│ Retrofit/  │               │
│  │ (retry + backoff)  │          │ OkHttp     │               │
│  └────────────────────┘          └─────┬──────┘               │
└────────────────────────────────────────┼──────────────────────┘
                                         │ HTTPS
                                         ▼
                              ┌──────────────────────┐
                              │   Telegram Bot API   │
                              │   (ou outro canal)   │
                              └──────────────────────┘
```

Sem servidor próprio no meio: o app fala diretamente com a API do canal escolhido (ex: `api.telegram.org`). Isto elimina uma camada inteira de infraestrutura a manter, à custa de menos flexibilidade de roteamento — um compromisso aceitável quando só tu usas a app e o destino é fixo.

Sem processo permanente: não há `Foreground Service`. O sistema acorda a app quando chega um SMS (receiver declarado no manifest) e o WorkManager trata do envio e dos retries, sobrevivendo a reinícios do dispositivo e a a app ser terminada.

### 2.1 Camadas

| Camada | Responsabilidade |
|---|---|
| **Interceção** | `BroadcastReceiver` estático, declarado no `AndroidManifest.xml`, captura o SMS assim que chega ao sistema — mesmo que o processo da app não esteja a correr |
| **Persistência local** | `Room` guarda cada mensagem e o estado do encaminhamento (pendente, enviado, falhado) |
| **Execução em segundo plano** | `WorkManager` (`CoroutineWorker`, trabalho expedited) garante o envio e o retry, mesmo com a app fechada e após reinício |
| **Transporte** | `Retrofit`/`OkHttp` envia o payload diretamente à API do canal escolhido (ex: Telegram) via HTTPS |
| **Canal de saída** | Telegram Bot API (recomendado pela simplicidade), ou alternativa (SMTP, etc.) |

### 2.2 Interceção: receiver estático

O receiver é declarado no manifest, com `android:permission="android.permission.BROADCAST_SMS"` (só o sistema detém essa permissão, por isso nenhuma outra app consegue injetar SMS falsos). `SMS_RECEIVED` está na lista de exceções às restrições de broadcasts implícitos do Android 8+, por isso continua a ser entregue a receivers do manifest nas versões atuais — mesmo com o processo da app morto.

O receiver só grava a mensagem e enfileira o envio; o funcionamento detalhado (multipartes, trabalho único, contrato de segurança) está em [pipeline.md](pipeline.md).

### 2.3 Token do bot: Android Keystore + DataStore

O token é introduzido pelo utilizador no ecrã de configuração em runtime — nunca em código, `BuildConfig`, `local.properties` ou recursos, pois tudo isso fica em texto plano dentro do APK.

- É gerada uma chave AES-256 no Android Keystore (`KeyGenParameterSpec`, `AES/GCM/NoPadding`); a chave nunca sai do Keystore.
- O token é encriptado com essa chave e o par (IV, ciphertext), em Base64, é guardado no DataStore. O chat ID e o formato da mensagem não são segredos e ficam no DataStore sem encriptação.
- Não se usa `androidx.security:security-crypto`: a biblioteca foi descontinuada pela Google em 2025, e o Keystore faz o necessário sem dependências extra.
- O token faz parte do URL da Bot API (`/bot<TOKEN>/sendMessage`). O `HttpLoggingInterceptor` só pode existir em builds de debug e tem de ocultar o token do URL — caso contrário, aparece no logcat.
- Se o bot Telegram for dedicado só a isto, o impacto de um eventual vazamento do token é limitado (dá só para enviar mensagens ao teu chat, não acesso à tua conta).

### 2.4 Permissões

Só `RECEIVE_SMS`, `INTERNET`, `POST_NOTIFICATIONS` e o pedido de isenção de bateria; `READ_SMS` não é pedida. Matriz completa e regras em [permissoes.md](permissoes.md).

---

## 3. Plano de Desenvolvimento

As fases, os passos e o estado de cada um estão em [roadmap.md](roadmap.md).

---

## 4. Bibliotecas Necessárias

| Biblioteca | Finalidade |
|---|---|
| `androidx.core:core-ktx` | Extensões Kotlin para APIs do Android |
| `androidx.compose.*` (BOM) | UI declarativa (Jetpack Compose) |
| `androidx.room:room-runtime` + `room-ktx` + `room-compiler` (KSP) | Persistência local (fila/log de mensagens) |
| `androidx.work:work-runtime-ktx` | Agendamento e retry de encaminhamento em segundo plano |
| `com.squareup.retrofit2:retrofit` + `converter-kotlinx-serialization` | Cliente HTTP para comunicação com a API do canal (Telegram, etc.) |
| `org.jetbrains.kotlinx:kotlinx-serialization-json` | Serialização JSON |
| `com.squareup.okhttp3:okhttp` + `logging-interceptor` (só debug) | Camada de transporte HTTP, debug de requisições |
| `androidx.datastore:datastore-preferences` | Guardar configurações e o token encriptado |
| `kotlinx.coroutines` | Programação assíncrona (Workers, chamadas de rede) |
| `androidx.navigation:navigation-compose` | Navegação entre ecrãs |

A encriptação do token usa o Android Keystore da plataforma (`java.security` / `javax.crypto`), sem biblioteca adicional.

Não é necessária nenhuma biblioteca de backend — toda a stack é Android/Kotlin.

---

## 5. Pontos de Atenção Críticos

- **Restrições de fabricante**: Xiaomi (MIUI/HyperOS), Huawei (EMUI), Samsung e outros têm camadas próprias de gestão de bateria que podem bloquear o arranque da app por broadcasts ("arranque automático") e atrasar o WorkManager. É necessário ativar o arranque automático e desativar essas otimizações manualmente no teu próprio dispositivo.
- **App forçada a parar**: se a app for parada em *Definições → Forçar paragem*, o Android deixa de lhe entregar broadcasts até ser aberta de novo. Os SMS recebidos nesse intervalo não são encaminhados.
- **Antes do primeiro desbloqueio**: após um reinício, a app só corre depois do primeiro desbloqueio do dispositivo (não é *direct boot aware*). SMS que cheguem antes disso ficam na caixa de entrada mas não são encaminhados. Limitação aceite: sem `READ_SMS` não há recuperação a partir da caixa de entrada.
- **Sem necessidade de Play Store**: como é uso pessoal, a distribuição é por sideload direto do APK — evita completamente as restrições da Play Store sobre permissões de SMS.
- **Privacidade e segurança**: o conteúdo dos SMS é dado sensível — a comunicação com a API do canal deve ser sempre via HTTPS, o token do bot deve ficar encriptado em disco (secção 2.3), e nem o token nem o conteúdo das SMS aparecem em logs.
- **Retenção do conteúdo**: o corpo de cada SMS é apagado do Room 1 minuto após o 200 OK do Telegram; ficam só os metadados para o histórico. Mecanismo em [pipeline.md](pipeline.md) §4.
- **Backup desativado**: `android:allowBackup="false"` e todas as regras de `backup_rules.xml` (API 29–30) / `data_extraction_rules.xml` (API 31+) excluem tudo, incluindo a transferência dispositivo-a-dispositivo (que no Android 12+ não é desligada só pelo `allowBackup`). Motivos: o conteúdo dos SMS no Room não deve sair do dispositivo, e o token encriptado não é recuperável após um restauro (a chave do Android Keystore não é incluída no backup). Consequência: ao mudar de telemóvel, a configuração (token, chat ID) tem de ser feita de novo.
- **Confiabilidade de entrega**: a combinação `Room` (estado local) + `WorkManager` (retry com backoff) é suficiente, sem backend, para garantir que uma mensagem não se perde em falhas temporárias de rede — desde que o canal de destino (Telegram, por exemplo) tenha boa disponibilidade. As regras que garantem isto estão no contrato de segurança de [pipeline.md](pipeline.md) §2.
- **Caminho de evolução**: se no futuro precisares de múltiplos canais, filtros de roteamento, ou vários dispositivos a reportar ao mesmo sítio, o backend intermediário (FastAPI/Spring Boot) volta a fazer sentido — a arquitetura atual não impede essa evolução, só a adia até ser realmente necessária.

---

## 6. Registo de Decisões

| Data | Decisão | Motivo |
|---|---|---|
| 2026-11-26 | Backup e transferência entre dispositivos desativados | Conteúdo dos SMS não sai do dispositivo; token encriptado não é restaurável |
| 2026-11-26 | Receiver estático no manifest em vez de registo dinâmico | Um receiver dinâmico só existe com o processo vivo — SMS recebidos com a app terminada perdiam-se, contrariando o objetivo principal |
| 2026-11-26 | Sem `Foreground Service` | Desnecessário com receiver estático + WorkManager; evita a notificação permanente, a escolha de `foregroundServiceType` (Android 14+) e o limite de ~6 h do tipo `dataSync` (Android 15+) |
| 2026-11-26 | Token encriptado com Android Keystore + DataStore; introduzido em runtime | `BuildConfig` deixa o token em texto plano no APK; `security-crypto` foi descontinuado |
| 2026-11-26 | Só `RECEIVE_SMS`, sem `READ_SMS` | Permissão mínima; recuperação a partir da caixa de entrada não compensa a complexidade com receiver estático |
| 2026-11-26 | Nome corrigido para `SMSForwarder` / `com.cxdy.smsforwarder` | Erro de escrita corrigido antes da primeira instalação real |
| 2026-11-26 | Corpo da SMS apagado 1 minuto após o 200 OK do Telegram; metadados mantidos | O conteúdo não deve ficar no dispositivo depois de entregue; os metadados bastam para o histórico |
| 2026-11-26 | Documentação dividida em arquitetura, pipeline, permissões, roadmap e desenvolvimento; cada facto num só documento | O documento único misturava decisões, plano e detalhe de implementação; duplicar factos leva a documentos que se contradizem |
| 2026-11-26 | Docs em português; código, commits, PRs e templates do GitHub em inglês | Formaliza a prática já seguida no repositório |
