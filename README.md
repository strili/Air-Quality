# Medidor de qualidade do ar
Este trabalho aborda o desenvolvimento de um medidor de qualidade do ar utilizando RASPBERRY PI3 monitorado via Ubidots.
Autores: 
- Gustavo Strilicherk Pinto RA: 20.01071-0
- Henrique Baraldi Cogo RA: 21.01811-0
- Rafael Callegaris Dias RA: 21.00531-0
- Lucas Silva Anholeto RA: 21.02145-7

# Introdução
Este projeto consiste em um medidor de qualidade do ar utilizando uma Raspberry Pi e sensores especializados (MQ2, MQ7 e BME280), capazes de monitorar gases e condições atmosféricas. Os dados coletados são enviados para a plataforma Ubidots, permitindo o monitoramento em tempo real da qualidade do ar.

# Componentes Eletrônicos e Esquema Elétrico
Os sensores MQ2 e MQ7 possuem saídas analógicas, que não são diretamente legíveis pela Raspberry Pi, pois ela não possui entradas analógicas. Por isso, utilizamos o conversor analógico-digital ADS1115, que converte os sinais dos sensores em dados digitais. 

No ADS1115, cada sensor é conectado a um canal de entrada analógica (por exemplo, o MQ2 no A0 e o MQ7 no A1). Em seguida, o ADS1115 se comunica com a Raspberry Pi através da interface I2C, utilizando os pinos SDA e SCL. Essa conexão permite que a Raspberry Pi leia os valores de tensão dos sensores e processe os dados.
![Captura de tela 2024-11-07 145058](https://github.com/user-attachments/assets/05616e0d-e1a0-4fb6-9af3-84edfcbf1844)

O sensor BME280 possui uma interface digital I2C, que permite sua conexão direta com a Raspberry Pi sem necessidade de conversor adicional. 

Na Raspberry Pi, conectamos o BME280 aos pinos I2C: o pino SDA do BME280 é ligado ao pino SDA da Raspberry Pi, e o pino SCL do BME280 é ligado ao pino SCL da Raspberry Pi. A alimentação do BME280 é fornecida conectando o pino VCC a 3.3V da Raspberry Pi e o GND a um pino de aterramento (GND). Assim, a Raspberry Pi consegue ler os valores de temperatura, umidade e pressão diretamente do BME280.

![BME280OO](https://github.com/user-attachments/assets/8aa5f70d-3f19-4e62-943c-530b8186a9f8)


A tabela abaixo indica os componentes utilizados e sua função no projeto
![tabela lista](https://github.com/user-attachments/assets/9953795f-f40f-477b-a8d6-a452eaaaeb87)

# Funcionamento do sistema

Este sistema de monitoramento de qualidade do ar usa uma Raspberry Pi conectada a três sensores: o MQ2, que detecta fumaça e gases inflamáveis; o MQ7, que mede monóxido de carbono (CO); e o BME280, que monitora temperatura, umidade e pressão atmosférica. A configuração funciona da seguinte forma:

1. **Leitura de Sensores**: Os sensores MQ2 e MQ7 produzem saídas analógicas. Como a Raspberry Pi não possui entradas analógicas, o ADS1115 é utilizado como um conversor analógico-digital (ADC), convertendo as leituras dos sensores para sinais digitais que a Raspberry Pi pode processar. Já o BME280 possui comunicação digital I2C, que se conecta diretamente à Raspberry Pi para fornecer medições de temperatura, umidade e pressão.

2. **Processamento de Dados**: A Raspberry Pi coleta os dados de todos os sensores, realizando leituras periódicas e transformando os sinais em valores que representam a qualidade do ar e as condições ambientais.

3. **Envio para o Ubidots**: A Raspberry Pi envia os dados coletados para a plataforma Ubidots via uma conexão HTTP, usando um token de autenticação. O Ubidots exibe as informações em tempo real em gráficos e dashboards, permitindo o monitoramento remoto dos níveis de gases e das condições atmosféricas.

Assim, o sistema fornece uma visão contínua da qualidade do ar, possibilitando alertas e acompanhamento a distância.


# Software
    import time
    import busio
    import board
    import adafruit_ads1x15.ads1115 as ADS
    from adafruit_ads1x15.analog_in import AnalogIn
    from adafruit_bme280 import basic as adafruit_bme280
    import requests

    # Configuração do ADS1115 com o barramento I2C
    i2c = busio.I2C(board.SCL, board.SDA)
    adc = ADS.ADS1115(i2c)

    # Configuração dos canais para os sensores MQ2 e MQ7
    mq2_channel = AnalogIn(adc, ADS.P0)
    mq7_channel = AnalogIn(adc, ADS.P1)

    # Configuração do BME280
    bme280 = adafruit_bme280.Adafruit_BME280_I2C(i2c)  # Tente esta classe novamente, pois é a forma mais recente

    # Configuração do Ubidots
    url = "https://industrial.api.ubidots.com/api/v1.6/devices/terminal_output"  # Substitua pelo nome do seu dispositivo
    token = "SEU_TOKEN_UBIDOTS_AQUI"  # Substitua pelo seu token Ubidots

    # Função para enviar os dados ao Ubidots usando HTTP
    def send_to_ubidots(temperature, humidity, pressure, mq2_value, mq7_value):
    headers = {
        "X-Auth-Token": token,
        "Content-Type": "application/json"
    }
    payload = {
        "temperature": temperature,
        "humidity": humidity,
        "pressure": pressure,
        "mq2": mq2_value,
        "mq7": mq7_value
    }
    
    # Envia a requisição POST
    response = requests.post(url, json=payload, headers=headers)
    
    # Verifica se o envio foi bem-sucedido
    if response.status_code == 201:
        print("Dados enviados ao Ubidots com sucesso.")
    else:
        print(f"Falha ao enviar dados: {response.status_code} - {response.text}")

    # Loop principal
    if __name__ == "__main__":
    while True:
        # Leitura dos sensores BME280
        temperature = bme280.temperature  # Temperatura em °C
        humidity = bme280.humidity        # Umidade relativa em %
        pressure = bme280.pressure        # Pressão atmosférica em hPa
        
        # Leitura dos sensores MQ2 e MQ7
        mq2_value = mq2_channel.voltage
        mq7_value = mq7_channel.voltage
        
        print(f"Temperatura: {temperature:.2f}°C, Umidade: {humidity:.2f}%, Pressão: {pressure:.2f}hPa")
        print(f"MQ2: {mq2_value:.2f} V, MQ7: {mq7_value:.2f} V")
        
        # Enviar dados ao Ubidots via HTTP
        send_to_ubidots(temperature, humidity, pressure, mq2_value, mq7_value)
        
        # Intervalo de leitura em segundos
        time.sleep(10)

# Imagens do Ubidots

Os dados coletados pelos sensores são enviados da Raspberry Pi para o Ubidots por meio de uma requisição HTTP POST. A Raspberry Pi usa um token de autenticação e envia as leituras de temperatura, umidade, pressão, e níveis de gases em um formato JSON para o endpoint do Ubidots.

No Ubidots, esses dados são recebidos e armazenados em um dispositivo virtual configurado para o projeto. A plataforma exibe as informações em dashboards personalizados, permitindo visualizar os valores em tempo real, gráficos históricos e configurar alertas, oferecendo um monitoramento completo e acessível remotamente.
![image](https://github.com/user-attachments/assets/7da396a4-cd34-4ed4-ae94-c7972c7aeea0)

# Fotos do projeto

![WhatsApp Image 2024-11-07 at 3 10 23 PM (1)](https://github.com/user-attachments/assets/ac40f901-130a-466f-a57a-d834e62c915e)
![WhatsApp Image 2024-11-07 at 3 10 23 PM](https://github.com/user-attachments/assets/e059ee25-b201-4e81-bd20-3dba944980f9)
![WhatsApp Image 2024-11-07 at 3 15 49 PM](https://github.com/user-attachments/assets/334228e8-38d7-46b8-8508-e946cad3b55d)

# Integrantes do grupo

![WhatsApp Image 2024-11-07 at 3 10 25 PM](https://github.com/user-attachments/assets/6bbcdc1d-dff9-4903-9cca-f754f7b8641b)

# Vídeo do projeto

https://github.com/user-attachments/assets/babbb60f-eb55-4cce-891b-a802dce47999







