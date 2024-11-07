Do uruchomienia mojego projektu termometra w serwerowni użyłem modułu ESP8266  dokładniej Wemos D1 mini oraz czujnika DHT11, koszt całego zestawu to jakieś 25PLN 
zainstalowałem na nim oprogramowanie Tasmota http://ota.tasmota.com/tasmota/release-9.1.0/ 
Po połaczeniu modułu do WiFi dodajemy na nim sam czujnik oraz adres IP brokera MQTT


1. **Instalacja i konfiguracja serwera Mosquitto**

   Upewnij się, że masz zainstalowany i działający broker MQTT (Mosquitto). Jeśli jeszcze go nie zainstalowałeś, wykonaj:

   sudo apt update
   sudo apt install -y mosquitto mosquitto-clients

   Sprawdź, czy Mosquitto działa:

   systemctl status mosquitto

   Jeśli potrzebujesz niestandardowej konfiguracji, edytuj plik `/etc/mosquitto/mosquitto.conf` zgodnie z wymaganiami i uruchom ponownie Mosquitto:

   Przechodzimy do ustawień dostępowych Mosquitto aby nasze urządzenia mogły sie do nioego dobić 
   nano /etc/mosquitto/conf.d/default.conf
   wklejamy tam :

   allow_anonymous true

   #HTTP listener
   listener 1883

   #Websocket standard listener
   listener 9001
   protocol websockets



 - Robimy restart 
   sudo systemctl restart mosquitto

2. **Instalacja Mosquitto MQTT na kliencie Python dla Zabbixa**

   Zabbix nie obsługuje natywnie MQTT, więc konieczne jest skonfigurowanie klienta Python, który odbierze dane MQTT i przekaże je do Zabbix. Zainstaluj `paho-mqtt` i `zabbix-sender`:

   sudo apt install -y python3-pip zabbix-sender
   pip3 install paho-mqtt


3. **Utworzenie skryptu Python do odbioru danych MQTT i wysyłki do Zabbix**

   Utwórz skrypt, który odbiera dane z Mosquitto i wysyła je do Zabbix. Poniżej znajduje się mój skrypt.

   nano /usr/local/bin/mqtt_to_zabbix.py

   wklej do niego poniższy kod:

import paho.mqtt.client as mqtt
import subprocess
import json

# Konfiguracja
MQTT_BROKER = "localhost"  # Adres Mosquitto (zmień na IP serwera, jeśli inny niż localhost)
MQTT_PORT = 1883
MQTT_TOPIC = "tele/tasmota/SENSOR"   # Ścieżka MQTT do odbioru danych z czujników

ZABBIX_SERVER = "127.0.0.1"  # Adres serwera Zabbix
ZABBIX_HOST = "mqtt_sensors" # Nazwa hosta w Zabbix

# Funkcja wysyłająca dane do Zabbix
def send_to_zabbix(key, value):
    try:
        # Tylko liczby wysyłamy
        if isinstance(value, (int, float)):
            subprocess.call(["zabbix_sender", "-z", ZABBIX_SERVER, "-s", ZABBIX_HOST, "-k", key, "-o", str(value)])
            print(f"Sent to Zabbix: {key} = {value}")
        else:
            print(f"Skipping invalid data: {key} = {value}")
    except Exception as e:
        print(f"Error sending data to Zabbix: {e}")

# Funkcja obsługująca odebrane wiadomości MQTT
def on_message(client, userdata, msg):
    try:
        # Odbierz dane w formacie JSON
        payload = json.loads(msg.payload.decode('utf-8'))
        print(f"Received data: {payload}")  # Debugowanie, aby sprawdzić, jakie dane przychodzą

        # Sprawdzamy, czy mamy zagnieżdżony obiekt DHT11
        if 'DHT11' in payload:
            # Wysyłamy dane z DHT11: Temperature, Humidity i DewPoint
            send_to_zabbix("DHT11.Temperature", payload['DHT11'].get('Temperature'))
            send_to_zabbix("DHT11.Humidity", payload['DHT11'].get('Humidity'))
            send_to_zabbix("DHT11.DewPoint", payload['DHT11'].get('DewPoint'))

        # Możesz także wysłać inne dane z JSON, jeśli chcesz (np. Time, TempUnit)
        send_to_zabbix("Time", payload.get("Time"))
        send_to_zabbix("TempUnit", payload.get("TempUnit"))

    except Exception as e:
        print(f"Error processing message: {e}")

# Konfiguracja klienta MQTT
client = mqtt.Client()
client.on_message = on_message

client.connect(MQTT_BROKER, MQTT_PORT, 60)
client.subscribe(MQTT_TOPIC)

# Uruchomienie klienta MQTT
client.loop_forever()


4. **Uruchomienie skryptu Python**

   Nadaj skryptowi uprawnienia do uruchamiania:


   cd /usr/local/bin/  
   chmod +x mqtt_to_zabbix.py

   Uruchom skrypt i sprawdź, czy odbiera i przesyła dane poprawnie:

   python3 mqtt_to_zabbix.py

   Możesz skonfigurować ten skrypt jako usługę systemd, aby uruchamiał się automatycznie po restarcie serwera.

5. **Tworzenie usługi systemd dla skryptu**

   Utwórz plik `mqtt_to_zabbix.service` w `/etc/systemd/system/`:

   nano /etc/systemd/system/mqtt_to_zabbix.service

   [Unit]
   Description=MQTT to Zabbix Service
   After=network.target

   [Service]
   ExecStart=/usr/bin/python3 /usr/local/bin/mqtt_to_zabbix.py
   Restart=always

   [Install]
   WantedBy=multi-user.target

 - Zapisz plik i wykonaj poniższe polecenia, aby aktywować usługę:

   sudo systemctl daemon-reload
   sudo systemctl enable mqtt_to_zabbix.service
   sudo systemctl start mqtt_to_zabbix.service

 - Możesz sprawdzić, czy usługa działa, używając:

   sudo systemctl status mqtt_to_zabbix.service

6. **Konfiguracja Zabbix**

   - W Zabbix utwórz host o nazwie zgodnej z `ZABBIX_HOST` (w moim przykładzie jest to `mqtt_sensors`). 
     Grupa hostów według uznania, dodajemy interfejs typu agent z adresem IP 127.0.0.1 i portem 10050

   - Dodaj odpowiednie elementy danych Itemy (Pozycje) w hostach Zabbix, które będą odpowiadały nazwom `key` w MQTT. 
     Typy elementów danych ustaw na `Trapper Zabbix`, ponieważ będą odbierane z `zabbix_sender`, 
     w polu Klucz w moim przypadku Temperatury ustawiamy DHT11.Temperature, Wilgotność DHT11.Humidity itp. 
   
   - Dodajemy wykresy w polu nazwa wpisujemy dla przykładu MQTT Temperature, przechodzimy do dodania danych wybieramy Pozycję wybieramy naszego hosta mqtt_sensors i zaznaczamy temperaturę .


 - Wszytko powinno śmigać

   Test sendera do zabbix, wysłanie samych danych temperatury które powinny dotrzeć do wykresu:
 
   zabbix_sender -z 127.0.0.1 -s "mqtt_sensors" -k "DHT11.Temperature" -o 22.9      

   Możemy też sprawdzić czy działa połączenie sendera z brokerem MQTT używając komendy :

   mosquitto_pub -h 127.0.0.1 -t "tele/tasmota/SENSOR" -m "{\"Time\":\"2024-11-06T08:52:03\",\"DHT11\":{\"Temperature\":24,\"Humidity\":10,\"DewPoint\":15},\"TempUnit\":\"C\"}" 

   Dzięki temu wygenerujemy dane emulowane jak by z czujnika DHT11 i już powinno nam się to narysowac na wykresie.
