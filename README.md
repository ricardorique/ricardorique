import pandas as pd
import paho.mqtt.client as mqtt
import time

# Configuração do broker
broker = 'localhost'
port = 1883
topico = "Dados_de_EEG"

# Carregar os dados CSV
dados = pd.read_csv('C:\\Users\\USER\\Desktop\\tiny_eeg_self_experiment_music.csv')

# Função que é chamada quando a conexão ao broker é estabelecida
def on_connect(client, userdata, flags, rc):
    if rc == 0:
        print("Conectado ao broker MQTT com sucesso!")
    else:
        print(f"Falha na conexão. Código de erro: {rc}")

# Função que é chamada quando uma mensagem é publicada
def on_publish(client, userdata, mid):
    print("Dado publicado com sucesso!")

# Função para enviar dados das colunas 1 a 4 (índices 1 a 4)
def enviar_dados_csv(client, topico):
    # Itera sobre as colunas 1 a 4 (índices 1 a 4)
    for coluna in dados.columns[1:5]:  # Certifique-se de incluir a coluna 4 também
        topico_canal = f"{topico}/Canal{coluna}"  # Ajuste o nome do tópico para incluir o número do canal
        
        # Itera sobre os valores da coluna
        for i, valor in enumerate(dados[coluna]):
            try:
                # Envia o valor numérico para o tópico MQTT, sem incluir o nome do canal na mensagem
                client.publish(topico_canal, valor)  # Envia apenas o valor (numérico) da coluna
                time.sleep(0,0)  # Aguarda 0,5 segundo antes de enviar o próximo dado
                print(f"Valor {valor} enviado para {topico_canal}. {i+1}/{len(dados[coluna])} valores enviados.")
            except Exception as e:
                print(f"Erro ao enviar o valor {valor} para o tópico {topico_canal}: {e}")
        
        # Após enviar todos os dados de uma coluna, imprime a mensagem de conclusão
        print(f"Coluna {coluna} processada.")


# Configuração do client MQTT
client = mqtt.Client()  
client.on_connect = on_connect  # Atribuindo a função de conexão
client.on_publish = on_publish  # Atribuindo a função de publicação

# Conectar ao broker
client.connect(broker, port, 60)

# Iniciar o loop do cliente MQTT
client.loop_start()  # Inicia a execução assíncrona

# Enviar os dados do CSV
enviar_dados_csv(client, topico)

# Aguarda a conclusão do envio
print("Todos os dados foram enviados.")
time.sleep(10)  # Aguarda alguns segundos para garantir o envio antes de finalizar
client.loop_stop()  # Para o loop MQTT
import matplotlib.pyplot as plt
import matplotlib.animation as animation
import paho.mqtt.client as mqtt
import numpy as np
import time

# Variáveis globais para armazenar os dados recebidos
x_data = []  # Eixo X (tempo ou índices)
y_data_dict = {1: [], 2: [], 3: [], 4: []}  # Eixo Y para cada canal (colunas 2, 3, 4, 5)

# Função que é chamada quando a conexão ao broker é estabelecida
def on_connect(client, userdata, flags, rc):
    if rc == 0:
        print("Conectado ao broker MQTT com sucesso!")
    else:
        print(f"Falha na conexão. Código de erro: {rc}")
    # Subscribing ao tópico
    client.subscribe("Dados_de_EEG/#")  # Certifique-se de que o tópico está correto

def on_message(client, userdata, msg):
    try:
        # Imprime o conteúdo da mensagem recebida para diagnóstico
        print(f"Mensagem recebida no tópico {msg.topic}: {msg.payload.decode('utf-8')}")

        # Extrai o nome do canal a partir do tópico
        canal = msg.topic.split('/')[-1]  # Divida o tópico e obtenha a última parte
        
        # Extração correta do número do canal
        if canal.startswith('Channel'):
            canal_num = int(canal.replace('Channel', ''))  # Converte o número do canal para int
        else:
            canal_num = int(''.join(filter(str.isdigit, canal)))  # Extrai o número, ignorando caracteres extras

        # Converte a carga útil para float
        value = float(msg.payload.decode("utf-8"))  # Tenta converter a carga para float

        # Adiciona o valor no eixo Y e o índice no eixo X
        x_data.append(len(x_data))  # Índice de tempo no eixo X
        y_data_dict[canal_num].append(value)  # Adiciona o valor ao canal correspondente

        # Limitar o tamanho dos arrays para cada canal (para gráficos dinâmicos)
        if len(y_data_dict[canal_num]) > 1000:  # Mantém o gráfico dinâmico
            y_data_dict[canal_num].pop(0)

    except Exception as e:
        print(f"Erro ao processar a mensagem: {e}")


# Configuração do client MQTT
client = mqtt.Client()
client.on_connect = on_connect  # Atribuindo a função de conexão
client.on_message = on_message  # Atribuindo a função de processamento da mensagem

# Conectar ao broker
broker = "localhost"
port = 1883
client.connect(broker, port, 60)

# Iniciar o loop do cliente MQTT para manter a conexão
client.loop_start()

# Criando o gráfico dinâmico com 2 linhas e 4 colunas para os gráficos
fig, axes = plt.subplots(2, 4, figsize=(16, 10))  # Agora temos 2 linhas e 4 colunas

# Criando gráficos para cada canal EEG (subgráficos superiores)
line_dict = {}
for i, col in enumerate([1, 2, 3, 4]):
    ax = axes[0, i]  # Primeira linha para gráficos de EEG
    ax.set_ylim(0, 1000)  # Ajuste do limite do gráfico conforme necessário
    ax.set_xlim(0, 1)  # Ajuste o limite do gráfico conforme necessário
    ax.set_xlabel('Tempo')
    ax.set_ylabel('Amplitudes (EEG)')
    ax.set_title(f'Canal {col} (EEG)')
    line_dict[col], = ax.plot([], [], lw=2)

# Criando gráficos para cada canal FFT (subgráficos inferiores)
fft_line_dict = {}
for i, col in enumerate([1, 2, 3, 4]):
    ax = axes[1, i]  # Segunda linha para gráficos de FFT
    ax.set_xlim(0, 10)  # Limite de frequência (0 a 50 Hz)
    ax.set_ylim(0, 100)  # Limite para a magnitude
    ax.set_xlabel('Frequência (Hz)')
    ax.set_ylabel('Magnitude (FFT)')
    ax.set_title(f'Canal {col} (FFT)')
    fft_line_dict[col], = ax.plot([], [], lw=2, color='r')

# Função para calcular a FFT e retornar a magnitude
def calcular_fft(dados):
    if len(dados) < 10:  # Verifique se há dados suficientes
        return np.array([]), np.array([])  # Retorna arrays vazios para evitar o erro
    N = len(dados)
    fft_result = np.fft.fft(dados)
    fft_freq = np.fft.fftfreq(N, 1)  # Frequência de amostragem = 1 (tempo unitário)
    fft_magnitude = np.abs(fft_result)  # Magnitude da FFT
    return fft_freq, fft_magnitude

# Função para verificar se há uma frequência entre 8Hz e 12Hz
def verificar_frequencia_fft(fft_freq, fft_magnitude):
    faixa_alfa_min = 8
    faixa_alfa_max = 12

    # Garantir que fft_freq seja um numpy array
    fft_freq = np.array(fft_freq)

    indices_faixa_alfa = np.where((fft_freq >= faixa_alfa_min) & (fft_freq <= faixa_alfa_max))[0]

    if len(indices_faixa_alfa) > 0 and np.any(fft_magnitude[indices_faixa_alfa] > 0):
        return True
    return False

# Função para atualizar os gráficos (incluindo a FFT)
def update_graph(frame):
    for col in y_data_dict.keys():
        # Verifica se os dados X e Y têm o mesmo tamanho
        if len(x_data) != len(y_data_dict[col]):
            print(f"Erro: Tamanhos de x_data e y_data_dict[{col}] não coincidem.")
            return tuple(line_dict[col] for col in y_data_dict.keys()) + tuple(fft_line_dict[col] for col in y_data_dict.keys())

        # Atualizar gráfico do canal original (EEG)
        line_dict[col].set_data(x_data, y_data_dict[col])

        # Calcular e atualizar o gráfico da FFT
        fft_freq, fft_magnitude = calcular_fft(y_data_dict[col])
        if len(fft_freq) > 0 and len(fft_magnitude) > 0:
            fft_line_dict[col].set_data(fft_freq[:len(fft_freq)//2], fft_magnitude[:len(fft_magnitude)//2])

        if verificar_frequencia_fft(fft_freq, fft_magnitude):
            print(f"Beep detectado no Canal {col} na faixa alfa (8-12 Hz)")

    return tuple(line_dict[col] for col in y_data_dict.keys()) + tuple(fft_line_dict[col] for col in y_data_dict.keys())

# Função para verificar se o fechamento deve ocorrer
def on_close(event):
    print("Fechando animação...")
    plt.close(fig)
    client.loop_stop()


# Animação do gráfico
ani = animation.FuncAnimation(fig, update_graph, frames=100, blit=True, interval=20)  # 50 vezes por segundo (intervalo de 20ms)


# Conectar o evento de fechamento à função
fig.canvas.mpl_connect('close_event', on_close)

plt.tight_layout()
plt.show()

# Manter o loop MQTT ativo enquanto o gráfico é exibido
try:
    while True:
        time.sleep(60)
except KeyboardInterrupt:
    print("Programa finalizado.")
    client.loop_stop()
<!--
**ricardorique/ricardorique** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:

- 🔭 I’m currently working on ...
- 🌱 I’m currently learning ...
- 👯 I’m looking to collaborate on ...
- 🤔 I’m looking for help with ...
- 💬 Ask me about ...
- 📫 How to reach me: ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...
-->
