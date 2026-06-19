## TFA98XX Driver - Compatibility Adaptation for Mediatek I2C Extension


### TFA9890 Audio Amplifier - Key Specifications

### Overview
NXP TFA9890 is a high-performance Class-D audio amplifier with integrated DSP and DC-DC boost converter, designed for mobile micro speakers.

### Key Features Relevant to Driver Development

| Feature | Detail |
|---|---|
| Control Interface | I2C (400 kHz Fast Mode) |
| DSP | NXP CoolFlux, embedded firmware |
| Output Power | 3.6W RMS into 8Ω (9.5V boost) |
| Supply Voltage | 2.7V - 5.5V |
| Sample Rates | 8 kHz - 48 kHz |
| I2S Inputs | 2 (dual audio source support) |
| Speaker Protection | Adaptive excursion control, real-time temperature monitoring |
| Firmware | Container-based (.cnt), profiles for music/voice |

### Driver Requirements

| Requirement | Implementation |
|---|---|
| I2C Communication | 400 kHz Fast Mode, 16-bit registers |
| Firmware Loading | Container (.cnt) loaded via regmap or direct I2C |
| DSP Control | Start/stop, profile switching (Music/Voice) |
| Audio Routing | ALSA SoC codec integration |
| Interrupt Handling | Speaker error, thermal warning, clip detection |
| Power Management | Suspend/resume, LDO control, GPIO reset |


#### Main Error ([error log](./dev_log_main_err.txt))

### Config
```
CONFIG_MTK_I2C_EXTENSION=y
```

```
static struct i2c_algorithm mt_i2c_algorithm = {
#ifdef CONFIG_MTK_I2C_EXTENSION
	.master_xfer = (pmaster_xfer)mtk_i2c_transfer,
#else
	.master_xfer = standard_i2c_transfer,
#endif
	.functionality = mt_i2c_functionality,
};
```

<pre>
  
                     KERNEL LINUX (GENERIC)
                    ┌─────────────────────┐
                    │  i2c_transfer()     │
                    │  regmap, SMBus, dll │
                    └──────────┬──────────┘
                               │ struct i2c_msg (standar)
                               │ no timing
                               │ no ext_flag
                               ▼
              ┌────────────────────────────────┐
              │   CONFIG_MTK_I2C_EXTENSION?    │
              └────────┬───────────────┬───────┘
                       │               │
                       =y              =n
                       │               │
                       ▼               ▼
              ┌──────────────┐  ┌──────────────┐
              │ mt_i2c_msg   │  │ i2c_msg      │
              │ (custom MTK) │  │ (standard)   │
              │ * timing     │  │              │
              │ * ext_flag   │  │              │
              └──────────────┘  └──────────────┘

</pre>

### Garbage Flow Error
<pre>
  
                                                 struct i2c_msg (12 BYTE)
        #include &lt;linux/i2c.h&gt;            ┌────────────────────────────────┐
        ─────────────────────────►        │ addr │ flags │ len  │ *buf     │
                                          └──────┴───────┴──────┴──────────┘
                                          no member timing
                                          no member ext_flag

                                                    │
                                                    │ CAST
                                                    ▼

        #include &lt;linux/mt_i2c.h&gt;        ┌────────────────────────────────────────────────────┐
        ─────────────────────────►       │ addr │ flags │ len  │ *buf     │ timing │ ext_flag │
                                         └──────┴───────┴──────┴──────────┴────────┴──────────┘
                                         has member timing (4 byte) → value 400 KHz
                                         has member  ext_flag (4 byte) → value 0


        ═══════════════════════════════════════════════════════════════════════════════════════
        COMPARING:
        &lt;linux/i2c.h&gt;                       &lt;linux/mt_i2c.h&gt;
        ┌────┬────┬────┬────┐               ┌────┬────┬────┬────┬────────┬──────────┐
        │addr│flg │len │*buf│               │addr│flg │len │*buf│ timing │ ext_flag │
        └────┴────┴────┴────┘               └────┴────┴────┴────┴──┬─────┴────┬─────┘
                                                                   │          │
              CAST ▼                                               │          │
        ┌────┬────┬────┬────┬────────┬──────────┐                  │          │
        │addr│flg │len │*buf│ ?????? │ ???????? │                  ▼          ▼
        └────┴────┴────┴────┴──┬─────┴────┬─────┘             400 KHz     0 (safe)
                               │          │
                         GARBAGE!    GARBAGE!
                         0x4DE6D418  0xFFFFFFC0
                         (54 MHz!)   (RND!)
                               │          │
                               ▼          ▼
                          ┌─────────────────────┐
                          │  ERROR!             │
                          │  speed too fast     │
                          │  probe FAILS        │
                          └─────────────────────┘
</pre>

Result ([Log 1](./LOG1.txt)): 
```
		dev_err("Failed to read Revision register: -22")
		return -EIO;
```


### Fix Result ([Log 2](./LOG2.txt)).
<pre>
╔══════════════════════════════════════════════════════════════════════════════════════════╗
║                           COMPARE 3 METHODE I2C READ - TFA98XX                           ║
╠══════════════════════════════╦══════════════════════════════╦════════════════════════════╣
║  SMBus Byte                  ║  SMBus Word                  ║  Regmap Custom             ║
╠══════════════════════════════╬══════════════════════════════╬════════════════════════════╣
║                              ║                              ║                            ║
║  Func Name:                  ║  Func Name:                  ║  Func Name:                ║
║  i2c_smbus_read_byte_data()  ║  i2c_smbus_read_word_data()  ║  tfa98xx_regmap_read()     ║
║                              ║                              ║                            ║
╠══════════════════════════════╬══════════════════════════════╬════════════════════════════╣
║  PARAMS                      ║  PARAMS                      ║  PARAMS                    ║
║  ───────────────────────     ║  ───────────────────────     ║  ─────────────────────     ║
║  timing   = 0x0              ║  timing   = 0x0              ║  timing   = 0x0            ║
║  ext_flag = 0x0              ║  ext_flag = 0x0              ║  ext_flag = 0x0            ║
║  mode     = ST_MODE          ║  mode     = ST_MODE          ║  mode     = ST_MODE        ║
║  speed    = 100 KHz          ║  speed    = 100 KHz          ║  speed    = 100 KHz        ║
║  dma_en   = false            ║  dma_en   = false            ║  dma_en   = false          ║
║                              ║                              ║                            ║
╠══════════════════════════════╬══════════════════════════════╬════════════════════════════╣
║  TRANSFER                    ║  TRANSFER                    ║  TRANSFER                  ║
║  ───────────────────────     ║  ───────────────────────     ║  ─────────────────────     ║
║  Message : 1 (WRITE)         ║  Message : 1 (WRITE+READ)    ║  Message : 2 (WR + RD)     ║
║  Data    : 1 byte            ║  Data    : 2 byte            ║  Data    : 2 byte          ║
║  Protocol: SMBus             ║  Protocol: SMBus             ║  Protocol: I2C bare        ║
║                              ║                              ║                            ║
║  I2C Wire:                   ║  I2C Wire:                   ║  I2C Wire:                 ║
║  ┌──────────────────────┐    ║  ┌──────────────────────┐    ║  ┌──────────────────────┐  ║
║  │ S │ ADDR+W │ CMD │   │    ║  │ S │ ADDR+W │ CMD │   │    ║  │ S │ ADDR+W │ CMD │   │  ║
║  │   │        │0x03 │   │    ║  │   │        │0x03 │   │    ║  │   │        │0x03 │   │  ║
║  │   │ DATA_L │ STOP│   │    ║  │ Sr│ ADDR+R │     │   │    ║  │ Sr│ ADDR+R │     │   │  ║
║  │   │  0x00  │     │   │    ║  │   │ DATA_L │DTA_H│   │    ║  │   │ DATA_H │DTA_L│   │  ║
║  └──────────────────────┘    ║  │   │  0x00  │0x80 │   │    ║  │   │  0x00  │0x80 │   │  ║
║                              ║  │   │ STOP   │     │   │    ║  │   │ STOP   │     │   │  ║
║                              ║  └──────────────────────┘    ║  └──────────────────────┘  ║
║                              ║                              ║                            ║
╠══════════════════════════════╬══════════════════════════════╬════════════════════════════╣
║  Result                      ║  Result                      ║  Result                    ║
║  ───────────────────────     ║  ───────────────────────     ║  ─────────────────────     ║
║  Raw Data: 0x00              ║  Raw Data: 0x8000            ║  Raw Data: 0x00 0x80       ║
║  Value   : 0x00              ║  Value   : 0x8000            ║  Value   : 0x0080          ║
║                              ║                              ║                            ║
║  Only LOW byte               ║  Byte Order Swap             ║  HIGH=0x00, LOW=0x80       ║
║  8-bit                       ║                              ║  Revision = 0x80           ║
║                              ║                              ║  TFA9890 detected          ║
║                              ║                              ║                            ║
║                              ║                              ║                            ║
╠══════════════════════════════╩══════════════════════════════╩════════════════════════════╣
║                                                                                          ║
║  Result:                                                                                 ║
║  ──────────────────────────────────────────────────────────────────────────────────────  ║
║  SMBus Byte  = 8-bit, not enough for register 16-bit                                     ║
║  SMBus Word  = 16-bit byte order swap (little-endian vs big-endian)                      ║
║  Regmap Custom = 16-bit, byte order valid, flexible (support DMA, change speed)          ║
║                                                                                          ║
╚══════════════════════════════════════════════════════════════════════════════════════════╝

╔══════════════════════════════════════════════════════════╗
║               COMPARE 3 METHODE READ 0x03                ║
╠══════════════════════╦═══════════════════════════════════╣
║ SMBus byte           ║ 0x00        Only 1 byte           ║
║ SMBus word           ║ 0x8000      Byte order swap       ║
║ Regmap Custom        ║ 0x0080      Valid                 ║
╚══════════════════════╩═══════════════════════════════════╝
</pre>


### Final Result ([Final Log](./console-ramoops_DEV_Final_Log)).

Successfully resolved the TFA98XX driver integration by:

1. **Fixing incompatible I2C struct** - Adapted standard Linux `struct i2c_msg` (12 bytes) to 
   Mediatek's `struct mt_i2c_msg` (24 bytes) by properly initializing `timing` and `ext_flag` 
   fields, eliminating garbage values that caused "speed too fast" errors.

2. **Probing the chip** - Detected TFA9890 revision 0x80 at I2C address 0x35 on bus 0.

3. **Firmware installation** - Successfully loaded container firmware `tfa98xx.cnt` (4761 bytes),
   registered DSP instance (handle 0), and created 3 audio profiles:
   - MUSIC_44100 (44.1 kHz)
   - MUSIC_48000 (48 kHz)
   - VOICE_16000 (16 kHz)

Audio playback testing has not been performed yet. Further updates pending.
