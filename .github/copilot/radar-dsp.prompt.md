# Radar DSP Pipeline — CM55 Helium/MVE Acceleration

Patterns for building radar signal processing on PSOC Edge E84 using the Cortex-M55 with Arm Helium (MVE) SIMD.

> **Applies to:** PSOC Edge E84 with BGT60TR13C 60 GHz radar sensor (KIT_PSE84_AI).

---

## Architecture Overview

```
BGT60TR13C → SPI → CM55 DSP Pipeline → IPC → CM33 → WiFi/MQTT
                    ├── Range FFT
                    ├── Doppler FFT (if micro-motion)
                    ├── MTI Filter
                    └── Detection Logic
```

The CM55 handles all compute-intensive DSP at ~10 Hz frame rate. Detection results (state changes) are sent to CM33 via IPC for network publishing. See `#dual-core-setup` for the CM55 project configuration and `#ipc-patterns` for communication.

---

## CM55 Makefile Configuration

```makefile
# Enable Helium/MVE SIMD acceleration
CFLAGS+=-march=armv8.1-m.main+mve.fp+fp.dp
CFLAGS+=-mfloat-abi=hard

# FreeRTOS must save MVE registers on context switch
DEFINES+=configENABLE_MVE=1

# Optimization for DSP workloads
CFLAGS+=-O2
```

> **Toolchain note:** GCC_ARM (default) supports Helium intrinsics. LLVM may provide better auto-vectorization for some DSP patterns. Both require the MVE arch flag.

---

## Range FFT Pattern

Processes raw ADC samples from each radar chirp into range bins:

```c
#include "arm_math.h"  /* CMSIS-DSP with Helium support */

#define NUM_SAMPLES_PER_CHIRP  64
#define FFT_SIZE               64

static arm_rfft_fast_instance_f32 rfft_instance;
static float32_t fft_input[FFT_SIZE];
static float32_t fft_output[FFT_SIZE];
static float32_t magnitude[FFT_SIZE / 2];

void range_fft_init(void)
{
    arm_rfft_fast_init_f32(&rfft_instance, FFT_SIZE);
}

void range_fft_process(const float32_t *adc_samples, float32_t *range_profile)
{
    /* Copy raw samples — FFT operates in-place */
    memcpy(fft_input, adc_samples, NUM_SAMPLES_PER_CHIRP * sizeof(float32_t));

    /* Real FFT — Helium-accelerated when compiled with MVE flags */
    arm_rfft_fast_f32(&rfft_instance, fft_input, fft_output, 0);

    /* Compute magnitude of complex FFT output */
    arm_cmplx_mag_f32(fft_output, magnitude, FFT_SIZE / 2);

    /* Output range profile (magnitude per range bin) */
    memcpy(range_profile, magnitude, (FFT_SIZE / 2) * sizeof(float32_t));
}
```

---

## MTI (Moving Target Indication) Filter

Removes static clutter (walls, furniture) to isolate moving targets:

```c
#define NUM_RANGE_BINS  32

static float32_t mti_history[NUM_RANGE_BINS];
static bool mti_initialized = false;

/* Single-delay MTI: subtract previous frame from current */
void mti_filter(const float32_t *current_frame, float32_t *filtered, uint32_t num_bins)
{
    if (!mti_initialized)
    {
        /* First frame — no history to subtract, output zeros */
        memset(filtered, 0, num_bins * sizeof(float32_t));
        memcpy(mti_history, current_frame, num_bins * sizeof(float32_t));
        mti_initialized = true;
        return;
    }

    /* Helium-accelerated vector subtraction */
    arm_sub_f32(current_frame, mti_history, filtered, num_bins);

    /* Take absolute value — we care about magnitude of change */
    arm_abs_f32(filtered, filtered, num_bins);

    /* Update history for next frame */
    memcpy(mti_history, current_frame, num_bins * sizeof(float32_t));
}
```

---

## Presence Detection with Threshold

```c
typedef enum {
    PRESENCE_ABSENT = 0,
    PRESENCE_MICRO,     /* Breathing, small gestures */
    PRESENCE_MACRO,     /* Walking, large movement */
} presence_state_t;

/* Thresholds — tune for your environment and range */
#define MACRO_THRESHOLD  2.0f   /* Large motion energy threshold */
#define MICRO_THRESHOLD  0.3f   /* Small motion energy threshold */

presence_state_t detect_presence(const float32_t *mti_output, uint32_t num_bins)
{
    float32_t max_energy;
    uint32_t max_index;

    /* Find peak energy across range bins */
    arm_max_f32(mti_output, num_bins, &max_energy, &max_index);

    if (max_energy > MACRO_THRESHOLD)
        return PRESENCE_MACRO;
    else if (max_energy > MICRO_THRESHOLD)
        return PRESENCE_MICRO;
    else
        return PRESENCE_ABSENT;
}
```

---

## State Machine with Holdoff

Debounces detection to prevent rapid state toggling. Send IPC messages only on state transitions:

```c
#define HOLDOFF_FRAMES  30  /* ~3 seconds at 10 Hz frame rate */

static presence_state_t current_state = PRESENCE_ABSENT;
static uint32_t holdoff_counter = 0;

bool process_detection(presence_state_t detected, presence_state_t *new_state)
{
    if (detected == current_state)
    {
        holdoff_counter = 0;
        *new_state = current_state;
        return false;  /* No state change */
    }

    holdoff_counter++;
    if (holdoff_counter >= HOLDOFF_FRAMES)
    {
        *new_state = detected;
        current_state = detected;
        holdoff_counter = 0;
        return true;  /* State changed — send IPC message */
    }

    *new_state = current_state;
    return false;  /* Still in holdoff period */
}
```

---

## Doppler FFT (Micro-Motion Detection)

For fine-grained motion classification, apply a second FFT across multiple chirps (slow-time axis):

```c
#define NUM_CHIRPS_PER_FRAME  16
#define DOPPLER_FFT_SIZE      16

static arm_rfft_fast_instance_f32 doppler_rfft;
static float32_t doppler_input[DOPPLER_FFT_SIZE];
static float32_t doppler_output[DOPPLER_FFT_SIZE];

void doppler_fft_init(void)
{
    arm_rfft_fast_init_f32(&doppler_rfft, DOPPLER_FFT_SIZE);
}

/* Call for each range bin of interest */
void doppler_fft_process(
    const float32_t range_profiles[NUM_CHIRPS_PER_FRAME],
    float32_t *doppler_spectrum)
{
    /* range_profiles contains one range bin value per chirp */
    memcpy(doppler_input, range_profiles, DOPPLER_FFT_SIZE * sizeof(float32_t));

    arm_rfft_fast_f32(&doppler_rfft, doppler_input, doppler_output, 0);
    arm_cmplx_mag_f32(doppler_output, doppler_spectrum, DOPPLER_FFT_SIZE / 2);
}
```

---

## Performance Notes

| Operation | CM55 (Helium) | CM55 (scalar) | Speedup |
|-----------|--------------|---------------|---------|
| 64-pt Real FFT | ~3 µs | ~15 µs | ~5× |
| 32-bin MTI filter | ~0.5 µs | ~2 µs | ~4× |
| Vector magnitude (32 complex) | ~0.8 µs | ~4 µs | ~5× |

Helium acceleration comes from CMSIS-DSP auto-vectorization — no explicit intrinsics needed if compiled with `-march=armv8.1-m.main+mve.fp`.

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Missing `-march=armv8.1-m.main+mve.fp` | Helium intrinsics not used, slow execution | Add to CM55 CFLAGS |
| Missing `configENABLE_MVE=1` | Silent data corruption in DSP tasks | Add to FreeRTOSConfig.h |
| FFT on every frame without holdoff | IPC queue overflow, CM55 stalls | Send IPC only on state transitions |
| MTI filter not initialized | First frame produces garbage output | Handle first-frame case explicitly |
| Sharing FFT buffers between tasks | Race condition, corrupted spectra | Each task gets its own FFT instance |
