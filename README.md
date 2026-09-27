# Telemetria física das rodas

Cada execução do Coach em modo serial cria nesta pasta um arquivo
`telemetria_YYYYMMDD_HHMMSS_mmm.txt`. Ele usa formato CSV e é atualizado com
`flush` a cada ACK válido recebido pelo rádio.

O robô não precisa de USB durante a operação:

```text
encoders -> RP2040 do robô -> ACK nRF24 -> estação-base -> USB -> Coach -> TXT
```

A USB do robô só é necessária para instalar ou atualizar o firmware. Durante a
partida, alimente o robô pela bateria e mantenha somente a estação-base ligada
ao notebook.

## Instalação da telemetria nas placas

Com as duas placas no USB e os nomes de porta conferidos:

```bash
export BASE=/dev/serial/by-id/usb-MicroPython_Board_in_FS_mode_652701c5c26bf16f-if00
export ROBO=/dev/serial/by-id/usb-MicroPython_Board_in_FS_mode_e66178758b576e36-if00

ROBOAP_SERIAL_PORT="$BASE" ./tools/update_base_telemetry.sh
ROBOAP_ROBOT_PORT="$ROBO" ./tools/update_robot_telemetry.sh
```

Os atualizadores são transacionais: arquivos anteriores recebem um sufixo
`.pre_telemetry_DATA_HORA`. No robô, `robot_identity.py` e `main.py` são
preservados. Depois da atualização, desligue e religue completamente as duas
placas.

## Teste seguro sem movimento

Com o bridge já iniciado, valide o retorno do robô ID 2 enviando apenas STOP:

```bash
python3 tools/test_radio_telemetry_safe.py \
  --port "$BASE" --team 0 --robot 2 --frames 20 --rate-hz 20
```

O teste cria `telemetria_teste_seguro_*.txt`. O resultado esperado é
`TELEMETRY_TEST_PASS`; `team=0` significa azul. Como o primeiro comando
vincula a cor do receptor até a próxima inicialização, reinicie o robô antes de
trocar entre azul e amarelo.

## Uso normal

Inicie o bridge da base após cada ciclo de alimentação:

```bash
ROBOAP_SERIAL_PORT="$BASE" ./tools/start_base_channel125.sh
```

Em seguida, execute o Coach físico. Exemplo para o robô azul ID 2:

```bash
ROBOAP_SERIAL_PORT="$BASE" \
ROBOAP_SERIAL_ROBOT_IDS=2 \
./tools/run_physical_coach.sh blue
```

O terminal informa o nome completo do arquivo:

```text
[INFO] Telemetry log: /.../telemetria/telemetria_20260906_153000_123.txt
```

Para acompanhar o arquivo mais recente em outro terminal:

```bash
tail -f "$(ls -1t telemetria/telemetria_*.txt | head -n 1)"
```

## Colunas

| Coluna | Significado |
| --- | --- |
| `timestamp` | instante de recepção no notebook |
| `sequence` | sequência do comando confirmado pelo robô |
| `team` | `0` azul, `1` amarelo; `255` antes de vincular |
| `robot_id` | ID lógico persistente do robô |
| `interval_ms` | janela usada para medir os encoders |
| `left_target_eps` | referência da roda esquerda em bordas/s |
| `left_measured_eps` | velocidade medida da roda esquerda em bordas/s |
| `left_duty` | PWM aplicado à roda esquerda, de 0 a 65535 |
| `right_target_eps` | referência da roda direita em bordas/s |
| `right_measured_eps` | velocidade medida da roda direita em bordas/s |
| `right_duty` | PWM aplicado à roda direita, de 0 a 65535 |
| `flags` | estado do receptor em hexadecimal |

Bits de `flags`:

- `0x01`: saída dos motores habilitada;
- `0x02`: controle por encoder habilitado;
- `0x04`: comando de movimento ativo;
- `0x08`: watchdog detectou timeout.

O ACK transporta a amostra já pronta quando o comando seguinte chega, portanto
normalmente está uma interação atrás. Isso é esperado no nRF24L01+. Lacunas em
`sequence` indicam ACKs não registrados; uma linha só é gravada depois de
validar tamanho, versão, ID e CRC-16.


## Identificação dinâmica da planta

O controle de cada roda possui agora um estimador RLS de primeira ordem:

```text
velocidade[k] = a * velocidade[k-1] + b * PWM[k-1] + c
```

Ele é independente para esquerda e direita e atua somente no multiplicador do
feed-forward. O multiplicador começa em `1.0` após cada boot, muda lentamente e
fica limitado a `0.85..1.25`. `KP`, `KI`, teto de PWM, referência máxima,
watchdog e pinagem nunca são alterados automaticamente. Parada, reversão,
encoder parado, saturação e trechos sem variação suficiente de PWM são
ignorados, evitando que colisões ou rodas bloqueadas sejam aprendidas como
planta.

Para instalar apenas essa atualização, com o robô conectado por USB:

```bash
export ROBOAP_ROBOT_PORT=/dev/serial/by-id/usb-MicroPython_Board_in_FS_mode_...-if00
./tools/update_robot_adaptive_control.sh
```

O atualizador preserva `main.py` e `robot_identity.py` e deixa a placa no REPL.
O primeiro reinício deve ser feito com as rodas suspensas.

### Ensaio dedicado com rodas suspensas

Pare o Coach, inicie o bridge da base e mantenha o robô na bateria com as duas
rodas totalmente suspensas. Exemplo para equipe azul e ID de rádio 2:

```bash
ROBOAP_SERIAL_PORT="$BASE" ./tools/start_base_channel125.sh

python3 tools/collect_wheel_plant_identification.py \
  --port "$BASE" \
  --team 0 \
  --robot 2 \
  --confirm-wheels-suspended
```

O programa primeiro exige três ACKs de telemetria confirmando motor e encoder,
limita os comandos a `1500..7000`, executa no máximo 12 segundos, aborta após
500 ms sem telemetria e sempre envia oito frames STOP ao sair. São criados:

- `telemetria_identificacao_planta_*.txt`: todas as interações recebidas;
- `identificacao_planta_*.txt`: parâmetros estimados, erro, avisos e
  multiplicador sugerido/limitado.

Também é possível analisar arquivos já existentes sem movimentar o robô:

```bash
python3 tools/identify_wheel_plant.py \
  telemetria/telemetria_*.txt \
  --robot 2
```

Uma estimativa de zona morta negativa ou multiplicador fora de `0.85..1.25`
gera aviso e não deve ser copiada para `robot_config.py`; nesses casos, use o
ensaio dedicado para obter excitação suficiente.


Para validar toda a escala lógica até `10000`, continua valendo o teto interno
de PWM `40000`, mas é exigida uma segunda confirmação explícita:

```bash
python3 tools/collect_wheel_plant_identification.py \
  --port "$BASE" --team 0 --robot 2 \
  --levels 4000,6000,8000,10000 \
  --blocks 12 --dwell-ms 800 \
  --confirm-wheels-suspended \
  --confirm-full-command
```

Esse modo não aplica PWM bruto `65535`; `10000` é a referência lógica máxima
e todo PWM continua passando pelo controle por encoder e pelo limite de
produção.
