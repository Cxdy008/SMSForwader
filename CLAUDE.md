# CLAUDE.md

Este ficheiro orienta o Claude Code ao trabalhar neste repositório.

## Visão geral do projeto

**SMSForwarder** é uma aplicação Android nativa (Kotlin) que intercepta SMS recebidas no dispositivo e encaminha-as diretamente para um canal externo (Telegram Bot API, por defeito), sem backend intermediário. Uso pessoal, num único dispositivo, com destino fixo.

A documentação vive em `docs/`, um facto por documento:

- **`docs/arquitectura.md`** — princípios, arquitetura, bibliotecas e **registo de decisões** (§6). Consultar antes de propor mudanças estruturais (nova camada, novo canal de saída, reintrodução de backend, etc.) e acrescentar uma linha ao registo sempre que uma decisão mudar.
- **`docs/pipeline.md`** — caminho de uma SMS, **contrato de segurança** (§2), estados e retenção do conteúdo. Qualquer código no receiver, worker, repositório ou cliente de rede tem de respeitar o contrato.
- **`docs/permissoes.md`** — permissões e acessos especiais.
- **`docs/roadmap.md`** — fases e estado de cada uma; atualizar o estado no mesmo PR que o altera.
- **`docs/desenvolvimento.md`** — comandos, testes no emulador, testes manuais, release assinado e convenções de branch/commit/PR.

Não duplicar factos entre documentos (apontar para o documento dono) e não escrever versões na documentação — a fonte são os ficheiros de build.

## Stack técnica

- **Linguagem**: Kotlin
- **UI**: Jetpack Compose
- **Persistência local**: Room (fila/log de mensagens)
- **Background/retry**: WorkManager (`CoroutineWorker`, backoff exponencial)
- **Rede**: Retrofit + OkHttp (chamada direta à API do canal, ex. `api.telegram.org`)
- **Configurações**: DataStore (preferences); o token do bot é encriptado com uma chave AES-GCM do Android Keystore antes de ir para o DataStore (não usar `security-crypto`, descontinuado)
- **Build**: Gradle Kotlin DSL (`build.gradle.kts`), version catalog em `libs.versions.toml`; AGP 9 com Kotlin integrado (não aplicar o plugin `org.jetbrains.kotlin.android`); o AGP não pode passar a linha suportada pelo Android Studio instalado (ver `docs/desenvolvimento.md`, problemas frequentes)
- **SDK**: níveis em `app/build.gradle.kts` — subir o `targetSdk` é uma mudança de comportamento em runtime e deve ser feita e testada à parte do `compileSdk`

## Identificadores do projeto

- `applicationId` / package raiz: `com.cxdy.smsforwarder`
- Manter o package Kotlin e o `applicationId` sincronizados — um rename de package via refactor do Android Studio não atualiza o `applicationId` automaticamente, é preciso editar `build.gradle.kts (Module :app)` à parte.

## Estrutura de pacotes

```
com.cxdy.smsforwarder/
├── receiver/   # BroadcastReceiver estático para SMS_RECEIVED
├── data/       # Entidades Room, DAO, repositório, DataStore
├── network/    # Retrofit, clientes de API do canal (Telegram, etc.)
├── security/   # Encriptação do token com Android Keystore
├── worker/     # CoroutineWorker / WorkManager
└── ui/         # Ecrãs Compose
```

## Comandos comuns

```bash
./gradlew assembleDebug          # build de debug
./gradlew assembleRelease        # build de release (sideload, sem Play Store)
./gradlew test                   # testes unitários
./gradlew connectedAndroidTest   # testes instrumentados (requer dispositivo/emulador)
adb install app/build/outputs/apk/release/app-release.apk
```

## Convenções e decisões a respeitar

- **Sem backend**: o app fala diretamente com a API do canal de saída. Não introduzir um servidor intermediário sem atualizar `/docs/arquitectura.md` primeiro e justificar a mudança (ex: necessidade real de múltiplos canais, múltiplos dispositivos, ou roteamento condicional — ver secção "Caminho de evolução" nesse documento).
- **Token do bot nunca em texto plano**: introduzido em runtime no ecrã de configuração e encriptado com Android Keystore + DataStore (`docs/arquitectura.md` §2.3). Nunca em código, `BuildConfig`, `local.properties` ou recursos; nunca commitado; nunca em logs (o token faz parte do URL da Bot API — o `HttpLoggingInterceptor` só em debug e com o token ocultado).
- **Registo do `BroadcastReceiver`**: estático no manifest, protegido por `android:permission="android.permission.BROADCAST_SMS"` — não usar `registerReceiver` em runtime, que perde SMS quando o processo está morto. O receiver só grava no Room e enfileira trabalho (via `goAsync()`); nunca faz rede.
- **Sem Foreground Service**: o envio e os retries são do WorkManager (trabalho único por mensagem). Não reintroduzir um serviço permanente sem atualizar `docs/arquitectura.md`.
- **Permissões mínimas**: `RECEIVE_SMS`, não `READ_SMS` (ver `docs/permissoes.md`).
- **Conteúdo das SMS**: nunca em logs; o corpo é apagado do Room 1 minuto após o 200 OK do Telegram (`docs/pipeline.md` §4).
- **Lógica testável sem Android**: juntar multipartes, formatar e dividir mensagens vive em classes Kotlin puras (entrada `RawSms`), com testes JVM.
- **Dependências**: cada biblioteca nova é justificada no PR; sem frameworks de DI (Hilt/Koin), usar injeção por construtor.
- **Língua**: `docs/` e este ficheiro em português; código, commits, PRs e templates do GitHub em inglês.
- **Distribuição**: sideload direto do APK assinado — não há submissão à Play Store (política restritiva para permissões de SMS).
- **Gestão de bateria**: o app depende de isenção de otimização de bateria (`REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`); em dispositivos Xiaomi/Huawei/Samsung pode ser necessário ativar o arranque automático e outra configuração manual fora do app.

## Ao propor mudanças

Antes de alterar fluxo de dados, adicionar um novo canal de encaminhamento, ou mexer na camada de retry/fila, ler `/docs/arquitectura.md` para confirmar que a mudança é consistente com as decisões já tomadas — e atualizar esse documento como parte da mesma alteração, não como passo separado.