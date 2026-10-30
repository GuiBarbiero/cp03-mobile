# IoT Control: painel MQTT em React Native

Aplicativo mobile (React Native + Expo + TypeScript) que monitora e controla um ambiente inteligente simulado usando **MQTT sobre WebSocket**. O app recebe leituras de temperatura e umidade em tempo real e liga/desliga uma lâmpada virtual; um script simulador faz o papel do dispositivo IoT, sem necessidade de hardware.

**Objetivo:** demonstrar o modelo publish/subscribe, o uso de tópicos MQTT e o gerenciamento de conexão e mensagens em uma aplicação mobile.

## Integrantes

| Nome completo | RM |
| --- | --- |
| _PREENCHER_ | _PREENCHER_ |
| _PREENCHER_ | _PREENCHER_ |
| _PREENCHER_ | _PREENCHER_ |

## Tecnologias e versões

| Tecnologia | Versão |
| --- | --- |
| Expo SDK | 58.0.21 |
| React Native | 0.88.1 |
| React | 19.3.0 |
| TypeScript | 6.0.3 |
| Expo Router (navegação por abas) | 58.0.28 |
| MQTT.js (`mqtt`), cliente MQTT do app e do simulador | 5.16.1 |
| @react-native-async-storage/async-storage | 2.2.0 |
| @expo/vector-icons | 15.1.1 |
| react-native-web (execução no navegador) | 0.21.4 |
| Node.js usado no desenvolvimento | 24.15 (recomendado: 20 LTS ou superior) |

Plataformas: Android e iOS (via Expo Go) e Web. O MQTT.js tem build próprio para React Native e para navegador, então o mesmo código funciona nas três.

## Instalação e execução

Pré-requisitos: Node.js 20+ e o app **Expo Go** (compatível com o SDK 58) no celular, ou um emulador Android / simulador iOS.

```bash
git clone <url-deste-repositorio>
cd <pasta-do-repositorio>
npm install
```

Em um terminal, inicie o simulador do dispositivo:

```bash
npm run simulator
```

Em outro terminal, inicie o app:

```bash
npx expo start
```

Depois, escaneie o QR Code com o Expo Go (Android) ou com a câmera (iOS). Também é possível pressionar `a` (emulador Android), `i` (simulador iOS) ou `w` (navegador).

Para checar os tipos: `npm run typecheck`.

## Broker MQTT

| Item | Valor |
| --- | --- |
| Broker público padrão | EMQX (`broker.emqx.io`) |
| **URL usada pelo app (WebSocket seguro)** | `wss://broker.emqx.io:8084/mqtt` |
| Porta WebSocket sem TLS (alternativa) | `ws://broker.emqx.io:8083/mqtt` |
| Porta MQTT TCP (não funciona no app) | `mqtt://broker.emqx.io:1883` |
| Broker alternativo | HiveMQ: `wss://broker.hivemq.com:8884/mqtt` |

O app só aceita `ws://` ou `wss://`: o React Native não abre sockets TCP puros, então a porta 1883 não serve para ele. Se o usuário digitar `mqtt://...`, o app recusa e explica o motivo. Prefira `wss://`, porque iOS e Android bloqueiam tráfego sem TLS por padrão em builds de produção.

A URL pode ser trocada na tela **Configurações**, que tem atalhos para os brokers acima. O broker escolhido fica salvo no aparelho. App e simulador precisam usar o **mesmo broker** (o simulador lê o padrão de `mqtt.config.json` ou a variável `BROKER_URL`).

Nenhuma credencial é necessária: os brokers são públicos e não há senhas ou tokens no repositório.

## Tópicos e payloads

O prefixo dos tópicos fica em [`mqtt.config.json`](mqtt.config.json) e é lido **tanto pelo app quanto pelo simulador**. Para evitar interferência com outros grupos no broker público, troque `topicBase` por um identificador exclusivo do grupo.

Prefixo atual: `fiap/cp03/iot-control`

| Tópico | Direção no app | Payload | Quem publica |
| --- | --- | --- | --- |
| `fiap/cp03/iot-control/sensor/temperatura` | Subscribe | `{"value":24.5}` (°C, aceito de -40 a 85) | Simulador, a cada 5 s |
| `fiap/cp03/iot-control/sensor/umidade` | Subscribe | `{"value":58}` (%, aceito de 0 a 100) | Simulador, a cada 5 s |
| `fiap/cp03/iot-control/lampada/comando` | Publish | `{"action":"ON"}` ou `{"action":"OFF"}` | App (botões Ligar/Desligar) |
| `fiap/cp03/iot-control/lampada/estado` | Subscribe | `{"state":"ON"}` ou `{"state":"OFF"}` | Simulador, após cada comando (com `retain`) |

QoS 1 em todas as inscrições e publicações. O estado da lâmpada é publicado com `retain`, então o app já recebe o estado atual assim que se inscreve.

Mensagens fora desse formato (JSON inválido, `value` não numérico ou fora da faixa, `state` diferente de ON/OFF) **não alteram a tela**: elas aparecem no histórico marcadas como **Inválida**.

## Simulador do dispositivo

O script [`simulator/device.js`](simulator/device.js) é um segundo cliente MQTT (Node.js) que faz o papel do dispositivo:

- publica temperatura e umidade a cada 5 segundos (variação aleatória suave);
- se inscreve em `lampada/comando` e, ao receber `ON`/`OFF`, publica o novo estado em `lampada/estado`;
- ignora comandos inválidos.

```bash
npm run simulator
```

Com o simulador rodando, digite no terminal para forçar eventos durante a demonstração:

| Comando | Efeito |
| --- | --- |
| `t 30` | publica temperatura 30 |
| `u 75` | publica umidade 75 |
| `on` / `off` | muda a lâmpada direto pelo dispositivo |
| `invalido` | publica um payload inválido (`{"value":"quente"}`) para testar a validação |
| `sair` | desconecta e encerra |

Variáveis opcionais:

- `BROKER_URL`: broker usado pelo simulador (padrão: o mesmo do app, `wss://broker.emqx.io:8084/mqtt`). O simulador roda em Node, então também aceita TCP: `BROKER_URL=mqtt://broker.emqx.io:1883 npm run simulator` (o broker entrega as mensagens ao app, que continua via WebSocket).
- `INTERVAL_MS`: intervalo entre leituras (padrão 5000).

No PowerShell (Windows), defina a variável antes: `$env:BROKER_URL="mqtt://broker.emqx.io:1883"; npm run simulator`.

### Alternativa sem o script

Qualquer cliente MQTT serve, por exemplo o [MQTTX](https://mqttx.app/) (desktop ou [web](https://mqttx.app/web-client)):

1. Conecte em `broker.emqx.io` (TCP `1883`, ou WebSocket `wss://broker.emqx.io:8084/mqtt`).
2. Inscreva-se em `fiap/cp03/iot-control/lampada/comando`.
3. Publique `{"value":24.5}` em `.../sensor/temperatura` e `{"value":58}` em `.../sensor/umidade`.
4. Ao ver `{"action":"ON"}` chegar no tópico de comando, publique `{"state":"ON"}` em `.../lampada/estado` (e o equivalente para `OFF`).

## Roteiro de verificação

1. Rode `npm run simulator` e abra o app: o status muda para **Conectado** e temperatura e umidade passam a atualizar sozinhas, com o horário de cada leitura.
2. Digite `t 30` no simulador: a temperatura vai para 30.0 °C. Digite `u 75`: a umidade vai para 75.0 %.
3. Toque em **Ligar**: o app mostra "aguardando confirmação", o simulador registra o comando e publica o estado, e o app passa a exibir **Ligada**.
4. Toque em **Desligar**: mesmo fluxo, terminando em **Desligada**.
5. Em **Configurações**, toque em **Desconectar** e depois em **Ligar** no Dashboard: aparece "Sem conexão com o broker: o comando não foi enviado."
6. O **Histórico** mostra as 10 mensagens mais recentes, enviadas e recebidas.

Se o simulador estiver parado, o app avisa após 5 s que o dispositivo não confirmou o comando, e o estado exibido não muda.

## Telas e funcionalidades

| Dashboard (claro) | Dashboard (escuro) | Histórico | Configurações |
| --- | --- | --- | --- |
| ![Dashboard](docs/screenshots/dashboard.jpg) | ![Dashboard escuro](docs/screenshots/dashboard-escuro.jpg) | ![Histórico](docs/screenshots/historico.jpg) | ![Configurações](docs/screenshots/configuracoes.jpg) |

**Dashboard**
- Status da conexão (conectando, conectado, desconectado ou erro) com a URL do broker, a mensagem de erro e um botão **Reconectar** quando a conexão cai.
- Cartões de temperatura e umidade com o último valor e o horário de atualização. Atualizam sozinhos, sem botão de recarregar.
- Lâmpada: o estado exibido vem **somente** da mensagem recebida em `lampada/estado`, nunca do toque no botão. Depois de um comando, o app mostra "aguardando confirmação" até o dispositivo responder (ou um aviso após 5 s sem resposta).
- Sem conexão, os botões não publicam nada e mostram um aviso.

**Histórico**
- As 10 mensagens mais recentes, enviadas e recebidas, com direção, tópico, payload e horário. Payloads inválidos ficam marcados.
- O histórico fica em memória enquanto o app está aberto.

**Configurações**
- Campo da URL do broker com validação (rejeita `mqtt://` e URLs malformadas) e atalhos para brokers públicos.
- Botões Conectar/Reconectar e Desconectar.
- Alternância entre tema claro e escuro, aplicada a todas as telas.
- Lista dos tópicos em uso.
- A URL do broker e o tema são salvos com AsyncStorage e restaurados ao reabrir o app. Ao abrir, o app conecta automaticamente ao broker salvo.

**Conexão e reconexão:** se a conexão cair ou falhar, o cliente tenta de novo a cada 5 s e a tela mostra o erro até reconectar. O usuário também pode reconectar manualmente ou desconectar para interromper as tentativas.

## Estrutura do projeto

```
mqtt.config.json          prefixo dos tópicos e broker padrão (app + simulador)
simulator/device.js       simulador do dispositivo IoT
src/
  app/                    rotas do Expo Router (abas)
    _layout.tsx           providers, tema da navegação e abas
    index.tsx             -> DashboardScreen
    history.tsx           -> HistoryScreen
    settings.tsx          -> SettingsScreen
  components/
    Button.tsx
    ConnectionStatus.tsx
    MessageItem.tsx
    SensorCard.tsx
  context/
    MqttContext.tsx       estado MQTT compartilhado (sensores, lâmpada, histórico)
    PreferencesContext.tsx  tema e URL do broker (persistidos)
  screens/
    DashboardScreen.tsx
    HistoryScreen.tsx
    SettingsScreen.tsx
  services/
    mqttService.ts        conexão, inscrição, publicação e listeners (sem React)
    messageParser.ts      validação dos payloads recebidos
    storageService.ts     AsyncStorage
  themes/colors.ts        paletas claro/escuro
  types/mqtt.ts           tipos de estados, mensagens e payloads
  utils/format.ts
  config.ts               tópicos, limites dos sensores e brokers de exemplo
```

**Limpeza de listeners:** o `mqttService` mantém um único cliente. Antes de cada nova conexão ele remove os handlers do cliente anterior, cancela as inscrições e encerra a conexão, então nunca há eventos duplicados. Os listeners da interface são registrados em um `useEffect` do `MqttProvider` e removidos no cleanup do efeito. Os temporizadores do Dashboard também são limpos no cleanup.

## Limitações e observações

- **Brokers públicos** são compartilhados, sem autenticação e sem garantia de disponibilidade. Qualquer pessoa pode publicar nos mesmos tópicos, por isso o prefixo deve ser exclusivo do grupo e as mensagens recebidas são validadas. Se o broker padrão estiver fora do ar, use outro broker da tela Configurações e rode o simulador com o mesmo endereço em `BROKER_URL`.
- O app usa apenas WebSocket (`ws://`/`wss://`). Portas TCP como 1883 não funcionam no React Native.
- O estado exibido com `retain` pode ser de uma execução anterior do simulador. O horário mostrado é o do recebimento pelo app.
- O histórico fica só em memória: vale enquanto o app está aberto e começa vazio ao reabrir.
- Redes corporativas ou de faculdade podem bloquear as portas 8884/8000. Nesse caso, teste em outra rede ou pelos dados móveis.
- Testado no navegador (Expo Web), com o bundle Android compilado via `expo export`. A execução em aparelhos físicos deve ser conferida com o Expo Go.
