# volant

// Definice
  #include <Adafruit_NeoPixel.h>

  #define ENCODER1_CLK 2
  #define ENCODER1_DT 3
  #define ENCODER1_SW 4
  #define ENCODER2_CLK 5  // S1 na druhém enkodéru
  #define ENCODER2_DT 6   // S2 na druhém enkodéru
  #define ENCODER2_SW 7   // KEY na druhém enkodéru
  #define LED_PIN 8
  #define NUM_LEDS 24

  Adafruit_NeoPixel strip = Adafruit_NeoPixel(NUM_LEDS, LED_PIN, NEO_GRB + NEO_KHZ800);

  int lastCLK1 = LOW;
  int currentCLK1;
  int lastCLK2 = LOW;
  int currentCLK2;
  int section = 0;  // Current section (0-5)
  int pulseCount = 0;  // Pulse count for the first encoder
  unsigned long lastButtonPress1 = 0;
  unsigned long lastButtonPress2 = 0;
  int currentGame = 0;  // Current game (1-6)
  bool gameActive = false;  // Flag to indicate if a game is active
//

// Proměnné pro Hru 1
  int currentLED = 0;
  int speed = 50;  // Speed for rotating LED
  bool direction = true;  // Direction for rotating LED
  unsigned long lastUpdate = 0;
//

// Proměnné pro Hru 2
  int game2LEDs[4];  // Čtyři náhodné LED diody
  int game2TargetLED;  // Cílová LED dioda
  int game2PlayerLED = 0;  // Aktuální LED dioda hráče
  bool game2Active = false;  // Příznak pro aktivitu hry
  unsigned long game2StartTime = 0;
  int currentLevel = 1;  // Aktuální úroveň
  int maxLevel = 8;      // Maximální úroveň
  int numLEDs;           // Počet LED diod pro aktuální úroveň
  int displayTime;       // Doba zobrazení LED diod pro aktuální úroveň
//

// Proměnné pro Hru 3
  int game3Sequence[10];  // Sekvence LED diod
  int game3Index = 0;     // Aktuální index v sekvenci
  int game3PlayerIndex = 0;  // Aktuální index hráče
  bool game3PlayerTurn = false;  // Příznak pro tah hráče
  unsigned long game3LastUpdate = 0;
  int game3Speed = 500;  // Rychlost zobrazení sekvence
  int game3Level = 1;  // Aktuální úroveň
  int game3MaxLevel = 8;  // Maximální úroveň
//

// Proměnné pro Hru 4
  int catchPlayerPos = 0;  // Pozice hráče
  int catchTargetPos = 0;  // Pozice cílové červené LEDky
  unsigned long catchLastMoveTime = 0;
  int catchSpeed = 500;  // Rychlost pohybu červené LEDky (v ms)
  bool catchActive = false;
  int catchLevel = 1;  // Aktuální úroveň
  int catchMaxLevel = 8;  // Maximální úroveň
  bool catchDirection = true;  // Směr pohybu červené LEDky (true = vpřed, false = vzad)
//

void setup() {
  pinMode(ENCODER1_CLK, INPUT);
  pinMode(ENCODER1_DT, INPUT);
  pinMode(ENCODER1_SW, INPUT_PULLUP);
  pinMode(ENCODER2_CLK, INPUT);
  pinMode(ENCODER2_DT, INPUT);
  pinMode(ENCODER2_SW, INPUT_PULLUP);
  strip.begin();
  strip.setBrightness(50);  // Default brightness to 50%
  strip.show();              // Initialize all pixels to 'off'
  Serial.begin(9600);        // Initialize serial communication for debugging

  // Inicializace proměnných pro Hru 2
  randomSeed(analogRead(0));  // Inicializace generátoru náhodných čísel

  // Inicializace sekvence pro Hru 3
  for (int i = 0; i < 10; i++) {
    game3Sequence[i] = random(NUM_LEDS);
    }
}

void loop() {
  currentCLK1 = digitalRead(ENCODER1_CLK);

  // Zpracování prvního enkodéru pro výběr sekce
  if (!gameActive && currentCLK1 != lastCLK1) {
    if (digitalRead(ENCODER1_DT) != currentCLK1) {
      pulseCount++;
    } else {
      pulseCount--;
    }

    // Zajištění, že pulseCount je v rozmezí 0-29
    if (pulseCount < 0) {
      pulseCount = 29;
    } else if (pulseCount > 29) {
      pulseCount = 0;
    }

    // Určení aktuální sekce na základě pulseCount
    section = pulseCount / 5;
    updateLEDs();
  }

  // Zpracování stisku tlačítka na prvním enkodéru
  if (digitalRead(ENCODER1_SW) == LOW) {
    if (millis() - lastButtonPress1 > 500) {  // Debounce tlačítka
      if (!gameActive) {
        currentGame = section + 1;  // Výběr hry na základě aktuální sekce
        Serial.print("Vybraná hra: ");
        Serial.println(currentGame);
        if (currentGame == 1 || currentGame == 2 || currentGame == 3 || currentGame == 4 || currentGame == 5 || currentGame == 6) {
          blinkConfirmation(section);  // Blikající efekt pro potvrzení výběru hry
          delay(500);  // Zpoždění před spuštěním hry
          gameActive = true;  // Aktivace hry
        }
      } else {
        gameActive = false;  // Deaktivace hry
        resetGame();  // Resetování proměnných hry
        updateLEDs();  // Návrat do režimu výběru sekce
      }
      lastButtonPress1 = millis();
    }
  }

  lastCLK1 = currentCLK1;

  // Spuštění aktivní hry
  if (gameActive) {
    if (currentGame == 1) {
      runGame1();
    } else if (currentGame == 2) {
      runGame2();
    } else if (currentGame == 3) {
      runGame3();
    } else if (currentGame == 4) {
      runGame4();
    } else if (currentGame == 5) {
      runGame5();
    } else if (currentGame == 6) {
      runGame6();
    }
  }
}

void updateLEDs() {
  strip.clear();

  // Definuj barvy pro každou sekci
  uint32_t colors[6] = {
    strip.Color(255, 0, 0),    // Červená
    strip.Color(0, 255, 0),    // Zelená
    strip.Color(0, 0, 255),    // Modrá
    strip.Color(255, 255, 0),  // Žlutá
    strip.Color(0, 255, 255),  // Azurová
    strip.Color(255, 0, 255)   // Magenta
  };

  // Dim all LEDs with section colors
  for (int i = 0; i < NUM_LEDS; i++) {
    int sectionIndex = i / (NUM_LEDS / 6);
    strip.setPixelColor(i, dimColor(colors[sectionIndex], 10));  // Dim section color
  }

  // Highlight the selected section
  int startLED = section * (NUM_LEDS / 6);
  int endLED = startLED + (NUM_LEDS / 6);
  for (int i = startLED; i < endLED; i++) {
    strip.setPixelColor(i, colors[section]);
  }

  strip.show();
}

uint32_t dimColor(uint32_t color, int brightness) {
  uint8_t r = (color >> 16) & 0xFF;
  uint8_t g = (color >> 8) & 0xFF;
  uint8_t b = color & 0xFF;
  r = (r * brightness) / 255;
  g = (g * brightness) / 255;
  b = (b * brightness) / 255;
  return strip.Color(r, g, b);
}

void blinkConfirmation(int section) {
  // Definuj barvy pro každou sekci
  uint32_t colors[6] = {
    strip.Color(255, 0, 0),    // Červená
    strip.Color(0, 255, 0),    // Zelená
    strip.Color(0, 0, 255),    // Modrá
    strip.Color(255, 255, 0),  // Žlutá
    strip.Color(0, 255, 255),  // Azurová
    strip.Color(255, 0, 255)   // Magenta
  };

  // Získej začátek a konec sekce
  int startLED = section * (NUM_LEDS / 6);
  int endLED = startLED + (NUM_LEDS / 6);

  // Blikání LED diod ve vybrané sekci
  for (int i = 0; i < 3; i++) {
    for (int j = startLED; j < endLED; j++) {
      strip.setPixelColor(j, colors[section]);
    }
    strip.show();
    delay(200);  // Počkej 200 ms
    strip.clear();
    strip.show();
    delay(200);  // Počkej 200 ms
  }
}

void setLevel(int level) {
  switch (level) {
    case 1:
      numLEDs = 4;
      displayTime = 1000;
      break;
    case 2:
      numLEDs = 4;
      displayTime = 500;
      break;
    case 3:
      numLEDs = 3;
      displayTime = 1000;
      break;
    case 4:
      numLEDs = 3;
      displayTime = 500;
      break;
    case 5:
      numLEDs = 2;
      displayTime = 1000;
      break;
    case 6:
      numLEDs = 2;
      displayTime = 500;
      break;
    case 7:
      numLEDs = 1;
      displayTime = 1000;
      break;
    case 8:
      numLEDs = 1;
      displayTime = 500;
      break;
    default:
      numLEDs = 4;
      displayTime = 1000;
      break;
  }
}

void resetGame() {
  game2Active = false;
  game2StartTime = 0;
  game2PlayerLED = 0;
  lastButtonPress2 = 0;
  currentLevel = 1;  // Reset úrovně na začátek

  // Reset proměnných pro Hru 3
  game3Index = 0;
  game3PlayerIndex = 0;
  game3PlayerTurn = false;
  game3LastUpdate = 0;
  game3Level = 1;  // Reset úrovně na začátek

  // Reset proměnných pro Hru 4
  catchPlayerPos = 0;
  catchTargetPos = 0;
  catchLastMoveTime = 0;
  catchSpeed = 500;
  catchActive = false;
  catchLevel = 1;
  catchDirection = true;
}

void runGame1() {
  currentCLK2 = digitalRead(ENCODER2_CLK);

  // Handle second encoder for speed control
  if (currentCLK2 != lastCLK2) {
    if (digitalRead(ENCODER2_DT) == currentCLK2) {
      speed = min(speed + 5, 255);  // Increase speed
    } else {
      speed = max(speed - 5, 0);  // Decrease speed
    }
    lastCLK2 = currentCLK2;
  }

  // Handle button press on second encoder for direction change
  if (digitalRead(ENCODER2_SW) == LOW) {
    if (millis() - lastButtonPress2 > 500) {  // Debounce button
      direction = !direction;  // Change direction
      lastButtonPress2 = millis();
    }
  }

  // Update LED position based on speed and direction
  if (millis() - lastUpdate > (255 - speed)) {
    lastUpdate = millis();
    strip.clear();
    strip.setPixelColor(currentLED, strip.Color(255, 0, 0));  // Red color for the moving LED
    strip.show();
    if (direction) {
      currentLED = (currentLED + 1) % NUM_LEDS;
    } else {
      currentLED = (currentLED - 1 + NUM_LEDS) % NUM_LEDS;
    }
  }
}

void runGame2() {
  if (!game2Active) {
    setLevel(currentLevel);  // Nastavení aktuální úrovně

    // Zobrazení náhodných sousedních LED diod na začátku
    int startLED = random(NUM_LEDS);
    for (int i = 0; i < numLEDs; i++) {
      game2LEDs[i] = (startLED + i) % NUM_LEDS;
      strip.setPixelColor(game2LEDs[i], strip.Color(0, 255, 0));  // Zelená barva
    }
    strip.show();
    delay(displayTime);  // Počkej podle aktuální úrovně
    strip.clear();
    strip.show();
    
    // Rozsvícení jedné náhodné LED diody z těch zobrazených
    game2TargetLED = game2LEDs[random(numLEDs)];
    game2Active = true;
    game2StartTime = millis();

    // Náhodná počáteční pozice modré LED diody, ale ne v blízkosti zelených LED diod
    do {
      game2PlayerLED = random(NUM_LEDS);
    } while (abs(game2PlayerLED - startLED) < 8 || abs(game2PlayerLED - (startLED + numLEDs - 1) % NUM_LEDS) < 8);
    
    // Zobrazení modré LED diody na náhodné pozici
    strip.setPixelColor(game2PlayerLED, strip.Color(0, 0, 255));  // Modrá barva pro výběr hráče
    strip.show();
  } else {
    // Hráč hledá pozici pomocí 2. enkodéru
    currentCLK2 = digitalRead(ENCODER2_CLK);
    if (currentCLK2 != lastCLK2) {
      if (digitalRead(ENCODER2_DT) == currentCLK2) {
        game2PlayerLED = (game2PlayerLED + 1) % NUM_LEDS;
      } else {
        game2PlayerLED = (game2PlayerLED - 1 + NUM_LEDS) % NUM_LEDS;
      }
      lastCLK2 = currentCLK2;
      strip.clear();
      strip.setPixelColor(game2PlayerLED, strip.Color(0, 0, 255));  // Modrá barva pro výběr hráče
      strip.show();
    }

    if (digitalRead(ENCODER2_SW) == LOW) {
      if (millis() - lastButtonPress2 > 500) {  // Debounce tlačítka
        bool correct = false;
        for (int i = 0; i < numLEDs; i++) {
          if (game2PlayerLED == game2LEDs[i]) {
            correct = true;
            break;
          }
        }
        if (correct) {
          // Správná pozice, zablikání LED diod a zvýšení úrovně
          for (int i = 0; i < numLEDs; i++) {
            strip.setPixelColor(game2LEDs[i], strip.Color(0, 255, 0));  // Zelená barva
          }
          strip.show();
          delay(500);
          strip.clear();
          strip.show();
          delay(500);
          game2Active = false;
          if (currentLevel < maxLevel) {
            currentLevel++;  // Zvýšení úrovně
          }
        } else {
          // Špatná pozice, zablikání každé druhé LED diody žlutě a restart hry
          for (int i = 0; i < NUM_LEDS; i += 2) {
            strip.setPixelColor(i, strip.Color(255, 255, 0));  // Žlutá barva
          }
          strip.show();
          delay(500);
          strip.clear();
          strip.show();
          delay(500);
          game2Active = false;
          currentLevel = 1;  // Reset úrovně na začátek
        }
        lastButtonPress2 = millis();
      }
    }
  }
}

void runGame3() {
  if (!game3PlayerTurn) {
    // Zobrazení sekvence
    if (millis() - game3LastUpdate > game3Speed) {
      game3LastUpdate = millis();
      strip.clear();
      strip.setPixelColor(game3Sequence[game3Index], strip.Color(0, 255, 0));  // Zelená barva pro sekvenci
      strip.show();
      delay(1000);  // Delší svícení LEDek
      strip.clear();
      strip.show();
      game3Index++;
      if (game3Index >= game3Level) {
        game3Index = 0;
        game3PlayerTurn = true;  // Tah hráče pro zopakování sekvence
        strip.clear();
        strip.show();

        // Náhodná počáteční pozice modré LED diody, ale ne v blízkosti zelených LED diod
        int startLED = game3Sequence[0];
        do {
          game3PlayerIndex = random(NUM_LEDS);
        } while (abs(game3PlayerIndex - startLED) < 8 || abs(game3PlayerIndex - (startLED + game3Level - 1) % NUM_LEDS) < 8);

        // Zobrazení modré LED diody na náhodné pozici
        strip.setPixelColor(game3PlayerIndex, strip.Color(0, 0, 255));  // Modrá barva pro výběr hráče
        strip.show();
      }
    }
  } else {
    // Tah hráče pro zopakování sekvence
    currentCLK2 = digitalRead(ENCODER2_CLK);
    if (currentCLK2 != lastCLK2) {
      if (digitalRead(ENCODER2_DT) == currentCLK2) {
        game3PlayerIndex = (game3PlayerIndex + 1) % NUM_LEDS;
      } else {
        game3PlayerIndex = (game3PlayerIndex - 1 + NUM_LEDS) % NUM_LEDS;
      }
      lastCLK2 = currentCLK2;
      strip.clear();
      for (int i = 0; i < game3Index; i++) {
        strip.setPixelColor(game3Sequence[i], strip.Color(0, 100, 0));  // Lehce zelená barva pro správně označené diody
      }
      strip.setPixelColor(game3PlayerIndex, strip.Color(0, 0, 255));  // Modrá barva pro výběr hráče
      strip.show();
    }

    if (digitalRead(ENCODER2_SW) == LOW) {
      if (millis() - lastButtonPress2 > 500) {  // Debounce tlačítka
        if (game3PlayerIndex == game3Sequence[game3Index]) {
          game3Index++;
          if (game3Index >= game3Level) {
            // Hráč úspěšně zopakoval sekvenci, zvýšení úrovně
            for (int i = 0; i < NUM_LEDS; i += 2) {
              strip.setPixelColor(i, strip.Color(0, 255, 0));  // Zelená barva
            }
            strip.show();
            delay(500);
            strip.clear();
            strip.show();
            delay(500);
            game3PlayerTurn = false;
            game3Index = 0;
            if (game3Level < game3MaxLevel) {
              game3Level++;  // Zvýšení úrovně
            }
          }
        } else {
          // Hráč udělal chybu, zablikání každé druhé LED diody žlutě a restart hry
          for (int i = 0; i < NUM_LEDS; i += 2) {
            strip.setPixelColor(i, strip.Color(255, 255, 0));  // Žlutá barva
          }
          strip.show();
          delay(500);
          strip.clear();
          strip.show();
          delay(500);
          game3PlayerTurn = false;
          game3Index = 0;
          game3Level = 1;  // Reset úrovně na začátek
        }
        lastButtonPress2 = millis();
      }
    }
  }
}

void runGame4() {
  if (!catchActive) {
    // Inicializace hry
    catchPlayerPos = 0;
    catchTargetPos = random(NUM_LEDS);
    catchLastMoveTime = millis();
    catchActive = true;
    setCatchLevel(catchLevel);
    strip.clear();
    strip.setPixelColor(catchPlayerPos, strip.Color(0, 0, 255));  // Modrá barva pro hráče
    strip.setPixelColor(catchTargetPos, strip.Color(255, 0, 0));  // Červená barva pro cíl
    strip.show();
  } else {
    // Aktualizace pozice hráče
    currentCLK2 = digitalRead(ENCODER2_CLK);
    if (currentCLK2 != lastCLK2) {
      if (digitalRead(ENCODER2_DT) == currentCLK2) {
        catchPlayerPos = (catchPlayerPos + 1) % NUM_LEDS;
      } else {
        catchPlayerPos = (catchPlayerPos - 1 + NUM_LEDS) % NUM_LEDS;
      }
      lastCLK2 = currentCLK2;
      strip.clear();
      if (catchPlayerPos == catchTargetPos) {
        strip.setPixelColor(catchPlayerPos, strip.Color(255, 0, 255));  // Fialová barva pro shodnou pozici
      } else {
        strip.setPixelColor(catchPlayerPos, strip.Color(0, 0, 255));  // Modrá barva pro hráče
        strip.setPixelColor(catchTargetPos, strip.Color(255, 0, 0));  // Červená barva pro cíl
      }
      strip.show();
    }

    // Pohyb červené LEDky
    if (millis() - catchLastMoveTime > catchSpeed) {
      catchLastMoveTime = millis();
      if (catchDirection) {
        catchTargetPos = (catchTargetPos + 1) % NUM_LEDS;
      } else {
        catchTargetPos = (catchTargetPos - 1 + NUM_LEDS) % NUM_LEDS;
      }
      // Změna směru pohybu na vyšších úrovních
      if (catchLevel >= 3 && random(10) < 2) {
        catchDirection = !catchDirection;
      }
      // Přeskoky na vyšších úrovních
      if (catchLevel >= 5 && random(10) < 3) {
        catchTargetPos = (catchTargetPos + (catchDirection ? 6 : -6) + NUM_LEDS) % NUM_LEDS;
      }
      if (catchLevel >= 7 && random(10) < 3) {
        catchTargetPos = (catchTargetPos + (catchDirection ? random(5) : -random(5)) + NUM_LEDS) % NUM_LEDS;
      }
      strip.clear();
      if (catchPlayerPos == catchTargetPos) {
        strip.setPixelColor(catchPlayerPos, strip.Color(255, 0, 255));  // Fialová barva pro shodnou pozici
      } else {
        strip.setPixelColor(catchPlayerPos, strip.Color(0, 0, 255));  // Modrá barva pro hráče
        strip.setPixelColor(catchTargetPos, strip.Color(255, 0, 0));  // Červená barva pro cíl
      }
      strip.show();
    }

    // Kontrola, zda hráč chytil červenou LEDku
    if (digitalRead(ENCODER2_SW) == LOW) {
      if (millis() - lastButtonPress2 > 500) {  // Debounce tlačítka
        if (catchPlayerPos == catchTargetPos) {
          // Správná pozice, zablikání zeleně a zvýšení úrovně
          for (int i = 0; i < NUM_LEDS; i += 2) {
            strip.setPixelColor(i, strip.Color(0, 255, 0));  // Zelená barva
          }
          strip.show();
          delay(500);
          strip.clear();
          strip.show();
          delay(500);
          catchActive = false;
          if (catchLevel < catchMaxLevel) {
            catchLevel++;  // Zvýšení úrovně
          }
        } else {
          // Špatná pozice, zablikání žlutě a restart hry
          for (int i = 0; i < NUM_LEDS; i += 2) {
            strip.setPixelColor(i, strip.Color(255, 255, 0));  // Žlutá barva
          }
          strip.show();
          delay(500);
          strip.clear();
          strip.show();
          delay(500);
          catchActive = false;
          catchLevel = 1;  // Reset úrovně na začátek
        }
        lastButtonPress2 = millis();
      }
    }
  }
}

void runGame5() {

}

void runGame6() {
  
}

void setCatchLevel(int level) {
  switch (level) {
    case 1:
      catchSpeed = 500;
      break;
    case 2:
      catchSpeed = 300;
      break;
    case 3:
      catchSpeed = 500;
      break;
    case 4:
      catchSpeed = 300;
      break;
    case 5:
      catchSpeed = 500;
      break;
    case 6:
      catchSpeed = 300;
      break;
    case 7:
      catchSpeed = 500;
      break;
    case 8:
      catchSpeed = 300;
      break;
    default:
      catchSpeed = 500;
      break;
  }
}
