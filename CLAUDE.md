# CLAUDE.md

Este ficheiro orienta o Claude Code ao trabalhar neste repositório.

## Visão geral do projeto

**SMSForwarder** é uma aplicação Android nativa (Kotlin) que intercepta SMS recebidas no dispositivo e encaminha-as diretamente para um canal externo (Telegram Bot API, por defeito), sem backend intermediário. Uso pessoal, num único dispositivo, com destino fixo.

A arquitetura completa, o plano de desenvolvimento por fases e a justificação das decisões de design (incluindo por que não há backend) estão documentados em **`/docs/arquitectura.md`** — consultar esse ficheiro antes de propor mudanças estruturais (nova camada, novo canal de saída, reintrodução de backend, etc.), e mantê-lo atualizado sempre que uma decisão de arquitetura mudar.

## Stack técnica

- **Linguagem**: Kotlin
- **UI**: Jetpack Compose
- **Persistência local**: Room (fila/log de mensagens)
- **Background/retry**: WorkManager (`CoroutineWorker`, backoff exponencial)
- **Rede**: Retrofit + OkHttp (chamada direta à API do canal, ex. `api.telegram.org`)
- **Configurações**: DataStore (preferences); o token do bot é encriptado com uma chave AES-GCM do Android Keystore antes de ir para o DataStore (não usar `security-crypto`, descontinuado)
- **Build**: Gradle Kotlin DSL (`build.gradle.kts`), version catalog em `libs.versions.toml`; AGP 9 com Kotlin integrado (não aplicar o plugin `org.jetbrains.kotlin.android`)
- **SDK**: `compileSdk` 37.2, `targetSdk` 36, `minSdk` 29 — subir o `targetSdk` é uma mudança de comportamento em runtime e deve ser feita e testada à parte

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
- **Permissões mínimas**: `RECEIVE_SMS`, não `READ_SMS` (ver `docs/arquitectura.md` §2.4).
- **Distribuição**: sideload direto do APK assinado — não há submissão à Play Store (política restritiva para permissões de SMS).
- **Gestão de bateria**: o app depende de isenção de otimização de bateria (`REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`); em dispositivos Xiaomi/Huawei/Samsung pode ser necessário ativar o arranque automático e outra configuração manual fora do app.

## Ao propor mudanças

Antes de alterar fluxo de dados, adicionar um novo canal de encaminhamento, ou mexer na camada de retry/fila, ler `/docs/arquitectura.md` para confirmar que a mudança é consistente com as decisões já tomadas — e atualizar esse documento como parte da mesma alteração, não como passo separado.