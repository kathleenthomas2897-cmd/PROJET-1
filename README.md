# PROJET-1
WEATHERSTATION PRO 
WeatherStation Pro simule une station meteo industrielle complete. Elle mesure simultanement la
temperature (deux zones), l'humidite, la pression atmospherique et la qualite de l'air. Les donnees
s'affichent sur un ecran OLED, une LED RGB indique l'etat global, un buzzer declenche des alertes
sonores, et un servomoteur simule une ventilation automatique. Un bouton permet de naviguer entre
les differents ecrans de donnees.
import machine
import time
import dht
# Importez les bibliothèques d'affichage et BMP si présentes dans Wokwi
# (ssd1306 et bmp280 sont généralement supportés ou simulés)

# --- CONFIGURATION DES BROCHES (Strictement alignée sur votre JSON) ---

# Capteurs Température & Humidité (DHT22)
dht1 = dht.DHT22(machine.Pin(15)) # Zone 1
dht2 = dht.DHT22(machine.Pin(14)) # Zone 2

# Bus I2C distincts
i2c_oled = machine.I2C(0, sda=machine.Pin(4), scl=machine.Pin(5)) # OLED
i2c_bmp = machine.I2C(1, sda=machine.Pin(2), scl=machine.Pin(3))   # BMP280

# Entrées : Contrôles (Boutons et Potentiomètre MQ-135)
btn_navigation = machine.Pin(18, machine.Pin.IN, machine.Pin.PULL_UP) # btn1
btn_reset_alarme = machine.Pin(19, machine.Pin.IN, machine.Pin.PULL_UP) # btn2
pot_mq135 = machine.ADC(26) # Potentiomètre de simulation CO2

# Sorties : Indicateurs et Actionneurs
led_r = machine.PWM(machine.Pin(10))
led_g = machine.PWM(machine.Pin(11))
led_b = machine.PWM(machine.Pin(12))
servo = machine.PWM(machine.Pin(16))
buzzer = machine.Pin(17, machine.Pin.OUT)

# Configurations des fréquences PWM
led_r.freq(1000)
led_g.freq(1000)
led_b.freq(1000)
servo.freq(50)

# --- VARIABLES GLOBALES ---
current_screen = 1
total_screens = 4
alarm_active = False

# Variables de stockage des mesures
temp1, hum1 = 0.0, 0.0
temp2, hum2 = 0.0, 0.0
pression, temp_bmp = 1013.25, 25.0 # Valeurs par défaut si I2C simulé par print
co2_level = 400
min_temp, max_temp = 99.0, -99.0

# --- FONCTIONS DU SYSTÈME ---
def set_servo_angle(angle):
    duty = int(1638 + (angle / 180) * 6554)
    servo.duty_u16(duty)

def set_rgb(r, g, b):
    led_r.duty_u16(r)
    led_g.duty_u16(g)
    led_b.duty_u16(b)

def read_sensors():
    global temp1, hum1, temp2, hum2, co2_level, min_temp, max_temp
    try:
        # Lecture des vrais composants DHT22 de Wokwi
        dht1.measure()
        temp1, hum1 = dht1.temperature(), dht1.humidity()
        
        dht2.measure()
        temp2, hum2 = dht2.temperature(), dht2.humidity()
        
        # Historique Min/Max basé sur la Zone 1
        if temp1 < min_temp: min_temp = temp1
        if temp1 > max_temp: max_temp = temp1
    except Exception as e:
        print("Erreur lecture DHT:", e)
        
    # Lecture du niveau CO2 via le potentiomètre (0 à 2000 ppm)
    co2_level = int(400 + (pot_mq135.read_u16() / 65535) * 1600)

def update_display():
    print(f"\n[ÉCRAN OLED - PAGE {current_screen}/4]")
    if current_screen == 1:
        print(f"Z1: {temp1:.1f}°C | {hum1:.1f}%")
        print(f"Z2: {temp2:.1f}°C | {hum2:.1f}%")
    elif current_screen == 2:
        print(f"Pres: {pression} hPa")
        print(f"Temp BMP: {temp_bmp}°C")
    elif current_screen == 3:
        print(f"Qualite Air: {co2_level} ppm")
        print("Statut: " + ("DANGER" if co2_level > 1000 or temp1 > 35 else "CORRECT"))
    elif current_screen == 4:
        print("Historique Zone 1")
        print(f"Min: {min_temp if min_temp != 99.0 else 0:.1f}°C")
        print(f"Max: {max_temp if max_temp != -99.0 else 0:.1f}°C")
    print("-" * 25)

def check_alerts():
    global alarm_active
    # Seuil critique : Température > 35°C (Zone 1) ou CO2 > 1000 ppm
    if temp1 > 35 or co2_level > 1000:
        alarm_active = True
        
    if alarm_active:
        set_rgb(65535, 0, 0) # Rouge constant
        buzzer.value(1)      # Alerte sonore active
        set_servo_angle(90)  # Volet de ventilation ouvert
    else:
        set_rgb(0, 65535, 0) # Vert constant (Normal)
        buzzer.value(0)      # Alerte coupée
        set_servo_angle(0)   # Volet fermé

# --- BOUCLE D'ÉCOUTE ET D'ACTION ---
print("Système initialisé avec votre configuration Wokwi...")
set_servo_angle(0)

while True:
    # Navigation : Bouton 1 (GP18)
    if btn_navigation.value() == 0:
        current_screen = 1 if current_screen >= total_screens else current_screen + 1
        time.sleep(0.2) # Anti-rebond
        
    # Reset Alarme : Bouton 2 (GP19)
    if btn_reset_alarme.value() == 0:
        if alarm_active:
            alarm_active = False
            print(">> Alarme réinitialisée.")
        time.sleep(0.2)

    # Rafraîchissement des données
    read_sensors()
    check_alerts()
    update_display()
    
    time.sleep(0.4)



    {
  "version": 1,
  "author": "WeatherStation Pro (Validé Examen)",
  "editor": "wokwi",
  "parts": [
    { "type": "wokwi-pi-pico", "id": "pico", "top": 40, "left": 0, "attrs": {} },
    { "type": "wokwi-dht22", "id": "dht1", "top": -120, "left": -220, "attrs": {} },
    { "type": "wokwi-dht22", "id": "dht2", "top": -120, "left": -100, "attrs": {} },
    { "type": "wokwi-bmp280", "id": "bmp", "top": -120, "left": 180, "attrs": {} },
    { "type": "wokwi-ssd1306", "id": "oled", "top": 280, "left": 0, "attrs": {} },
    { 
      "type": "wokwi-rgb-led", 
      "id": "rgb", 
      "top": -220, 
      "left": 0, 
      "attrs": { "common": "cathode" } 
    },
    { "type": "wokwi-servo", "id": "servo", "top": 120, "left": 240, "attrs": {} },
    { "type": "wokwi-buzzer", "id": "buzzer", "top": 120, "left": -240, "attrs": {} },
    { "type": "wokwi-pushbutton", "id": "btn1", "top": 280, "left": -180, "attrs": { "label": "Nav", "color": "blue" } },
    { "type": "wokwi-pushbutton", "id": "btn2", "top": 280, "left": 180, "attrs": { "label": "Reset", "color": "red" } },
    { "type": "wokwi-potentiometer", "id": "mq135_pot", "top": -260, "left": 180, "attrs": { "label": "Simul MQ135" } }
  ],
  "connections": [
    ["pico:GP15", "dht1:DATA", "green"],
    ["pico:3V3", "dht1:VCC", "red"],
    ["pico:GND", "dht1:GND", "black"],

    ["pico:GP14", "dht2:DATA", "green"],
    ["pico:3V3", "dht2:VCC", "red"],
    ["pico:GND", "dht2:GND", "black"],

    ["pico:GP4", "oled:SDA", "blue"],
    ["pico:GP5", "oled:SCL", "yellow"],
    ["pico:3V3", "oled:VCC", "red"],
    ["pico:GND", "oled:GND", "black"],

    ["pico:GP2", "bmp:SDA", "blue"],
    ["pico:GP3", "bmp:SCL", "yellow"],
    ["pico:3V3", "bmp:VCC", "red"],
    ["pico:GND", "bmp:GND", "black"],

    ["pico:GP10", "rgb:R", "red"],
    ["pico:GP11", "rgb:G", "green"],
    ["pico:GP12", "rgb:B", "blue"],
    ["pico:GND", "rgb:COM", "black"],

    ["pico:GP16", "servo:PWM", "orange"],
    ["pico:VBUS", "servo:V+", "red"],
    ["pico:GND", "servo:GND", "black"],

    ["pico:GP17", "buzzer:1", "purple"],
    ["pico:GND", "buzzer:2", "black"],

    ["pico:GP18", "btn1:1", "white"],
    
    ["pico:GND", "btn1:2", "black"],

    ["pico:GP19", "btn2:1", "white"],
    ["pico:GND", "btn2:2", "black"],

    ["pico:GP26", "mq135_pot:SIG", "green"],
    ["pico:3V3", "mq135_pot:VCC", "red"],
    ["pico:GND", "mq135_pot:GND", "black"]
  ]
}
