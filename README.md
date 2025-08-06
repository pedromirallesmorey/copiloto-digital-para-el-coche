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

  kia_mantenimiento_actual:
    name: Kia detalles de mantenimiento
    initial: "Sin mantenimiento registrado"

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

## 🧩 Paso 5 – Automatizaciones principales

🔹 automations.yaml:

📍 Al conectar al coche
```
automation:
  - alias: Kia ubicación al conectar coche
    description: ""
    triggers:
      - entity_id: binary_sensor.movil_pedro_coche
        to: "on"
        trigger: state
    actions:
      - action: script.kia_guardar_ubicacion_aprox
        data: {}
```

📍 Al desconectar del coche
```
  - alias: Kia ubicación al desconectar coche
    description: ""
    triggers:
      - entity_id: binary_sensor.movil_pedro_coche
        to: "off"
        trigger: state
    actions:
      - action: script.kia_guardar_ubicacion_aprox
        data: {}
```

📍 Cada 5 minutos conectado
🧠 Tiempo de registro

Elegir entre registrar la ubicación cada 5 minutos o siempre depende de lo que busques optimizar: batería, datos, precisión, o contexto.
Para mi caso, donde el objetivo es monitorizar el coche cuando estoy conectado, la opción cada 5 minutos mientras estés en el coche es perfecta.

Vamos a comparar:

🕒 Registrar cada 5 minutos
Ventajas:
- 🔋 Menor consumo de batería, especialmente en móviles.
- 🧠 Ideal para trayectos en coche, donde los cambios de ubicación suelen ser más significativos.
- 📁 Menos datos almacenados → base histórica más ligera.
- 
Ideal para:
- Seguimiento de trayectos.
- Automatizaciones activas solo durante viajes.
- Menor impacto en rendimiento del sistema.

♾️ Registrar siempre
Ventajas:
- 📍 Máxima precisión en el historial.
- 📊 Permite analizar trayectos a pie, cambios breves de ubicación o patrones de movimiento.
- 🛡️ Útil en escenarios de seguridad o emergencia.
- 
Desventajas:
- 🔋 Mayor consumo de batería en móviles.
- 🧠 Puede generar muchos datos innecesarios si el usuario está en casa o sin moverse.
- ⚙️ Mayor carga para Home Assistant si se usa constantemente.

Registro cada 5 min conectado:

```
automation:
  - alias: Kia registro cada 5 minutos conectado
    description: ""
    triggers:
      - minutes: /5
        trigger: time_pattern
    conditions:
      - condition: state
        entity_id: binary_sensor.movil_pedro_coche
        state: "on"
    actions:
      - action: script.kia_guardar_ubicacion_aprox
        data: {}
```

## 🧩 Paso 6 – Sensor de coche parado

### 🧠 ¿Cómo detectar una parada?

Podemos considerar “parada” cuando:
El móvil está conectado al Bluetooth del coche.
La ubicación no ha cambiado significativamente en los últimos X minutos (por ejemplo, menos de 30 metros en 3 minutos).
Vamos a crear un sensor que detecte si hubo movimiento. Este sensor calcula la distancia recorrida desde la última ubicación guardada. Si es menor a 30 metros, se considera “parado”.

```
sensor:
  - platform: template
    sensors:
      kia_coche_parado:
        friendly_name: "Kia coche parado"
        unique_id: kia_coche_parado
        icon_template: mdi:car-brake-hold
        value_template: >
          {% set lat1 = states('input_text.kia_ultima_latitud') | float(0) %}
          {% set lon1 = states('input_text.kia_ultima_longitud') | float(0) %}
          {% set lat2 = state_attr('device_tracker.sm_a536b', 'latitude') | float(0) %}
          {% set lon2 = state_attr('device_tracker.sm_a536b', 'longitude') | float(0) %}
          {% set r = 6371000 %}
          {% set dlat = ((lat2 - lat1) * 3.1416 / 180) %}
          {% set dlon = ((lon2 - lon1) * 3.1416 / 180) %}
          {% set lat1_rad = lat1 * 3.1416 / 180 %}
          {% set lat2_rad = lat2 * 3.1416 / 180 %}
          {% set a = (dlat / 2)**2 + cos(lat1_rad) * cos(lat2_rad) * (dlon / 2)**2 %}
          {% set c = 2 * (a + 1) ** 0.5 - 2 %}
          {% set distancia = r * c %}
          {{ 'si' if distancia < 30 else 'no' }}

```

📍 Solo registrar si está detenido

```
automation:
  - alias: "Kia registrar si está parado"
    trigger:
      - platform: time_pattern
        minutes: "/5"
    condition:
      - condition: state
        entity_id: binary_sensor.movil_pedro_coche_nombre
        state: "on"
      - condition: state
        entity_id: sensor.kia_coche_parado
        state: "si"
    action:
      - service: script.kia_guardar_ubicacion_aprox
```

Registra ubicación solo si estás conectado al coche y este no se mueve, lo cual ahorra batería, datos y mejora la calidad del historial. 🤖🗺️

## 🧩 Paso 7 – Alerta por parada prolongada

Vamos a añadir una función extra para que tu sistema te avise si el coche está detenido demasiado tiempo. Esto puede servir como indicador de aparcamiento, tiempo en ralentí, o incluso situaciones inusuales.

### ⏳ Sensor de parada prolongada

Primero, vamos a crear un sensor de duración desde la última vez que se detectó movimiento:

```
sensor:
  - platform: template
    sensors:
      kia_tiempo_coche_parado:
        friendly_name: "Kia tiempo coche parado"
        unique_id: kia_tiempo_coche_parado
        value_template: >
          {{ (as_timestamp(now()) - as_timestamp(states.binary_sensor.movil_pedro_coche_nombre.last_changed)) | int }}
        unit_of_measurement: "s"
        icon_template: mdi:car-brake-alert
```

Este sensor mide cuántos segundos han pasado desde el último cambio de estado (por ejemplo, desde que el coche se detuvo).

### 🚨 Automatización: alerta por parada prolongada

Ahora podemos lanzar una alerta si el coche está conectado y lleva más de X minutos detenido (ej. 10 minutos = 600 segundos):

```
  - alias: "Kia alerta parada larga"
    trigger:
      - platform: numeric_state
        entity_id: sensor.kia_tiempo_coche_parado
        above: 600
    condition:
      - condition: state
        entity_id: binary_sensor.movil_pedro_coche_nombre
        state: 'on'
    action:
      - service: notify.mobile_app_mi_movil
        data:
          title: "🚗 Coche detenido"
          message: "Más de 10 minutos parado en {{ states.device_tracker.sm_a536b.state }}"
```

🔔 Esta alerta puede ir a tu móvil, Telegram o incluso encender una luz si estás en casa.

## 🧩 Paso 8 – Zonas de aparcamiento y alerta

Puedes crear zonas en Home Assistant directamente en el mapa o en tu archivo zones.yaml o dentro del configuration.yaml. Por ejemplo:

```
zone:
  - name: Casa
    latitude: 39.6100
    longitude: 2.7000
    radius: 30
    icon: mdi:home

  - name: Trabajo
    latitude: 39.5850
    longitude: 2.6760
    radius: 30
    icon: mdi:office-building
```
Ajusta las coordenadas a tus ubicaciones reales. El radius define el área (en metros).

Si ya tienes tus zonas creadas en el mapa usando la interfaz de Home Assistant (la UI), entonces no necesitas declararlas manualmente en zones.yaml o configuration.yaml — ya están integradas como entidades del tipo zone. y se gestionan automáticamente.

Sensor:
Tu móvil ya está actuando como rastreador del coche. Puedes usar el atributo state del device_tracker.mi_movil para saber si está dentro de alguna zona:

### 📍 Sensor de zona actual
```
- platform: template
  sensors:
    kia_zona_coche_actual:
      friendly_name: "Kia zona coche actual"
      unique_id: kia_zona_coche_actual
      value_template: "{{ states('device_tracker.sm_a536b') }}"
      icon_template: "mdi:map-marker"
```
Esto mostrará directamente "Casa", "Trabajo", o "not_home" si está fuera de cualquier zona definida.

### 📍 Alerta si aparcas fuera de zona

Vamos a lanzar una notificación si aparcas en un lugar fuera de las zonas conocidas:

```
automation:
  - alias: "Kia alerta aparcamiento no reconocido"
    trigger:
      - platform: state
        entity_id: binary_sensor.movil_pedro_coche_nombre
        to: "off"
    condition:
      - condition: not
        conditions:
          - condition: state
            entity_id: device_tracker.sm_a536b
            state: "home"
          - condition: state
            entity_id: device_tracker.sm_a536b
            state: "gsaib"
      - condition: state
        entity_id: sensor.kia_coche_parado
        state: "si"
    action:
      - service: notify.mobile_app_sm_a536b
        data:
          title: "🚗 Aparcado fuera de zona"
          message: >
            Aparcado fuera de zona conocida. Ubicación: [Ver en mapa](https://maps.google.com/?q={{ state_attr('device_tracker.sm_a536b', 'latitude') }},{{ state_attr('device_tracker.sm_a536b', 'longitude') }})
```

## 🧩 Paso 9 – Activar “Modo Coche” con ambiente

Este modo se activa automáticamente cuando tu móvil se conecta al Bluetooth del coche. Puedes añadir tantas acciones como desees: luces, música, cambiar el estado de presencia, etc.

### 🎵 Automatización del “Modo Coche”

```
automation:
  - alias: "Kia activar modo coche"
    trigger:
      - platform: state
        entity_id: binary_sensor.movil_pedro_coche_nombre
        to: "on"
    action:
      - service: notify.mobile_app_sm_a536b
        data:
          title: "🚗 Modo Coche Activado"
          message: "¡Modo coche activado!"
      # Ejemplo: enciende la luz del salón
      - service: light.turn_on
        target:
          entity_id: light.salon
      # Ejemplo: inicia música en Spotify
      - service: media_player.play_media
        target:
          entity_id: media_player.spotify_pedro
        data:
          media_content_id: "spotify:playlist:37i9dQZF1DXcBWIGoYBM5M"
          media_content_type: "music"
      # Aquí puedes añadir más acciones, como abrir el garaje, cambiar estado de presencia, etc.
```
Cambia las entidades por las tuyas: light.salon, media_player.spotify_pedro, etc.

💡 Consejo: Puedes añadir más servicios como activar el panel del coche, abrir el garaje, etc.

## 🧩 Paso 10 – Registro de mantenimiento e historial

### Script para guardar eventos de mantenimiento
```
script:
  kia_registrar_mantenimiento_completo:
    alias: Kia registrar mantenimiento completo
    sequence:
      - variables:
          nuevo_registro: >
            {{ now().strftime('%d-%m-%Y %H:%M') }} | Km: {{ states('input_number.kia_kilometros_actuales') }} | {{ states('input_text.kia_detalle_mantenimiento') }}
      - service: input_text.set_value
        data:
          entity_id: input_text.kia_mantenimiento_actual
          value: >
            {{ nuevo_registro }}{{ '\n' }}{{ states('input_text.kia_mantenimiento_actual') }}
```
### 🧩 ¿Qué hace?
- 📅 Fecha y hora: la del momento en que se registra.
- 🔢 Kilómetros actuales: del coche, al momento del registro.
- 📝 Detalles del mantenimiento: escritos por ti antes de ejecutar el script.
- 📚 Historial acumulado: cada registro se añade al principio del texto, manteniendo todo lo anterior.

### Automatización para lanzar el script tras confirmar el mantenimiento

Automatización para lanzar el script de registro de mantenimiento cuando confirmas un nuevo mantenimiento (por ejemplo, al cambiar el valor de input_text.kia_detalle_mantenimiento).
Se asume que confirmas el mantenimiento introduciendo un texto en el campo (puedes adaptarlo si usas un botón o helper diferente):

```
automation:
  - alias: "Kia - Guardar nuevo evento de mantenimiento"
    trigger:
      - platform: state
        entity_id: input_text.kia_detalle_mantenimiento
        to: ~  # Detecta cualquier cambio de valor
    condition:
      - condition: template
        value_template: "{{ states('input_text.kia_detalle_mantenimiento') | length > 3 }}"
    action:
      - service: script.kia_registrar_mantenimiento_completo
      # (Opcional) Limpia el campo tras registrar el mantenimiento
      - service: input_text.set_value
        data:
          entity_id: input_text.kia_detalle_mantenimiento
          value: ""
```

### Script para registrar un nuevo evento de mantenimiento en el historial de tu coche

El script script.kia_registrar_mantenimiento_completo es el encargado de registrar un nuevo evento de mantenimiento en el historial de tu coche, combinando la fecha, los kilómetros actuales y la descripción del mantenimiento que introduzcas.

```
kia_registrar_mantenimiento_completo:
  alias: Kia registrar mantenimiento completo
  sequence:
    - variables:
        nuevo_registro: >
          {{ now().strftime('%d-%m-%Y %H:%M') }} | Km: {{ states('input_number.kia_kilometros_actuales') }} | {{ states('input_text.kia_detalle_mantenimiento') }}
    - service: input_text.set_value
      data:
        entity_id: input_text.kia_mantenimiento_actual
        value: >
          {{ nuevo_registro }}{{ '\n' }}{{ states('input_text.kia_mantenimiento_actual') }}
```
¿Qué hace este script?
Crea una cadena con la fecha/hora actual, los kilómetros actuales y la descripción del mantenimiento (introducida previamente).
Añade ese registro al principio del historial (input_text.kia_mantenimiento_actual), manteniendo los registros anteriores debajo.
El resultado: El historial muestra los mantenimientos más recientes primero.

Requisitos
Debes tener estos helpers creados (puedes hacerlo desde la UI o en YAML):

```
input_text:
  kia_mantenimiento_actual:
    name: Kia detalles de mantenimiento
    initial: "Sin mantenimiento registrado"
  kia_detalle_mantenimiento:
    name: Kia registrar nuevo mantenimiento
    initial: ""
input_number:
  kia_kilometros_actuales:
    name: Kilómetros actuales
    initial: 0
    min: 0
    max: 1000000
```

Para lanzar el script kia_registrar_mantenimiento_completo desde una automatización en Home Assistant, solo necesitas una acción que llame al servicio script.kia_registrar_mantenimiento_completo.

```
automation:
  - alias: "Kia - Guardar nuevo evento de mantenimiento"
    trigger:
      - platform: state
        entity_id: input_text.kia_detalle_mantenimiento
        to: ~  # Se dispara al cambiar el texto
    condition:
      - condition: template
        value_template: "{{ states('input_text.kia_detalle_mantenimiento') | length > 3 }}"
    action:
      - service: script.kia_registrar_mantenimiento_completo
      - service: input_text.set_value
        data:
          entity_id: input_text.kia_detalle_mantenimiento
          value: ""
```

Explicación:

Trigger: Cuando cambias el valor del campo de detalle de mantenimiento.
- Condition: Solo si el texto tiene más de 3 caracteres (puedes ajustar este filtro).
- Action:
  - Llama al script kia_registrar_mantenimiento_completo.
  - Opcionalmente, borra el campo de texto para dejarlo listo para el siguiente mantenimiento.
  - 
Puedes lanzar el script desde cualquier automatización usando la acción:
```
action:
  - service: script.kia_registrar_mantenimiento_completo
```




















