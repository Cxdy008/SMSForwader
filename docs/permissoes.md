# Permissões — SMSForwarder

> **Estado:** as permissões estão declaradas no `AndroidManifest.xml`; o pedido em runtime e os ecrãs de acesso especial ainda não existem (ver [roadmap](roadmap.md), Fase 6).

## 1. Dois mecanismos diferentes

A app depende de dois tipos de acesso que não se pedem da mesma forma. Não os confundir no código nem na UI.

### Permissões runtime

Declaradas no manifest e pedidas com `ActivityResultContracts.RequestPermission`. O utilizador responde num diálogo do sistema.

- `RECEIVE_SMS` — obrigatória; sem ela não chega nenhum broadcast.
- `POST_NOTIFICATIONS` (Android 13+) — necessária para a notificação que o WorkManager mostra ao correr trabalho expedited como foreground em API < 31, e para alertas de falha persistente.

### Acessos especiais nas Definições

Não têm diálogo de permissão. A app só pode abrir o ecrã certo das Definições e o utilizador concede lá.

- **Isenção de otimização de bateria** — `Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` (exige `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` no manifest); estado lido com `PowerManager.isIgnoringBatteryOptimizations()`.
- **Arranque automático / gestão de bateria do fabricante** (Xiaomi, Huawei, Samsung, etc.) — sem API pública. Não é possível detetar o estado; a UI só pode explicar e, quando existir, abrir o ecrã do fabricante.

## 2. Matriz

| Capacidade | Mecanismo | Obrigatório | Estado |
|---|---|---|---|
| Receber SMS | `RECEIVE_SMS` (runtime) | Sim | Declarada; pedido planeado |
| Enviar para o Telegram | `INTERNET` (normal, concedida na instalação) | Sim | Declarada |
| Notificação do worker / alertas | `POST_NOTIFICATIONS` (runtime, API 33+) | Recomendado | Declarada; pedido planeado |
| Não ser adiado pelo Doze | Isenção de bateria (Definições) | Recomendado | `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` declarada; ecrã planeado |
| Arrancar após ser terminada (OEM) | Arranque automático (Definições do fabricante) | Depende do fabricante | Orientação na UI planeada |
| Ler a caixa de entrada | `READ_SMS` | — | **Nunca pedida** (ver decisão em [arquitectura.md](arquitectura.md) §6) |
| Enviar SMS, contactos, localização, armazenamento, câmara, microfone | — | — | **Nunca necessárias** |

O manifest declara também `android.hardware.telephony` com `required="false"`: `RECEIVE_SMS` implica telefonia, mas a app deve instalar-se em dispositivos de teste sem SIM.

### Permissões acrescentadas por bibliotecas

O manifest final (fundido) inclui permissões que não estão no nosso `AndroidManifest.xml`. Todas são normais (concedidas na instalação, sem diálogo). Verificar com `aapt2 dump permissions <apk>` sempre que se adicionar uma biblioteca:

| Permissão | Origem | Para quê |
|---|---|---|
| `WAKE_LOCK` | WorkManager | Manter o CPU acordado enquanto um worker corre |
| `ACCESS_NETWORK_STATE` | WorkManager | Restrição "só com rede" |
| `RECEIVE_BOOT_COMPLETED` | WorkManager | Reagendar trabalho pendente após reinício |
| `FOREGROUND_SERVICE` | WorkManager | Trabalho expedited corre como foreground em API < 31. Em API ≥ 31 usa jobs expedited, por isso não é preciso declarar `foregroundServiceType` |
| `<applicationId>.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION` | androidx.core | Permissão interna, de assinatura, para receivers dinâmicos não exportados |

## 3. Regras

- **Detetar revogação**: o estado de cada permissão e acesso é lido de novo sempre que o ecrã volta a estar visível (`ON_RESUME`), nunca guardado em cache. O utilizador pode revogar tudo nas Definições a qualquer momento.
- **Uma de cada vez**: no onboarding, cada permissão é pedida no seu passo, com explicação do motivo antes do diálogo do sistema.
- **Sem `RECEIVE_SMS`, a app diz que não está a funcionar** — o ecrã principal mostra isso como bloqueio, não como aviso.
- **Recusar é tão fácil como aceitar**: sem insistência nem texto de culpa; o pedido pode ser repetido a partir do ecrã principal.
- **Permissões são sensíveis**: qualquer forma de obter conteúdo de SMS sem `RECEIVE_SMS` concedida, ou de o expor a outra app, é um bug de segurança.
