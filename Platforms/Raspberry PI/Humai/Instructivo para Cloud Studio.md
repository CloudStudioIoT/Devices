# Cómo vincular la Raspberry Pi Pico W con Cloud Studio empleando MicroPython

**Cloud Studio** cuenta con todos los recursos necesarios para ofrecer una solución integral a los profesionales que trabajan en el ámbito de **IoT**, permitiendo la creación de notificaciones y alarmas, y la elaboración de paneles de visualización para mostrar información en tiempo real sobre el rendimiento y estado de los **dispositivos IoT** que deseen vincularse con ella. 

Para ilustrar esto, mostraremos un ejemplo práctico con la placa de desarrollo **Raspberry Pi Pico W (RPico W)**, monitoreando la temperatura interna de la misma y realizando el envío de datos correspondientes a la plataforma **Cloud Studio** a través del protocolo **HTTP**. Esto nos permitirá generar gráficos que representen los valores históricos y actuales de la variable que estamos monitoreando. 

Comenzaremos por incluir las líneas de código necesarias para establecer la conexión de la **RPico W** a una red **WiFi**. Para ello, necesitaremos utilizar la librería *network*, que nos proporciona las herramientas necesarias para la configuración y gestión de redes en dispositivos que ejecutan **MicroPython**. 

Para organizar los pasos de manera eficiente, definiremos una función llamada *connect()* para manejar la conexión a la red **WiFi**, e implementaremos una estructura de manejo de excepciones *try/except* para gestionar posibles errores.

También incluiremos la configuración del **Convertidor Analógico a Digital** (*ADC*, por sus siglas en inglés, *Analogic-to-Digital Converter*) que se encuentra conectado al sensor de temperatura interno de la **RPico W**, junto con un *factor de conversión* que establece una forma matemática de convertir el número que arroja el **ADC** en una aproximación justa del voltaje real que representa. Posteriormente, agregaremos las líneas de código necesarias para realizar la lectura efectiva del sensor. Tengamos en cuenta que esta configuración debe ajustarse de acuerdo al sensor que estemos utilizando para nuestro proyecto **IoT**.

Esta primera parte del código completo queda entonces de la siguiente manera:

```python
import network
from machine import Pin, ADC
from utime import sleep

ssid = 'CAMBIA POR TU SSID'
password = 'TU PASSWORD'

sensor_temp = ADC(4)
factor_conversion = 3.3 / (65535)

def connect():
    red = network.WLAN(network.STA_IF)
    red.active(True)
    red.connect(ssid,password)
    while red.isconnected() == False :
        print("Estableciendo conexión..")
        sleep(1)
    
    ip = red.ifconfig()[0]
    print("Conexión Establecida")
    print(red.ifconfig())
    return ip

try:
    ip = connect()
except KeyboardInterrupt:
    machine.reset()
```

Por otro lado, en **MicroPython**, la librería *urequests* es utilizada para realizar solicitudes HTTP a través de internet. Esta librería permite a dispositivos que utilizan **MicroPython**, como la **RPico W**, interactuar con servicios web y acceder a recursos remotos, como **Cloud Studio** en este caso.

La librería *urequests* simplifica el proceso de envío de solicitudes GET, POST, PUT o DELETE a URLs específicas, así como el manejo de respuestas y datos recibidos. Al utilizar *urequests*, los dispositivos con recursos limitados pueden aprovechar la funcionalidad de comunicación con servicios web de manera eficiente y efectiva.

Para comenzar, importaremos la librería *urequests* junto con las librerías cargadas anteriormente:

```python
import network
import urequests as req
from machine import Pin, ADC
from utime import sleep
```

Ahora procederemos a integrar nuestro código con la plataforma **Cloud Studio**. Para ello, comenzaremos por utilizar dos datos que son fundamentales para interactuar con una **plataforma IoT** y acceder a sus servicios: el *access_token* y los *endpointID*.

El *access_token* es una credencial de seguridad que se utiliza para autenticar y autorizar el acceso a la **plataforma IoT**. Por otro lado, los *endpoints* son las direcciones a través de las cuales le podemos enviar solicitudes a la API de la **plataforma IoT**. Estos *endpoints* se representan como URLs específicas que indican la ubicación de un servicio o recurso en la plataforma.

Recordemos que previamente debemos crear nuestro dispositivo en la plataforma (en nuestro caso la **RPico W**) y el/los endpopints correspondientes a la variable que deseamos monitorear (en nuestro caso la temperatura interna).

En este caso, para monitorear la temperatura de nuestra **RPico W**, definiremos lo siguiente:

```python
# Esto se obtiene de la wiki de Cloud Studio
temperature_url = 'https://gear.cloud.studio/services/gear/DeviceIntegrationService.svc/UpdateTemperatureSensorStatus'

access_token = 'COLOCA TU ACCESS TOKEN'

internal_temperature = 'COLOCA EL ENDPOINT ID CORRESPONDIENTE AL SENSOR INTERNO DE TEMPERATURA'
```

A continuación, crearemos el *payload*, que representa el conjunto de datos que se envían en una solicitud **HTTP**. En este caso, se estructurará de la siguiente manera:

```python
payload_temperature = {
	'accessToken': access_token, # Se repite en todos los payloads que realicemos
	'endpointID': internal_temperature, # Es un numero entero y se modifica de acuerdo al sensor que utilicemos
	'temperatureCelsius': 30 # Valor inicial aleatorio
}
```

Y ahora definiremos una función *enviar_datos()* que realice efectivamente la transmisión de los datos a la plataforma **Cloud Studio**:

```python
def enviar_datos(ip):
    while True:
        # Realizo la lectura correspondiente
        lectura = sensor_temp.read_u16() * factor_conversion
        temperature = 27 - (lectura - 0.706) / 0.001721
        
        # La agrega el dato al payload
        payload_temperature['temperatureCelsius'] = temperature
        print('Tomando temperatura del sensor interno', payload_temperature['temperatureCelsius'])
        
        # Le envío al servidor y aguardo la respuesta del servidor (200)
        response = req.post(temperature_url, json = payload_temperature)
        print('Respuesta del servidor: ', response.status_code)
        response.close()
        sleep(1)
```

Además, incorporaremos la función *enviar_datos()* dentro de la estructura de manejo de excepciones *try/except* para gestionar posibles errores.

```python
try:
    ip = connect()
    enviar_datos(ip)
except KeyboardInterrupt:
    machine.reset()
```

El código completo queda de la siguiente manera:

```python
import network
import urequests as req	
from machine import Pin, ADC
from utime import sleep

ssid = 'CAMBIA POR TU SSID'
password = 'TU PASSWORD'

sensor_temp = ADC(4)
factor_conversion = 3.3 / (65535)

# Esto se obtiene de la wiki de Cloud Studio
temperature_url = 'https://gear.cloud.studio/services/gear/DeviceIntegrationService.svc/UpdateTemperatureSensorStatus'

access_token = 'COLOCA TU ACCESS TOKEN'

internal_temperature = 'COLOCA EL ENDPOINT ID CORRESPONDIENTE AL SENSOR INTERNO DE TEMPERATURA'

payload_temperature = {
	'accessToken': access_token, # Se repite en todos los payloads que realicemos
	'endpointID': internal_temperature, # Es un numero entero y se modifica de acuerdo al sensor que utilicemos
	'temperatureCelsius': 30 # Valor inicial aleatorio
}

def connect():
    red = network.WLAN(network.STA_IF)
    red.active(True)
    red.connect(ssid,password)
    while red.isconnected() == False :
        print("Estableciendo conexión..")
        sleep(1)
    
    ip = red.ifconfig()[0]
    print("Conexión Establecida")
    print(red.ifconfig())
    return ip

def enviar_datos(ip):
    while True:
        # Realizo la lectura correspondiente
        lectura = sensor_temp.read_u16() * factor_conversion
        temperature = 27 - (lectura - 0.706) / 0.001721
        
        # La agrega el dato al payload
        payload_temperature['temperatureCelsius'] = temperature
        print('Tomando temperatura del sensor interno', payload_temperature['temperatureCelsius'])
        
        # Le envío al servidor y aguardo la respuesta del servidor (200)
        response = req.post(temperature_url, json = payload_temperature)
        print('Respuesta del servidor: ', response.status_code)
        response.close()
        sleep(1)
        
try:
    ip = connect()
    enviar_datos(ip)
except KeyboardInterrupt:
    machine.reset()
```

Si el dato se envió correctamente, deberíamos ver el código HTTP *200* en la consola de nuestro compilador, como se muestra la **Figura 01**. Esto confirma una comunicación adecuada con la plataforma.

![Figura 01 - Comunicación efectiva de los datos a Cloud Studio](./images/Figura01-ComunicaciónEfectivaDeLosDatosACloudStudio.jpg)  
*Figura 01 - Comunicación efectiva de los datos a Cloud Studio*

Concluido esto, ya tenemos todo listo para comenzar a desarrollar nuestros tableros en **Cloud Studio**.