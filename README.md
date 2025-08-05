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

🔹 Opción 1: Detección por MAC
Este sensor detecta si tu móvil está conectado al Bluetooth del coche. Se basa en la MAC del coche. Cambia sensor.sm_a536b_bluetooth_connection según tu móvil.

```
binary_sensor:
  - platform: template
    sensors:
      movilpedrocoche:
        friendly_name: "Móvil Pedro coche"
        value_template: >
          {% if 'A0:6F:AA:90:90:32' in state_attr('sensor.sm_a536b_bluetooth_connection','connected_paired_devices') %} on
          {% else %} off
          {% endif %}
        icon_template: mdi:car
```

🎯 Qué estamos usando como referencia de “coche conectado”:
- Tu móvil con Home Assistant Companion App.
- Estado del Bluetooth conectado al coche.

🔹 Opción 2: Usando nombre Bluetooth
Si no quieres depender de la dirección MAC del Bluetooth del coche, puedes construir el binary_sensor.movilpedrocoche basándote directamente en otro estado más genérico, como el sensor de conexión Bluetooth que expone la app de Home Assistant Companion en tu móvil Android.
Aquí te dejo un ejemplo que no usa la MAC, sino que detecta si tu móvil tiene algún dispositivo conectado por Bluetooth:
```
template:
  - binary_sensor:
      - name: "Móvil Pedro coche"
        unique_id: movilpedrocoche_sensor
        device_class: connectivity
        state: >
          {{ state_attr('sensor.sm_a536b_bluetooth_connection', 'connected_paired_devices') | length > 0 }}
        icon: mdi:car
```

🧠 Explicación:
- sensor.sm_a536b_bluetooth_connection → esta entidad debería ser la que tu móvil expone con los dispositivos Bluetooth conectados (verifica su nombre en Developer Tools).
- La plantilla verifica si hay al menos un dispositivo emparejado y conectado, lo cual suele ser tu coche cuando estás dentro.
- Esto sirve si el coche es el único dispositivo que se conecta regularmente.

🔹 Opción 3 filtrar por nombre del dispositivo conectado

🔍 ¿Cómo mejorar la precisión?

Si tu móvil a veces se conecta a otros dispositivos (auriculares, altavoz, etc.), podemos filtrar por nombre del dispositivo conectado:

```
template:
  - binary_sensor:
      - name: "Móvil Pedro coche"
        unique_id: movilpedrocoche_sensor
        device_class: connectivity
        state: >
          {{ 'KIA MOTORS' in state_attr('sensor.sm_a536b_bluetooth_connection', 'connected_paired_devices') }}
        icon: mdi:car
```

Esto funciona si el nombre del coche en el Bluetooth es "KIA MOTORS" y aparece tal cual en los atributos del sensor.

🔍 Explicación:

- Revisa si 'KIA MOTORS' está en la lista de dispositivos conectados.
- Si está, el sensor se activa (on), indicando que el coche está conectado.
- Puedes cambiar 'KIA MOTORS' por cualquier nombre que aparezca en tu lista.

🔹 Opción 4: Varios nombres simultáneos
Si tienes varios dispositivos Bluetooth que podrían representar tu coche (por ejemplo, "KIA MOTORS", "PedroCar", "MiCoche"), podemos crear un sensor que detecte si cualquiera de esos nombres está conectado a tu móvil.

```
template:
  - binary_sensor:
      - name: "Móvil Pedro coche"
        unique_id: movilpedrocoche_sensor
        device_class: connectivity
        state: >
          {% set nombres_coche = ['KIA MOTORS', 'PedroCar', 'MiCoche'] %}
          {% set conectados = state_attr('sensor.sm_a536b_bluetooth_connection', 'connected_paired_devices') %}
          {{ conectados is not none and nombres_coche | select('in', conectados) | list | length > 0 }}
        icon: mdi:car
```

