# Segurança em IoT

# Relatório Técnico: Análise de Vulnerabilidades em Servidor Web ESP32  
**Objeto de Análise:** Código fonte e implementação de "ESP32 Web Server" (Referência: Random Nerd Tutorials).  
**Data:** 10 de Dezembro de 2025.  
**Contexto:** Internet das Coisas (IoT) e Sistemas Embarcados.


## 1. Análise Estática do Código (Vulnerabilidades)

Ao examinar o código fonte fornecido no tutorial, foram identificadas vulnerabilidades críticas de segurança que violam os princípios da confidencialidade, integridade e disponibilidade.

### A. Ausência de Autenticação e Autorização (CWE-306)
O código não implementa nenhum mecanismo de verificação de identidade. Qualquer dispositivo conectado à mesma rede Wi-Fi que conheça (ou descubra) o endereço IP do ESP32 pode enviar comandos GET para alterar o estado dos GPIOs. Não há login, senha ou token de API.

### B. Comunicação em Texto Claro / Ausência de Criptografia (CWE-319)
O servidor opera sobre HTTP padrão (porta 80) e não HTTPS.  
**Ponto Fraco:** Todo o tráfego de rede, incluindo os cabeçalhos e o conteúdo da página HTML, trafega em texto plano.  
**Consequência:** Um atacante na mesma rede pode realizar Sniffing (captura de pacotes) para ver exatamente quais comandos estão sendo enviados e qual o estado atual do dispositivo.

### C. Credenciais "Hardcoded" (CWE-798)
As credenciais de Wi-Fi (ssid e password) estão escritas diretamente no código-fonte.  
**Ponto Fraco:** Se este código for compartilhado (ex: GitHub) ou se o binário for extraído do chip (dump de memória), a rede Wi-Fi local fica comprometida.

### D. Tratamento Ineficiente de Entrada (Input Handling)
O código utiliza a classe String do Arduino para concatenar a requisição HTTP caractere por caractere (`header += c;`).  
**Ponto Fraco:** Não há limite definido para o tamanho do cabeçalho armazenado antes do processamento. Um atacante pode enviar uma requisição infinitamente longa sem caracteres de terminação, levando ao estouro da memória RAM (Heap Overflow/Exhaustion), travando o dispositivo.


## 2. Cenários de Ataque e Passo-a-Passo

Abaixo estão descritos dois ataques viáveis explorando as vulnerabilidades acima.

### **Ataque 1: Controle Não Autorizado (Unauthorized Access)**  
Este ataque explora a falta de autenticação para manipular o mundo físico (acender/apagar luzes ou acionar relés).  
**Vulnerabilidade explorada:** Falta de autenticação.

**Passo-a-passo:**
- **Reconhecimento:** O atacante conecta-se à mesma rede Wi-Fi.  
- **Varredura:** Utiliza uma ferramenta como *nmap* ou Angry IP Scanner para encontrar o IP do ESP32 (identificando a porta 80 aberta).  
- **Execução:** O atacante digita no navegador ou usa um script (cURL/Python) para enviar:  
  `http://<IP-DO-ESP32>/26/on`
- **Resultado:** O GPIO 26 é ativado sem o consentimento do proprietário.


### **Ataque 2: Negação de Serviço (DoS – Resource Exhaustion)**  
Este ataque visa tornar o ESP32 inoperante, impedindo que o usuário legítimo o acesse.  
**Vulnerabilidade explorada:** Gestão de memória precária e processamento síncrono (single thread).

**Passo-a-passo:**
- **Conexão:** O atacante identifica o IP do alvo.  
- **Execução:** O atacante inicia múltiplas conexões simultâneas ou envia requisições HTTP parciais muito lentas (Slowloris) ou extremamente longas.  
- **Saturação:** Como o ESP32 tem recursos limitados e o código trata um cliente por vez (`while (client.connected())`), o dispositivo fica preso processando o atacante ou trava por falta de memória.  
- **Resultado:** O servidor web para de responder ao usuário legítimo.


## 3. Análise de Risco (Probabilidade, Impacto e Risco)

Usando a matriz:  
**Risco = Probabilidade × Impacto**

### **Ataque 1 – Controle Não Autorizado**
- **Probabilidade:** Muito alta  
  *Não exige ferramentas; um navegador basta.*
- **Impacto:** Médio/Alto  
  *Varia conforme o dispositivo acionado; pode comprometer segurança física.*
- **Risco resultante:** **CRÍTICO**

### **Ataque 2 – Negação de Serviço (DoS)**
- **Probabilidade:** Média  
  *Requer conhecimento mínimo e intenção deliberada.*
- **Impacto:** Médio  
  *O dispositivo trava e requer reinicialização manual.*
- **Risco resultante:** **ALTO**


## 4. Tabela Consolidada de Riscos (Ordem Decrescente)

| Título do Ataque                   | Probabilidade | Impacto                  | Risco |
|-----------------------------------|--------------|--------------------------|--------|
| Controle Não Autorizado de GPIO  | Muito Alta   | Alto (Físico)            | CRÍTICO |
| Negação de Serviço (DoS)         | Média        | Médio (Disponibilidade)  | ALTO |
| Sniffing de Rede                 | Alta         | Baixo (Privacidade)      | MÉDIO |
