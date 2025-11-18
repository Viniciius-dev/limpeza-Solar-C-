# limpeza-Solar-C-
// --- BIBLIOTECAS ---
#include <WiFi.h>
#include <Wire.h>
#include <RTClib.h>
#include <ESP32Servo.h>
#include <time.h>
#include <Firebase_ESP_Client.h>

// --- Wi-Fi (com Fallback) ---
const char* wifi_ssid_1 = "*******";       // Rede Principal (Opção 1)
const char* wifi_pass_1 = "*******";
const char* wifi_ssid_2 = "*******"; // Rede Secundária (Opção 2)
const char* wifi_pass_2 = "*******";

// --- Firebase Config ---
#define API_KEY "*******"
#define PROJECT_ID "limpeza-solar"
#define USER_EMAIL "*******"
#define USER_PASSWORD "*******"

// --- Firestore ---
String firestoreDocumentPath = "sistemas_limpeza/sistema_001";

// --- Pinos ---
const int pinServo = 13;
const int pinBotao = 25;
const int pinLDR = 34;         // Pino do LDR (Sensor de Sujeira)
const int pinLedSensor = 33;   // Pino para ligar o LED do sensor de sujeira
const int pinSensorChuva = 35; // Sensor de Chuva
const int pinMotor1 = 26; // Motor Esteira (PIN 1 -> L9110S A-IA)
const int pinMotor2 = 27; // Motor Esteira (PIN 2 -> L9110S A-IB)
const int pinMotor3 = 14; // Motor Rolo (PIN 1 -> L9110S B-IA)
const int pinMotor4 = 12; // Motor Rolo (PIN 2 -> L9110S B-IB)

// --- Firebase Globais ---
FirebaseData fbdo;
FirebaseAuth auth;
FirebaseConfig config;

// --- Hardware Globais ---
DS3231 rtc;
Servo escova;

// --- Variáveis de Controle ---
String ultimoStatusEnviado = "";
String comandoPendente = "";
unsigned long ultimoLoopFirebase = 0;
const long intervaloFirebase = 10000;

bool limpouManha = false;
bool limpouNoite = false;

// Horários padrão (serão substituídos pelo Firebase)
int configHoraManha = 7;
int configMinManha = 0;
int configHoraNoite = 19;
int configMinNoite = 1;

// Máquina de Estados da Limpeza
bool estaLimpando = false;
int etapaLimpeza = 0;
unsigned long tempoEtapaAnterior = 0;
const long INTERVALO_MOVIMENTO = 1000; // Tempo por etapa

unsigned long ultimoCheckConexao = 0;

// Debounce do Botão
int estadoBotaoAnterior = HIGH;
unsigned long ultimoTempoBotao = 0;
const long delayDebounce = 50;

// Limite do sensor de chuva (Calibre este valor!)
const int CHUVA_THRESHOLD = 2000;


void setup() {
  Serial.begin(9600);
  Serial.println("\n\nINICIANDO SISTEMA DE LIMPEZA SOLAR (Versão Completa L9110S)...");

  Wire.begin(21, 22);

  if (!rtc.begin()) {
    Serial.println("ERRO: Não foi possível encontrar o RTC.");
  }
  Serial.println("RTC inicializado.");

  escova.attach(pinServo);
  escova.write(90);
  Serial.println("Servo inicializado.");

  pinMode(pinBotao, INPUT_PULLUP);
  Serial.println("Botão inicializado.");

  pinMode(pinSensorChuva, INPUT);
  Serial.println("Sensor de chuva inicializado.");

  // LIGA O LED DO SENSOR DE SUJEIRA
  pinMode(pinLedSensor, OUTPUT);
  digitalWrite(pinLedSensor, HIGH); // Mantém o LED sempre aceso
  Serial.println("LED do sensor de sujeira LIGADO.");

  Serial.println("Inicializando motores (Esteira e Rolo)...");
  pinMode(pinMotor1, OUTPUT);
  pinMode(pinMotor2, OUTPUT);
  pinMode(pinMotor3, OUTPUT);
  pinMode(pinMotor4, OUTPUT);
  pararTodosMotores(); // Garante que comecem desligados

  conectarWiFi(); // Tenta conectar nas redes Wi-Fi

  Serial.println("Verificando hora do RTC...");
  DateTime agoraRTC = rtc.now();

  // Se o ano for < 2020, o RTC provavelmente perdeu energia
  if (agoraRTC.year() < 2020) {
    Serial.println("RTC com hora inválida. Sincronizando com NTP...");
    configTime(-3 * 3600, 0, "pool.ntp.org", "time.nist.gov"); // Fuso GMT-3
    struct tm timeinfo;
    if (!getLocalTime(&timeinfo)) {
      Serial.println("ERRO: Falha ao obter hora do NTP.");
    } else {
      Serial.println("✅ Relógio (NTP) Sincronizado!");
      rtc.adjust(DateTime(timeinfo.tm_year + 1900, timeinfo.tm_mon + 1, timeinfo.tm_mday,
                          timeinfo.tm_hour, timeinfo.tm_min, timeinfo.tm_sec));
      Serial.println("✅ RTC ajustado com sucesso!");
    }
  } else {
    Serial.printf("✅ RTC OK. Hora atual: %02d:%02d:%02d\n", agoraRTC.hour(), agoraRTC.minute(), agoraRTC.second());
  }

  config.api_key = API_KEY;
  auth.user.email = USER_EMAIL;
  auth.user.password = USER_PASSWORD;

  Firebase.begin(&config, &auth);
  Firebase.reconnectWiFi(true);
  Serial.println("✅ Firebase Inicializado. Aguardando autenticação...");

  Serial.print("Aguardando autenticação do Firebase...");
  unsigned long inicioEspera = millis();
  while (!Firebase.ready() && (millis() - inicioEspera < 10000)) {
    Serial.print(".");
    delay(500); // Delay aceitável, pois só ocorre no setup
  }

  if (Firebase.ready()) {
    Serial.println("\n✅ Firebase Autenticado!");
    buscarConfiguracoesFirebase(); // Busca horários no Firebase
  } else {
    Serial.println("\nERRO: Timeout na autenticação do Firebase. Usando horários padrão.");
    Serial.printf("Padrão: Manhã (%02d:%02d), Noite (%02d:%02d)\n",
                  configHoraManha, configMinManha, configHoraNoite, configMinNoite);
  }
}


void loop() {
  verificarConexoes();
  loopLimpeza();

  DateTime agora = rtc.now();
  
  processarBotao();

  // Lógica 1: Acionamento automático por horário
  if (agora.hour() == configHoraManha && agora.minute() == configMinManha && !limpouManha) {
    int sujeiraPercent = lerSujeira();
    if (sujeiraPercent > 50) {
      Serial.println(">> ACIONAMENTO: Automático (Manhã + Sujeira)");
      limpar();
      enviarStatusParaFirestore("Limpeza automática (Manhã)");
    } else {
      Serial.println(">> ACIONAMENTO: Horário (Manhã) verificado, painel limpo.");
      enviarStatusParaFirestore("Painel limpo (Manhã)");
    }
    limpouManha = true;

  } else if (agora.hour() == configHoraNoite && agora.minute() == configMinNoite && !limpouNoite) {
    int sujeiraPercent = lerSujeira();
    if (sujeiraPercent > 50) {
      Serial.println(">> ACIONAMENTO: Automático (Noite + Sujeira)");
      limpar();
      enviarStatusParaFirestore("Limpeza automática (Noite)");
    } else {
      Serial.println(">> ACIONAMENTO: Horário (Noite) verificado, painel limpo.");
      enviarStatusParaFirestore("Painel limpo (Noite)");
    }
    limpouNoite = true;
  }

  // Reseta as flags todo dia à meia-noite
  if (agora.hour() == 0 && agora.minute() == 0) {
    if (limpouManha || limpouNoite) {
      Serial.println("Meia-noite. Resetando flags de limpeza diária.");
      limpouManha = false;
      limpouNoite = false;
    }
  }

  // Lógica 2: Acionamento por comando remoto
  if (comandoPendente != "") {
    processarComandoPendente();
  }

  // Lógica 3: Comunicação com Firebase
  if (Firebase.ready() && (millis() - ultimoLoopFirebase > intervaloFirebase)) {
    ultimoLoopFirebase = millis();
    int sujeiraPercent = lerSujeira();
    Serial.printf("[Firebase] Hora: %02d:%02d:%02d | Sujeira: %d%% | Status: %s\n",
                  agora.hour(), agora.minute(), agora.second(), sujeiraPercent, ultimoStatusEnviado.c_str());
    enviarSujeiraParaFirestore(sujeiraPercent);
    if (comandoPendente == "" && !estaLimpando) {
      verificarComandosFirestore();
    }
  }
}

/**
 * @brief Processa o botão físico com debounce.
 */
void processarBotao() {
  int leituraBotao = digitalRead(pinBotao);

  if (leituraBotao == LOW && estadoBotaoAnterior == HIGH) {
    if (millis() - ultimoTempoBotao > delayDebounce) {
      Serial.println(">> ACIONAMENTO: Botão Físico");
      if (estaLimpando) {
        Serial.println("...Ciclo interrompido pelo botão.");
        estaLimpando = false;
        etapaLimpeza = 0;
        escova.write(90);
        pararTodosMotores();
        enviarStatusParaFirestore("Parado (Botão)");
      } else {
        limpar();
        enviarStatusParaFirestore("Limpeza manual");
      }
      ultimoTempoBotao = millis();
    }
  }
  estadoBotaoAnterior = leituraBotao;
}

/**
 * @brief Apenas INICIA o ciclo de limpeza, se não estiver chovendo.
 */
void limpar() {
  if (estaLimpando) return; // Já está limpando

  // *** LÓGICA DE CHUVA (PREVENÇÃO) ***
  if (estaChovendo()) {
    Serial.println(">> BLOQUEADO: Tentativa de limpeza, mas está chovendo.");
    enviarStatusParaFirestore("Parado (Chuva)");
    return; // NÃO INICIA A LIMPEZA
  }

  // Se não estiver chovendo, continua normally...
  Serial.println("...Iniciando ciclo de limpeza...");
  enviarStatusParaFirestore("Limpando");

  estaLimpando = true;
  etapaLimpeza = 1;
  tempoEtapaAnterior = millis();
  
  Serial.println("...Etapa 1: Servo 0, Esteira Frente, Rolo Ligado");
  escova.write(0);
  controlarEsteira(1);  // 1 = Frente
  controlarRolo(true);  // true = Ligado
}

/**
 * @brief Máquina de estados da limpeza (com controle de motor).
 */
void loopLimpeza() {
  // *** LÓGICA DE CHUVA (PARADA DE EMERGÊNCIA) ***
  if (estaLimpando && estaChovendo()) {
    Serial.println(">> PARADA DE EMERGÊNCIA: Chuva detectada durante a limpeza!");
    estaLimpando = false;
    etapaLimpeza = 0;
    escova.write(90);
    pararTodosMotores();
    enviarStatusParaFirestore("Parado (Chuva)");
    return; // Interrompe a limpeza
  }

  if (!estaLimpando) {
    pararTodosMotores();
    return;
  }

  if (millis() - tempoEtapaAnterior < INTERVALO_MOVIMENTO) {
    return; // Aguarda o tempo da etapa
  }

  tempoEtapaAnterior = millis(); // Reseta o cronômetro da etapa

  switch (etapaLimpeza) {
    case 1:
      Serial.println("...Etapa 2: Servo 180, Esteira Trás, Rolo Ligado");
      escova.write(180);
      controlarEsteira(-1); // -1 = Trás
      controlarRolo(true);
      etapaLimpeza = 2;
      break;

    case 2:
      Serial.println("...Etapa 3: Servo 0, Esteira Frente, Rolo Ligado");
      escova.write(0);
      controlarEsteira(1);  // 1 = Frente
      controlarRolo(true);
      etapaLimpeza = 3;
      break;

    case 3:
      Serial.println("...Etapa 4: Finalizando...");
      escova.write(180);
      controlarEsteira(-1); // -1 = Trás
      controlarRolo(true);
      etapaLimpeza = 4;
      break;

    case 4:
      Serial.println(">> LIMPEZA FINALIZADA");
      escova.write(90); // Posição neutra
      pararTodosMotores();
      enviarStatusParaFirestore("Parado");
      estaLimpando = false;
      etapaLimpeza = 0;
      break;
  }
}

/**
 * @brief Lê o LDR e mapeia o valor para 0-100% de sujeira.
 */
int lerSujeira() {
  // ATENÇÃO: Calibre estes valores!
  const int ldrLimpo = 3500; // Exemplo: 3500 (muita luz)
  const int ldrSujo = 500;   // Exemplo: 500 (pouca luz)

  int valorLDR = analogRead(pinLDR);

  // Mapeia o valor lido (invertido) para a porcentagem de sujeira
  int sujeiraPercent = map(valorLDR, ldrLimpo, ldrSujo, 0, 100);

  // Garante que o valor fique sempre entre 0 e 100
  sujeiraPercent = constrain(sujeiraPercent, 0, 100);
  
  return sujeiraPercent;
}

/**
 * @brief Envia o nível de sujeira para o Firestore.
 */
void enviarSujeiraParaFirestore(int sujeira) {
  FirebaseJson content;
  content.set("fields/nivel_sujeira/stringValue", String(sujeira) + "%");
  String mask = "nivel_sujeira";
  if (!Firebase.Firestore.patchDocument(&fbdo, PROJECT_ID, "(default)", firestoreDocumentPath.c_str(), content.raw(), mask.c_str())) {
    Serial.println("ERRO ao enviar sujeira: " + fbdo.errorReason());
  }
}

/**
 * @brief Envia o status atual do robô para o Firestore.
 */
void enviarStatusParaFirestore(String status) {
  if (status == ultimoStatusEnviado) return; // Otimização

  FirebaseJson content;
  content.set("fields/status_atual/stringValue", status);
  String mask = "status_atual";

  if (Firebase.Firestore.patchDocument(&fbdo, PROJECT_ID, "(default)", firestoreDocumentPath.c_str(), content.raw(), mask.c_str())) {
    Serial.println("Status enviado: " + status);
    ultimoStatusEnviado = status;
  } else {
    Serial.println("ERRO ao enviar status: " + fbdo.errorReason());
  }
}

/**
 * @brief Verifica se há um novo comando no Firestore.
 */
void verificarComandosFirestore() {
  if (Firebase.Firestore.getDocument(&fbdo, PROJECT_ID, "(default)", firestoreDocumentPath.c_str())) {
    if (fbdo.httpCode() == FIREBASE_ERROR_HTTP_CODE_OK) {
      FirebaseJson json(fbdo.payload().c_str());
      FirebaseJsonData result;
      json.get(result, "fields/comando_pendente/stringValue");
      if (result.success && result.stringValue != "" && result.stringValue != "nenhum") {
        comandoPendente = result.stringValue;
        Serial.println("Novo comando recebido do site: " + comandoPendente);
      }
    }
  } else {
    Serial.println("ERRO ao verificar comandos: " + fbdo.errorReason());
  }
}

/**
 * @brief Executa o comando que foi recebido do site.
 */
void processarComandoPendente() {
  if (comandoPendente == "iniciar_limpeza") {
    Serial.println(">> ACIONAMENTO: Remoto (Site)");
    limpar();

  } else if (comandoPendente == "parar_limpeza") {
    Serial.println(">> ACIONAMENTO: Parada Remota");
    if (estaLimpando) {
      estaLimpando = false;
      etapaLimpeza = 0;
      escova.write(90);
      pararTodosMotores();
      enviarStatusParaFirestore("Parado (Remoto)");
      Serial.println("Ciclo de limpeza interrompido remotamente.");
    }

  } else if (comandoPendente == "voltar_a_base") {
    Serial.println(">> ACIONAMENTO: Voltar para Base");
    // (Lógica da base aqui)
  }

  limparComandoPendente();
  comandoPendente = "";
}

/**
 * @brief Limpa o comando no Firestore, escrevendo "nenhum".
 */
void limparComandoPendente() {
  FirebaseJson content;
  content.set("fields/comando_pendente/stringValue", "nenhum");
  String mask = "comando_pendente";
  if (Firebase.Firestore.patchDocument(&fbdo, PROJECT_ID, "(default)", firestoreDocumentPath.c_str(), content.raw(), mask.c_str())) {
    Serial.println("Comando limpo no Firebase.");
  } else {
    Serial.println("ERRO ao limpar comando: " + fbdo.errorReason());
  }
}

/**
 * @brief Tenta conectar ao Wi-Fi com duas opções (Fallback).
 */
void conectarWiFi() {
  Serial.print("Conectando à Rede Principal (");
  Serial.print(wifi_ssid_1);
  Serial.print(")...");

  WiFi.begin(wifi_ssid_1, wifi_pass_1);
  unsigned long inicioTentativa = millis();

  // Tenta a Rede 1 por 10 segundos
  while (WiFi.status() != WL_CONNECTED && (millis() - inicioTentativa < 10000)) {
    Serial.print(".");
    delay(500); // Delay aceitável, pois só ocorre no setup
  }

  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\n✅ Wi-Fi CONECTADO (Rede Principal)!");
    Serial.println("IP: " + WiFi.localIP().toString());
    return;
  }

  Serial.println("\nFalha na Rede Principal.");
  Serial.print("Conectando à Rede Secundária (");
  Serial.print(wifi_ssid_2);
  Serial.print(")...");

  WiFi.begin(wifi_ssid_2, wifi_pass_2);
  inicioTentativa = millis();

  // Tenta a Rede 2 por mais 10 segundos
  while (WiFi.status() != WL_CONNECTED && (millis() - inicioTentativa < 10000)) {
    Serial.print(".");
    delay(500);
  }

  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\n✅ Wi-Fi CONECTADO (Rede Secundária)!");
    Serial.println("IP: " + WiFi.localIP().toString());
  } else {
    Serial.println("\n❌ ERRO: Falha ao conectar em ambas as redes.");
  }
}

/**
 * @brief Gerenciador de Conexão Wi-Fi e Firebase.
 */
void verificarConexoes() {
  if (millis() - ultimoCheckConexao < 30000) return;
  ultimoCheckConexao = millis();

  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("[Conexão] Wi-Fi perdido. Tentando reconectar...");
    WiFi.reconnect(); // Tenta reconectar à última rede que funcionou
  }
  else if (!Firebase.ready()) {
    Serial.println("[Conexão] Firebase não está pronto. Verificando...");
  }
}

/**
 * @brief Busca e atualiza as configurações de horário lendo o Firestore.
 */
void buscarConfiguracoesFirebase() {
  Serial.println("[Config] Buscando configurações de horário no Firebase...");

  if (Firebase.Firestore.getDocument(&fbdo, PROJECT_ID, "(default)", firestoreDocumentPath.c_str())) {
    if (fbdo.httpCode() == FIREBASE_ERROR_HTTP_CODE_OK) {
      FirebaseJson json(fbdo.payload().c_str());
      FirebaseJsonData result;

      json.get(result, "fields/config_hora_manha/stringValue");
      if (result.success) {
        String horaManhaStr = result.stringValue; // Ex: "07:00"
        int separador = horaManhaStr.indexOf(':');
        if (separador > 0) {
          configHoraManha = horaManhaStr.substring(0, separador).toInt();
          configMinManha = horaManhaStr.substring(separador + 1).toInt();
          Serial.printf("[Config] Horário da manhã atualizado para: %02d:%02d\n", configHoraManha, configMinManha);
        }
      } else {
        Serial.println("[Config] ERRO: Campo 'config_hora_manha' não encontrado. Usando padrão.");
      }

      json.get(result, "fields/config_hora_noite/stringValue");
      if (result.success) {
        String horaNoiteStr = result.stringValue; // Ex: "19:01"
        int separador = horaNoiteStr.indexOf(':');
        if (separador > 0) {
          configHoraNoite = horaNoiteStr.substring(0, separador).toInt();
          configMinNoite = horaNoiteStr.substring(separador + 1).toInt();
          Serial.printf("[Config] Horário da noite atualizado para: %02d:%02d\n", configHoraNoite, configMinNoite);
        }
      } else {
        Serial.println("[Config] ERRO: Campo 'config_hora_noite' não encontrado. Usando padrão.");
      }
    }
  } else {
    Serial.println("ERRO ao buscar configurações: " + fbdo.errorReason());
  }
}

/**
 * @brief Verifica se o sensor de chuva está molhado.
 */
bool estaChovendo() {
  int valorChuva = analogRead(pinSensorChuva);
  // Lógica invertida: valor baixo = molhado.
  if (valorChuva < CHUVA_THRESHOLD) {
    return true;
  } else {
    return false;
  }
}

/**
 * @brief Controla os motores da ESTEIRA (frente/trás/parar)
 */
void controlarEsteira(int direcao) {
  if (direcao == 1) { // Para Frente
    digitalWrite(pinMotor1, HIGH);
    digitalWrite(pinMotor2, LOW);
  } else if (direcao == -1) { // Para Trás
    digitalWrite(pinMotor1, LOW);
    digitalWrite(pinMotor2, HIGH);
  } else { // Parar (direcao == 0)
    digitalWrite(pinMotor1, LOW);
    digitalWrite(pinMotor2, LOW);
  }
}

/**
 * @brief Controla os motores do ROLO (ligado/desligado)
 */
void controlarRolo(bool ligado) {
  if (ligado) {
    digitalWrite(pinMotor3, HIGH);
    digitalWrite(pinMotor4, LOW);
  } else { // Desligado
    digitalWrite(pinMotor3, LOW);
    digitalWrite(pinMotor4, LOW);
  }
}

/**
 * @brief Desliga todos os motores
 */
void pararTodosMotores() {
  controlarEsteira(0); // 0 = Parar
  controlarRolo(false);  // false = Desligar
}
