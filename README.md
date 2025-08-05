# 🚗 Copiloto Digital para el Coche
Este proyecto está pensado para los que “no tenemos vehículos inteligentes”, y queremos transformar nuestro Home Assistant en un asistente inteligente para el vehículo, capaz de:
- Detectar conexión al coche vía Bluetooth.
- Registrar ubicación durante trayectos.
- Calcular kilómetros recorridos.
- Detectar paradas prolongadas.
- Enviar alertas geográficas.
- Activar “modo coche” con luces y música.
- Llevar control completo de mantenimiento: ITV, seguro...
- Mostrar todo en un panel visual personalizado.

Todo configurado con lógica eficiente para ahorrar batería, mejorar el contexto histórico y facilitar el mantenimiento del vehículo.

## 🧭 Paso 1 – Sensor de conexión Bluetooth al coche
El sensor detecta si el móvil está conectado al Bluetooth del coche. Puedes elegir según tu preferencia.

### 🔹 Opción 1: Detección por MAC
Este sensor detecta si tu móvil está conectado al Bluetooth del coche. Se basa en la MAC del coche. Cambia sensor.sm_a536b_bluetooth_connection según tu móvil.

```
binary_sensor:
  - platform: template
    sensors:
      movilpedrocoche:
        friendly_name: "Móvil Pedro coche"
        value_template: >
          {{ 'A0:6F:AA:90:90:32' in (state_attr('sensor.sm_a536b_bluetooth_connection', 'connected_paired_devices') | join(' ')) }}
        icon_template: mdi:car
```

🎯 Qué estamos usando como referencia de “coche conectado”:
- Tu móvil con Home Assistant Companion App.
- Estado del Bluetooth conectado al coche.

### 🔹 Opción 2 filtrar por nombre del dispositivo conectado

🔍 ¿Cómo mejorar la precisión?

Si tu móvil a veces se conecta a otros dispositivos (auriculares, altavoz, etc.), podemos filtrar por nombre del dispositivo conectado:

```
template:
  - binary_sensor:
      - name: "Móvil Pedro coche"
        unique_id: movilpedrocoche_sensor
        device_class: connectivity
        state: >
          {{ 'KIA MOTORS' in (state_attr('sensor.sm_a536b_bluetooth_connection', 'connected_paired_devices') | join(' ')) }}
        icon: mdi:car
```

Esto funciona si el nombre del coche en el Bluetooth es "KIA MOTORS" y aparece tal cual en los atributos del sensor.

🔍 Explicación:

- Revisa si 'KIA MOTORS' está en la lista de dispositivos conectados.
- Si está, el sensor se activa (on), indicando que el coche está conectado.
- Puedes cambiar 'KIA MOTORS' por cualquier nombre que aparezca en tu lista.
- | join(' ') une todos los elementos de la lista en un solo string, para que puedas buscar fácilmente "KIA MOTORS" aunque esté junto a la MAC.


## 🧩 Paso 2 – Helpers para kilometraje y ubicación

Define variables que guardan la posición y los kilómetros acumulados. Calculamos la distancia entre cada punto registrado y lo sumamos para obtener un estimado de kilómetros recorridos.

🔢 Método:
- Usar un script que compare la ubicación actual con la anterior.
- Calcular la distancia usando la fórmula de Haversine.
- Acumular los kilómetros en un counter.

🔹 Añade estos input_text e input_number a tu configuration.yaml o desde Helpers en la UI:

```
input_text:
  kia_ultima_latitud:
    name: Última latitud
    initial: "0"

  kia_ultima_longitud:
    name: Última longitud
    initial: "0"

  kia_ultima_ubicacion_coche:
    name: Última ubicación coche
    initial: "Sin datos"

  kia_detalle_mantenimiento:
    name: Detalle del mantenimiento
    initial: ""

input_number:
  kia_kilometros_recorridos:
    name: Kia kilómetros recorridos
    initial: 0
    min: 0
    max: 1_000_000
    step: 0.1
    mode: box

  kia_kilometros_actuales:
    name: Kia kilómetros del coche
    initial: 150000
    min: 0
    max: 500000
    step: 100
    mode: box

  kia_kilometros_respaldo:
    name: Kilómetros respaldados
    min: 0
    max: 500000
    step: 1
    mode: box
```

En kilómetros actuales --> Initial ponemos los kilómetros que tiene el vehículo en el momento de iniciar todo este proceso.

## 🧩 Paso 3 – Script de registro y cálculo de distancia

Calcula los kilómetros y actualiza datos:
Método aproximado: Fórmula de distancia Euclídea simplificada
Este método calcula la distancia en kilómetros con una aproximación lineal, válida para distancias menores a 1–2 km, y suficiente si estás usando el script cada 5 minutos para trayectos.

🔹 scripts.yaml:

```
alias: "Kia guardar ubicación (cálculo aproximado)"
sequence:
  - variables:
      lat1: "{{ states('input_text.kia_ultima_latitud') | float }}"
      lon1: "{{ states('input_text.kia_ultima_longitud') | float }}"
      lat2: "{{ state_attr('device_tracker.sm_a536b', 'latitude') | float }}"
      lon2: "{{ state_attr('device_tracker.sm_a536b', 'longitude') | float }}"
      delta_lat: "{{ lat2 - lat1 }}"
      delta_lon: "{{ lon2 - lon1 }}"
  - service: input_number.set_value
    data:
      entity_id: input_number.kia_kilometros_recorridos
      value: >
        {% set distancia_aprox = ((delta_lat * 111)**2 + (delta_lon * 85)**2)**0.5 %}
        {{ (states('input_number.kia_kilometros_recorridos') | float + distancia_aprox) | round(2) }}
  - service: input_text.set_value
    target:
      entity_id: input_text.kia_ultima_latitud
    data:
      value: "{{ lat2 }}"
  - service: input_text.set_value
    target:
      entity_id: input_text.kia_ultima_longitud
    data:
      value: "{{ lon2 }}"
  - service: input_text.set_value
    data:
      entity_id: input_text.kia_ultima_ubicacion_coche
      value: >
        {{ now().strftime('%H:%M') }} → {{ lat2 }}, {{ lon2 }}
```

Cambiar device_tracker.sm_a536b, por tu tracker móvil.

🧠 Consideraciones

- Este método usa valores fijos: 111 km/°lat y 85 km/°lon, que son válidos en latitudes medias (como España).
- La fórmula es suficiente para trayectos urbanos y seguimiento cada pocos minutos.
- Evitas funciones incompatibles (sin, cos, radians, etc.).

## 🧩 Paso 4 – Módulo de Protección de Kilómetros

Esto mantendrá los datos confiables, detectará errores y permitirá restaurarlos automáticamente. Evita pérdidas o caídas de kilometraje.

### 4.1 Sensor de validación visual

Detecta si el valor actual es sospechoso (como 0.0) y muestra un estado claro en el panel.

```
- platform: template
  sensors:
    kia_kilometros_estado:
      friendly_name: Estado de kilómetros
      unique_id: kia_kilometros_estado
      value_template: >
        {% set km = states('input_number.kia_kilometros_actuales') | float(0) %}
        {% if km == 0 %}
          Sin datos válidos
        {% else %}
          {{ km }} km
        {% endif %}
      icon_template: >
        {% if states('input_number.kia_kilometros_actuales') | float(0) == 0 %}
          mdi:alert
        {% else %}
          mdi:shield-check
        {% endif %}
```

### 4.2 Automatización para respaldo automático

Esa automatización está diseñada para crear un respaldo (una copia de seguridad) del valor actual de los kilómetros del coche cada vez que cambian, siempre que el nuevo valor sea mayor que cero.

🔍 ¿Qué hace cada parte?

Elemento => Función
trigger => Se activa cuando cambian los kilómetros (kia_kilometros_actuales).
condition => Sólo continúa si el nuevo valor es mayor a 0.
action => Copia el valor actual a kia_kilometros_respaldo (la entidad respaldo).

🧩 ¿Por qué es útil?

- Protege tu dato clave de kilometraje si Home Assistant se reinicia o si por error el valor se pone en cero.
- Puedes usar kia_kilometros_respaldo en un script de restauración, por ejemplo, si detectas que kia_kilometros_actuales se ha reiniciado.
- También sirve para comprobar cambios de kilometraje, sin necesidad de registro externo.

```
automation:
  - alias: "Actualizar respaldo de kilómetros"
    trigger:
      - platform: state
        entity_id: input_number.kia_kilometros_actuales
    condition:
      - condition: template
        value_template: "{{ trigger.to_state.state | float(0) > 0 }}"
    action:
      - service: input_number.set_value
        data:
          entity_id: input_number.kia_kilometros_respaldo
          value: "{{ states('input_number.kia_kilometros_actuales') }}"
```

### 4.3 Automatización de restauración en caso de error

Detecta si el valor actual de los kilómetros se reinicia a 0 o a un número menor que el respaldo, y entonces los restaura automáticamente usando el respaldo guardado 🚗🔧:

🧩 ¿Qué hace?

- Disparador: Reacciona cada vez que cambia el valor de kia_kilometros_actuales.
- Condición: Verifica si el nuevo valor es menor que el respaldo (incluyendo 0).
- Acción: Restaura el valor correcto desde kia_kilometros_respaldo.

```
alias: Restaurar kilómetros desde respaldo
triggers:
  - entity_id: input_number.kia_kilometros_actuales
    trigger: state
conditions:
  - condition: template
    value_template: |
      {{ states('input_number.kia_kilometros_actuales') | float(0) <
          states('input_number.kia_kilometros_respaldo') | float(0) }}
actions:
  - data:
      entity_id: input_number.kia_kilometros_actuales
      value: "{{ states('input_number.kia_kilometros_respaldo') }}"
    action: input_number.set_value
  - action: telegram_bot.send_message
    metadata: {}
    data:
      config_entry_id: 01JZ81RV8MR09SQ***********
      message: >-
        🚘 Se ha restaurado el valor de kilómetros desde el respaldo.          
        Valor corregido: {{ states('input_number.kia_kilometros_respaldo') }}
        km.
```

### 4.4 Dashboard visual para el panel

Crea una tarjeta tipo vertical-stack para monitorizar los datos del módulo:

```
type: vertical-stack
cards:
  - type: entities
    title: 🚘 Protección de Kilómetros - Kia Rio
    entities:
      - entity: input_number.kia_kilometros_actuales
        name: Kilómetros actuales
        icon: mdi:car-speedometer
      - entity: input_number.kia_kilometros_respaldo
        name: Kilómetros respaldo
        icon: mdi:backup-restore
      - entity: sensor.kia_kilometros_estado
        name: Estado del sensor
        icon: mdi:shield-check
```





