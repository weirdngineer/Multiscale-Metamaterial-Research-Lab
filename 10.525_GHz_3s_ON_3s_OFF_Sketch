#include <Arduino.h>
#include <SPI.h>

// =============================================================
// PIN DEFINITIONS (ESP32 -> ADF5355 eval board test points)
// =============================================================
//
// PIN_SCLK : SPI clock line. This drives TP4 (CLK) on the eval board.
// PIN_MOSI : SPI data line (Master Out, Slave In). This drives TP3 (DATA).
// PIN_LE   : Latch Enable. This drives TP5 (LE). VERY important for ADF PLLs.
//            The chip only "accepts" the 32-bit word into a register when LE pulses.
// PIN_CE   : Chip Enable. This drives TP6 (CE). HIGH = synth active, LOW = power down.
// PIN_LD   : Lock Detect (usually MUXOUT configured as LD). This is an INPUT to ESP32.
//            If not wired, set it to -1 to disable lock reads.
//
static const int PIN_SCLK = 14;
static const int PIN_MOSI = 15;
static const int PIN_LE   = 27;
static const int PIN_CE   = 25;
static const int PIN_LD   = 26;   // set to -1 if not used

// =============================================================
// SPI OBJECTS (ESP32 has multiple SPI peripherals)
// =============================================================
//
// SPIClass hspi(HSPI) chooses the HSPI peripheral.
// ESP32 can route HSPI signals to many GPIO pins (via begin()).
// We'll use this as our SPI engine to shift 32-bit words out.
//
SPIClass hspi(HSPI);

// SPISettings(speedHz, bitOrder, dataMode)
//
// speedHz  = 1,000,000 => 1 MHz SPI clock (safe for bring-up)
// MSBFIRST = most-significant bit first (ADF PLLs expect MSB first)
// SPI_MODE0= clock idles low, data changes on falling edge, sampled on rising edge
//            (typical for ADI PLL parts)
//
// If you have clean wiring you can increase this later.
//
SPISettings pllSPI(1000000, MSBFIRST, SPI_MODE0);

// =============================================================
// pulseLE(): "Commit" the just-shifted 32 bits into an ADF register
// =============================================================
//
// On many ADI PLLs, there is no traditional chip-select for writing.
// You shift 32 bits in via DATA+CLK, then pulse LE.
// The LE rising edge (and/or pulse) causes the internal latch to update.
//
// The delays are tiny, just to ensure the pulse width is long enough.
//
static inline void pulseLE() {
  digitalWrite(PIN_LE, HIGH);
  delayMicroseconds(2);
  digitalWrite(PIN_LE, LOW);
  delayMicroseconds(2);
}

// =============================================================
// writeReg(reg): Shift out one 32-bit register word, then pulse LE
// =============================================================
//
// This function does the actual SPI transfer.
// ADF registers are 32 bits, so we send 4 bytes, MSB first.
//
// beginTransaction/endTransaction applies the SPI settings safely.
//
// After the bytes are clocked out, we pulse LE so the ADF chip latches the word.
//
static void writeReg(uint32_t reg) {
  hspi.beginTransaction(pllSPI);

  // Send MSB first
  hspi.transfer((reg >> 24) & 0xFF);
  hspi.transfer((reg >> 16) & 0xFF);
  hspi.transfer((reg >>  8) & 0xFF);
  hspi.transfer((reg >>  0) & 0xFF);

  hspi.endTransaction();

  // Critical for ADF PLLs: latch the shift register into the addressed register
  pulseLE();
}

// =============================================================
// programPLL(): Write a full register configuration to the ADF5355
// =============================================================
//
// This is the "program the chip" step.
//
// The ADF5355 has registers R0..R12 (13 registers total).
// Each 32-bit word typically contains:
//
//  - control bits and numeric fields (INT, FRAC, MOD, output enable, power, etc)
//  - and the register address in the low bits (so the chip knows which register)
//
// WHY WRITE ORDER MATTERS:
// Many ADI PLLs want you to write from highest register down to R0.
// R0 is often written last because it can trigger calibration/update/lock sequence.
// If you write R0 early and then change other settings after, you can get weird behavior.
//
void programPLL() {

  Serial.println("\nProgramming ADF5355 for 10.525 GHz RFOUTB...");

  // -----------------------------------------------------------
  // regs[]: 13 x 32-bit words to configure the chip
  // -----------------------------------------------------------
  //
  // IMPORTANT:
  // These numbers are NOT self-describing. Each one is a packed bitfield.
  // The comment "R12" means: "this word targets register 12" (address bits).
  //
  // In a correct ADF5355 word, the *lowest bits* encode the register number.
  // Example pattern: ...0C means register 12, ...0B means register 11, etc.
  //
  // WHAT THEY "MEAN" FUNCTIONALLY (high level):
  //
  // R0/R1/R2 are usually the big frequency math registers:
  //   - INT / FRAC / MOD
  //   - reference divider/doubler, PFD settings
  //
  // Mid registers typically control:
  //   - charge pump current
  //   - loop filter / lock detect settings
  //   - output enable & output power for RFOUTA/RFOUTB
  //
  // Upper registers often control:
  //   - VCO calibration behavior / internal biasing
  //   - mux settings
  //   - misc “plumbing” needed for stable operation
  //
  // Without the ADF5355 datasheet map, we cannot truthfully say
  // "bit 17 does X" for each of these particular words.
  //
  // If you paste the exact register map table (R0..R12 bit descriptions)
  // or use ADI software exported registers, we can decode each field.
  //
  uint32_t regs[13] = {
    0x0001040C, // R12 (address bits indicate register 12)
    0x0061300B, // R11
    0x00C0000A, // R10
    0x00000009, // R9
    0x102D0428, // R8  <-- NOTE: this ends with 0x28 (not 0x08). That is a red flag:
                // many ADF PLLs use the lowest 4 bits as the register address (0..12).
                // If that's true for ADF5355, 0x...28 would imply address=8 (0b1000) *and*
                // extra bits set in the low nibble, which might be invalid or just a different encoding.
                // This is why I treat these values as "placeholder-ish" unless verified.
    0x12000007, // R7
    0x35000006, // R6
    0x00800005, // R5
    0x00000004, // R4
    0x00000003, // R3
    0x00001002, // R2
    0x00000A41, // R1
    0x00550000  // R0 LAST (often triggers update/cal)
  };

  // -----------------------------------------------------------
  // Write them all out in sequence
  // -----------------------------------------------------------
  for (int i = 0; i < 13; i++) {

    // 12 - i is just printing "R12 ... R0" while i goes 0..12
    Serial.printf("Writing register %d: 0x%08X\n", 12 - i, regs[i]);

    // Actually send it to the ADF chip
    writeReg(regs[i]);

    // Tiny delay so the chip has time between writes (usually not required,
    // but helps with bring-up + messy wiring)
    delay(2);
  }

  Serial.println("PLL programmed.\n");
}

// =============================================================
// setup(): Runs once at power-up/reset
// =============================================================
void setup() {
  Serial.begin(115200);
  delay(1000);

  Serial.println("ADF5355 10.525 GHz 3s ON / 3s OFF Test");

  // Configure LE/CE as outputs
  pinMode(PIN_LE, OUTPUT);
  pinMode(PIN_CE, OUTPUT);

  // Start with LE low and the synth disabled (CE low)
  digitalWrite(PIN_LE, LOW);
  digitalWrite(PIN_CE, LOW);

  // Configure LD pin as input IF you wired it.
  // (If PIN_LD = -1 we skip this entirely.)
  if (PIN_LD >= 0) {
    pinMode(PIN_LD, INPUT);
  }

  // Start HSPI using the chosen pins.
  // begin(SCLK, MISO, MOSI, SS)
  //
  // We pass -1 for MISO and SS because:
  //  - ADF5355 is write-only in this interface (no MISO needed)
  //  - We aren't using a chip select; LE is the latch trigger instead
  //
  hspi.begin(PIN_SCLK, -1, PIN_MOSI, -1);

  // Program the PLL registers once at startup.
  // After this, the output frequency *should* be configured (if regs[] is correct).
  programPLL();
}

// =============================================================
// loop(): Runs forever after setup()
// =============================================================
void loop() {

  // === TRANSMIT ON ===
  //
  // CE HIGH powers up/enables the synthesizer core.
  // If the PLL is configured correctly, the RF output should appear.
  //
  Serial.println("TRANSMIT ON (CE HIGH)");
  digitalWrite(PIN_CE, HIGH);

  // Optionally sample lock detect after 200 ms.
  // This is useful because PLLs can take some time to lock after enabling.
  //
  if (PIN_LD >= 0) {
    delay(200);
    int lockState = digitalRead(PIN_LD);
    Serial.printf("Lock status: %d\n", lockState); // 1=locked, 0=not locked (typical)
  }

  // Keep transmitting for 3 seconds
  delay(3000);

  // === TRANSMIT OFF ===
  //
  // CE LOW powers down/disables the synthesizer core,
  // so output should disappear.
  //
  Serial.println("TRANSMIT OFF (CE LOW)");
  digitalWrite(PIN_CE, LOW);

  // Stay off for 3 seconds
  delay(3000);
}
