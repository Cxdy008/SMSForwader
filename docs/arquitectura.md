# SMSForwarder — Documento de Arquitetura e Plano de Desenvolvimento

## 1. Visão Geral

O SMSForwarder é uma aplicação Android nativa (Kotlin) responsável por interceptar mensagens SMS recebidas num dispositivo e encaminhá-las diretamente para um canal externo (Telegram, email, etc.), sem backend intermediário — arquitetura adequada para uso pessoal, num único dispositivo, com destino fixo.

**Objetivo principal:** garantir que nenhuma mensagem se perde, mesmo em condições adversas de conectividade, gestão agressiva de bateria dos fabricantes, ou falhas temporárias do canal de destino.

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

O receiver é declarado no manifest, com `android:exported="true"` e `android:permission="android.permission.BROADCAST_SMS"` (só o sistema detém essa permissão, por isso nenhuma outra app consegue injetar SMS falsos).

- `SMS_RECEIVED` está na lista de exceções às restrições de broadcasts implícitos do Android 8+, por isso continua a ser entregue a receivers do manifest nas versões atuais.
- O receiver faz o mínimo: usa `goAsync()` para, dentro da janela de ~10 s do broadcast, gravar a mensagem no Room e enfileirar o envio. Nunca faz chamadas de rede.
- Uma SMS longa chega em vários PDUs: usar `Telephony.Sms.Intents.getMessagesFromIntent()` e concatenar os corpos do mesmo remetente numa só mensagem.
- O envio é enfileirado como trabalho único por mensagem (`enqueueUniqueWork` com o id do Room, política `KEEP`), para nunca enviar a mesma SMS duas vezes por causa da fila.

### 2.3 Token do bot: Android Keystore + DataStore

O token é introduzido pelo utilizador no ecrã de configuração em runtime — nunca em código, `BuildConfig`, `local.properties` ou recursos, pois tudo isso fica em texto plano dentro do APK.

- É gerada uma chave AES-256 no Android Keystore (`KeyGenParameterSpec`, `AES/GCM/NoPadding`); a chave nunca sai do Keystore.
- O token é encriptado com essa chave e o par (IV, ciphertext), em Base64, é guardado no DataStore. O chat ID e o formato da mensagem não são segredos e ficam no DataStore sem encriptação.
- Não se usa `androidx.security:security-crypto`: a biblioteca foi descontinuada pela Google em 2025, e o Keystore faz o necessário sem dependências extra.
- O token faz parte do URL da Bot API (`/bot<TOKEN>/sendMessage`). O `HttpLoggingInterceptor` só pode existir em builds de debug e tem de ocultar o token do URL — caso contrário, aparece no logcat.
- Se o bot Telegram for dedicado só a isto, o impacto de um eventual vazamento do token é limitado (dá só para enviar mensagens ao teu chat, não acesso à tua conta).

### 2.4 Permissões

| Permissão | Motivo |
|---|---|
| `RECEIVE_SMS` | Receber o broadcast `SMS_RECEIVED` (runtime, pedida no onboarding) |
| `INTERNET` | Chamada à API do canal |
| `POST_NOTIFICATIONS` | Android 13+: notificação exigida pelo WorkManager quando trabalho expedited corre como foreground em API < 31, e alertas de falha persistente |
| `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` | Pedir isenção de otimização de bateria |

`READ_SMS` não é pedida: `RECEIVE_SMS` basta para encaminhar SMS recebidas, e com o receiver estático não há um mecanismo de recuperação a partir da caixa de entrada (ver limitações na secção 5).

---

## 3. Plano de Desenvolvimento — Passo a Passo

### Fase 1 — Fundação do projeto Android
1. Criar projeto Kotlin + Jetpack Compose no Android Studio
2. Configurar `build.gradle.kts` com as dependências base
3. Definir estrutura de pacotes (`receiver`, `data`, `network`, `security`, `worker`, `ui`)
4. Configurar `AndroidManifest.xml` com as permissões da secção 2.4

### Fase 2 — Interceção de SMS
5. Implementar `BroadcastReceiver` para `SMS_RECEIVED_ACTION`, declarado no manifest com `BROADCAST_SMS` (secção 2.2)
6. Extrair remetente, corpo e timestamp com `getMessagesFromIntent()`, concatenando SMS multipartes
7. Usar `goAsync()` para gravar no Room e enfileirar o envio sem bloquear o broadcast
8. Testar receção em emulador via `adb emu sms send`, incluindo com a app terminada (`adb shell am kill`)

### Fase 3 — Persistência local
9. Modelar entidade `SmsMessage` (id, remetente, corpo, timestamp, estado, tentativas)
10. Criar `Room` database com DAO para inserir/atualizar/consultar mensagens
11. Implementar repositório (`SmsRepository`) como camada de abstração entre UI/Worker e a base de dados

### Fase 4 — Resistência à gestão de bateria
12. Pedir isenção de otimização de bateria (`REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`)
13. Adicionar orientação na UI para ativar o arranque automático e desativar a gestão agressiva de bateria, caso o dispositivo seja de um fabricante como Xiaomi/Huawei/Samsung

### Fase 5 — Encaminhamento e fila
14. Implementar `CoroutineWorker` (expedited, com `getForegroundInfo()` para API < 31) que envia uma mensagem pendente
15. Configurar trabalho único por mensagem, restrição de rede e retry com backoff exponencial
16. Implementar chamada HTTP via `Retrofit` diretamente para a API do canal (ex: `https://api.telegram.org/bot<TOKEN>/sendMessage`), com serialização `kotlinx.serialization`
17. Dividir mensagens acima de 4096 caracteres (limite do Telegram)
18. Atualizar estado da mensagem no `Room` conforme resposta (sucesso/falha)

### Fase 6 — UI da aplicação
19. Ecrã principal: estado (permissões, isenção de bateria, configuração válida), histórico de mensagens encaminhadas
20. Ecrã de configuração: token do bot (encriptado, secção 2.3), chat ID de destino, formato da mensagem
21. Onboarding: pedido de permissões passo a passo, explicação da necessidade de isenção de bateria

### Fase 7 — Testes e robustez
22. Testes unitários do repositório e da lógica de formatação/envio
23. Testes de integração: simular perda de rede, reinício do dispositivo, app morta pelo sistema
24. Teste em condições reais no teu próprio dispositivo por alguns dias antes de confiar na app para uso contínuo

### Fase 8 — Distribuição
25. Build do APK de release (sideload — não há necessidade de Play Store para uso pessoal)
26. Assinar o APK com keystore próprio
27. Instalar diretamente no dispositivo (`adb install` ou transferência manual)

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
- **Privacidade e segurança**: o conteúdo dos SMS é dado sensível — a comunicação com a API do canal deve ser sempre via HTTPS, e o token do bot deve ficar encriptado em disco (secção 2.3) e nunca aparecer em logs.
- **Backup desativado**: `android:allowBackup="false"` e todas as regras de `backup_rules.xml` (API 29–30) / `data_extraction_rules.xml` (API 31+) excluem tudo, incluindo a transferência dispositivo-a-dispositivo (que no Android 12+ não é desligada só pelo `allowBackup`). Motivos: o conteúdo dos SMS no Room não deve sair do dispositivo, e o token encriptado não é recuperável após um restauro (a chave do Android Keystore não é incluída no backup). Consequência: ao mudar de telemóvel, a configuração (token, chat ID) tem de ser feita de novo.
- **Confiabilidade de entrega**: a combinação `Room` (estado local) + `WorkManager` (retry com backoff) é suficiente, sem backend, para garantir que uma mensagem não se perde em falhas temporárias de rede — desde que o canal de destino (Telegram, por exemplo) tenha boa disponibilidade. Se o envio tiver sucesso mas a resposta se perder, o retry pode duplicar a mensagem no Telegram; é preferível a perdê-la.
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
