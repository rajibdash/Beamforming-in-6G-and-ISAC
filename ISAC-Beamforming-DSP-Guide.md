# Intelligent beamforming and DSP for ISAC

## 1. Scope and terminology

**Integrated sensing and communication (ISAC)** uses radio signals, spectrum,
hardware, and/or processing jointly for communication and sensing. For
example, a base station can transmit a waveform to a user while a sensing
receiver analyzes reflections from objects. Sensing can provide information
such as direction, range, radial velocity, and environmental changes.

**Intelligent beamforming** adaptively selects transmit and receive spatial
weights to balance communication and sensing objectives. "Intelligent" can
mean model-based optimization, adaptive signal processing, or AI/ML-assisted
decisions; it does not necessarily mean AI.

This guide covers the discussion's 5G-Advanced and prospective 6G context,
using an OFDM-based gNB (base station) and UE (user equipment) as a concrete
example. It distinguishes existing NR mechanisms from possible ISAC
implementations. An ISAC architecture, sensing report, waveform, or scheduler
interface described here is **not automatically a standardized 3GPP feature**.
6G techniques discussed here include research directions, not a claim about a
finalized 6G specification.

### Abbreviations

| Term | Meaning |
| --- | --- |
| ADC / DAC | Analog-to-digital / digital-to-analog converter |
| CSI / CSI-RS | Channel state information / CSI reference signal |
| DM-RS | Demodulation reference signal |
| DSP / PHY / MAC | Digital signal processing / physical layer / medium access control |
| gNB / UE | NR base station / user equipment |
| ISAC | Integrated sensing and communication |
| MIMO / RF | Multiple-input multiple-output / radio frequency |
| OFDM / CP | Orthogonal frequency-division multiplexing / cyclic prefix |
| PDCCH / PDSCH | Physical downlink control / shared channel |
| PUCCH / PUSCH | Physical uplink control / shared channel |
| SRS / SSB | Sounding reference signal / synchronization signal block |
| TCI | Transmission configuration indicator |

## 2. Why beamforming matters to ISAC

An antenna array changes the relative phase and, where supported, amplitude
of signals at its elements. This shapes the radiation or receive sensitivity
pattern, increasing gain in desired directions and potentially reducing
interference elsewhere.

In ISAC, the pattern affects both the communication link and the energy
illuminating a target. A beam optimized only for user throughput may miss the
sensing region; a sensing-focused beam may reduce user rate or interfere with
other users. Beam design therefore balances objectives such as:

- Communication coverage, reliability, throughput, latency, and interference.
- Sensing detection, range/angle/velocity accuracy, coverage, and update rate.
- Transmit power, RF-chain count, hardware constraints, and processing latency.

The PHY implements the transmit and receive signal processing. A scheduler,
beam manager, or sensing controller supplies objectives, resources, or candidate
beam decisions. Their exact division is implementation-dependent.

## 3. Techniques and their trade-offs

| Technique | Principle | ISAC and steering trade-off |
| --- | --- | --- |
| Analog beamforming | RF phase shifters, and sometimes amplitude controls, steer signals before or after RF-chain conversion. | Relatively few RF chains and lower hardware cost, but limited independent beams and flexibility. |
| Digital beamforming | Complex baseband weights control signals across RF chains, often one per antenna. | Flexible multi-user and sensing patterns and receive processing, but more converters, power, data movement, and DSP. |
| Hybrid analog-digital | A digital precoder/combiner and an RF network jointly shape the array pattern. | Often a practical large-array, mmWave, or THz compromise; constrained RF connectivity and RF-chain count restrict patterns. |
| Codebook-based beamforming | Select from predefined beam or precoder candidates. | Simple signaling and search; finite candidates can limit accuracy and flexibility. |
| Adaptive / joint ISAC design | Update weights using communication channels, sensing estimates, and constraints. | Can balance both tasks, but needs reliable estimates and timely optimization. |
| Hierarchical search and tracking | Search broad beams, refine to narrower candidates, then track changes. | Reduces search overhead, but prediction errors or blockage can require a new sweep. |
| Near-field focusing | Focus spatial energy at a location, including range, rather than only a far-field angle. | Relevant to large arrays and short wavelengths; more complex calibration and focusing. |
| AI/ML-assisted selection | Learn or predict beams, targets, or resource decisions from observations. | May reduce search or optimization cost; robustness, training data, latency, and uncertainty matter. |

There is **no universally most efficient technique**. Hybrid beamforming with
adaptive or hierarchical tracking is often a useful compromise for directional
ISAC steering, especially with large mmWave/THz arrays. Fully digital may be
better when spatial flexibility is paramount; analog/codebook steering may
better suit a low-cost design. Efficiency can mean hardware cost, energy,
search overhead, rate, or sensing accuracy, and the answer changes accordingly.

Massive MIMO supplies large arrays and spatial degrees of freedom. Higher
frequencies can support compact arrays and wide bandwidths, but suffer
propagation, blockage, hardware, and calibration challenges. Wideband phase
shifters can exhibit **beam squint**; frequency-aware digital processing or
true-time-delay hardware may mitigate it.

## 4. End-to-end architecture

```text
                  SCHEDULER / BEAM MANAGER / ISAC CONTROLLER
     Inputs: UE reports, channels, sensing tracks, QoS, constraints
     Outputs: schedules, beam choices, power and sensing opportunities
                             |                    ^
                             v                    |
+-------------------------------------------------------------------------+
| gNB BASEBAND + RADIO + ANTENNA ARRAY                                    |
|                                                                         |
| DOWNLINK TX                                                             |
| Control/data/reference -> coding/modulation -> grid -> digital precoder |
| -> OFDM -> DAC/RF -> analog beamforming -> antenna array ---------------+---+
|                                                                         |   |
| UPLINK / SENSING RX                                                     |   |
| Antenna -> analog combining -> RF/ADC -> DSP                            |   |
|    -> communication decoding OR echo processing -> measurements         |   |
+----------------------------^--------------------------------------------+   |
                             |                                                |
                UE uplink ---+                    Downlink waveform ----------+
                             |                             |
                        +----+----+                +-------+-------+
                        | UE TX/RX|                | UE and target |
                        +---------+                +-------+-------+
                                                           |
                                              Target echo -+
                                                    to gNB RX
```

In **monostatic sensing**, the same site transmits and receives the echo. In
**bistatic or multistatic sensing**, transmitter and receiver are at different
locations; geometry, synchronization, and processing differ. The UE need not
perform sensing for the gNB to sense an echo.

Simultaneous transmit/receive is not guaranteed. A design may use separate
time intervals, separate antennas, or simultaneous operation with appropriate
duplexing, isolation, and interference management.

## 5. Who calculates the weights and how the scheduler uses them

1. The PHY estimates channels from reference signals and receives configured
   UE measurements. Sensing DSP can produce target and environment estimates.
2. A PHY beamforming algorithm, beam manager, or joint optimizer calculates
   weights or selects candidates. It may use channel estimates, target
   direction, uncertainty, interference, power, and hardware constraints.
3. The scheduler selects users, time-frequency resources, and transmission
   parameters, potentially considering beam compatibility and sensing needs.
4. The PHY constructs the waveform and applies the digital precoder. The radio
   applies the selected RF beam settings. The receiver similarly applies
   analog and digital combining.
5. New measurements and feedback drive subsequent updates.

The scheduler typically exchanges **summaries**, not an entire antenna-weight
matrix: channel quality, beam or precoder indices, available layers, predicted
interference, sensing priorities, and resource needs are examples. It sends
the schedule and beam-related configuration back to the PHY. The exact
interface, ownership, timing, and update rate are vendor-specific; there is no
single universal ISAC scheduler interface. Some implementations jointly
optimize scheduling and beamforming.

A feedback index or TCI state is not necessarily the actual physical antenna
weight matrix. TCI configuration and related procedures identify reference
signal relationships used by the UE; the internal antenna implementation and
specific channel associations depend on the applicable NR configuration.

## 6. Beamforming for control, data, and reference signals

Beamforming applies to physical signals carrying control, data, and reference
information. It does **not** imply one independent beam per logical channel.
Signals can share resources, beams, or beam states.

| Signal or channel | Purpose and representative beam behavior |
| --- | --- |
| SSB, including PSS, SSS, PBCH and PBCH DM-RS | Cell detection, synchronization, and broadcast information. A gNB may transmit SSBs in different directions for initial beam discovery. |
| PDCCH | Downlink control information. The UE monitors configured control resources with an applicable beam/reference relationship; the design prioritizes reliable control reception. |
| PDSCH | Downlink data. The gNB applies a user/layer precoder; associated DM-RS supports estimation of the effective channel for demodulation. |
| CSI-RS | Configured downlink channel and/or beam measurements; results can support beam selection and CSI feedback. |
| PUCCH / PUSCH | UE uplink control / data. The UE may use a transmit beam and the gNB may use receive combining. |
| SRS | UE uplink sounding used by the gNB for channel and/or beam measurements when configured. |
| PRACH | Random access; the gNB detects access occasions and can use receive beamforming where supported. |

SSB sweeping is useful for access, CSI-RS can refine measurements, and SRS
can support uplink channel estimation. In TDD, calibrated reciprocity can help
derive downlink beam information from uplink measurements; radio-chain
calibration and channel aging still matter. FDD generally cannot assume
instantaneous downlink/uplink channel reciprocity.

An implementation may reuse known communication signals for sensing or
schedule a sensing-oriented transmission. It must coordinate this with
existing allocations and control reliability. Not every control or reference
signal necessarily provides useful sensing performance.

## 7. Detailed gNB transmit chain

```text
 Higher layers / scheduler / optional ISAC controller
  data, control, resources, MCS, beam configuration, sensing requirements
                                  |
                                  v
+-----------------------------------------------------------------------+
| Channel-specific processing                                           |
| PDSCH: transport block -> CRC -> LDPC -> rate matching -> scrambling  |
|        -> modulation -> layer mapping                                 |
| PDCCH: control information -> CRC -> polar coding -> rate matching    |
|        -> scrambling -> modulation                                    |
| SSB: synchronization signals and PBCH, with associated reference      |
| References: generate configured DM-RS, CSI-RS, etc.                   |
+---------------------------------+-------------------------------------+
                                  v
+-----------------------------------------------------------------------+
| Resource-grid construction                                            |
| Map channel symbols and reference signals to configured time-frequency|
| resources; retain the transmitted reference and beam state for sensing|
+---------------------------------+-------------------------------------+
                                  v
+-----------------------------------------------------------------------+
| Digital precoding: layer symbols -> digital RF-chain streams          |
| Weights selected from channel, scheduling, sensing and hardware inputs|
+---------------------------------+-------------------------------------+
                                  v
 Per RF chain: IFFT -> CP -> digital waveform conditioning -> DAC
                                  |
                                  v
 Filtering -> RF upconversion -> analog distribution/beamforming network
                                  |
                                  v
                 Power amplification -> antenna elements
                                  |
                     Radiated signal to UE and target
```

PDSCH and PDCCH have different channel-specific processing; PBCH and reference
signals also have their own rules. A logical antenna port is not necessarily a
physical antenna element or RF chain. The diagram is conceptual: actual RF
distribution, phase shifter, PA, filtering, and calibration placement varies.
Waveform conditioning can include filtering, resampling, crest-factor
reduction, and digital predistortion, depending on the radio.

### 7.1 Resource grid

The OFDM resource grid organizes signals by subcarrier, OFDM symbol, and
logical spatial dimension:

```text
Frequency
    ^
    |  D  D  D  D  D  D      D: scheduled data
    |  D  R  D  D  R  D      R: reference signal
    |  C  C  C  -  -  D      C: control
    |  D  R  D  D  R  D      -: unused/other allocation
    +---------------------> Time
```

This is illustrative, not an NR mapping. The scheduler and PHY must respect
the actual channel rules and resource configuration. ISAC may use known
symbols/resources as sensing references or allocate sensing-specific
opportunities. Bandwidth, time sampling, beam state, and transmitted power
all influence sensing performance.

### 7.2 Digital and analog precoding

Let `L` be the number of input streams, `N_RF` the number of RF chains, and
`N_ant` the number of antenna elements. For subcarrier `k` and OFDM symbol
`m`, a simplified hybrid transmit model is:

```text
s[k,m]       : L x 1 input-symbol vector
W_BB[k,m]    : N_RF x L digital precoder
W_RF         : N_ant x N_RF analog network

x_RF[k,m]  = W_BB[k,m] s[k,m]
x_ant[k,m] ≈ W_RF x_RF[k,m]
```

`W_BB` contains complex digital coefficients. `W_RF` represents the RF
network's phase and possibly amplitude controls. The model is a per-subcarrier
abstraction; wideband hardware, calibration, connectivity, and RF
nonlinearities can make the actual response frequency- and time-dependent.

```text
 L streams                 N_RF chains                 N_ant elements
 s1 --+                    +-- chain 1 --+             +-- antenna 1
 s2 --+--> digital W_BB ---+-- chain 2 --+-- RF W_RF --+-- antenna 2
 ... -+                    +-- ... -------+             +-- ...
```

In an ISAC-aware design, weights may aim to maintain a communication
signal-to-interference-and-noise ratio while illuminating a target direction.
If the directions differ significantly, the system might use multiple beams
or separate time slots, subject to RF-chain, power, and scheduling constraints.

### 7.3 OFDM and radio

The IFFT converts frequency-domain subcarriers to time-domain samples. The CP
helps OFDM cope with multipath under suitable conditions. DACs convert digital
samples to analog; the RF chain filters and upconverts them to the carrier.
Analog phase/amplitude controls shape the array signal before radiation.

The CP is not by itself a universal sensing-range limit: usable range and
ambiguity also depend on timing, waveform, receiver architecture, observation
window, and processing.

## 8. UE receive and transmit chains

```text
DOWNLINK UE RX
 UE antenna(s) -> optional analog beam -> RF/LNA/filter -> ADC
 -> timing/frequency correction -> CP removal -> FFT
 -> reference-signal-based effective-channel estimate
 -> digital combining/equalization -> demodulation -> decoding -> CRC
 -> control information / user data

UPLINK UE TX
 Uplink control/data + configured references -> channel-specific coding
 -> modulation / resource mapping -> optional digital precoding
 -> waveform generation -> DAC / RF -> optional analog beam -> antenna(s)
```

The UE estimates the **effective channel** including the propagation channel
and applicable transmit/receive beams; it need not know all gNB element
weights. It can report configured beam measurements or CSI, including
channel quality, rank, or precoder recommendations as applicable. Feedback is
delayed and quantized, so it may not represent the current channel perfectly.

Uplink processing is channel-specific. For example, PUSCH can use configured
CP-OFDM or transform-precoded transmission; the generic diagram should not be
read as prescribing one waveform for every uplink channel.

## 9. Detailed gNB uplink and sensing receive chain

```text
 UE uplink + target echoes + interference + possible TX leakage
                              |
                              v
                       Receive antennas
                              |
             TX/RX separation, protection and filtering
                              |
             LNA / analog receive combining / RF chains
                              |
                 Downconversion -> ADC per RF chain
                              |
          Digital calibration and timing/frequency correction
                              |
                  CP removal and FFT where applicable
                              |
              +---------------+----------------+
              |                                |
              v                                v
    COMMUNICATION PROCESSING            SENSING PROCESSING
    reference-based channel estimate    transmit-reference alignment
    digital combining/equalization      echo extraction / cancellation
    demodulation and decoding           delay, Doppler, angle estimation
    CRC and higher-layer delivery       detection and tracking
              |                                |
              +---------------+----------------+
                              v
                measurements for future decisions
```

The actual order of analog combining, amplification, filtering, and
downconversion depends on the hardware. Analog combining can discard spatial
information before ADC, so receive beams must preserve the information needed
for communication and sensing.

A simplified frequency-domain model for analog combining is
`z[k,m] = W_RF,RX^H y[k,m]`, where `y` is the antenna-element signal,
`W_RF,RX` is the receive RF-combining matrix, and `H` denotes conjugate
transpose. Digital combining can then process the sampled RF-chain signals,
often after the FFT. This representation abstracts the actual RF hardware.

For communication, the PHY synchronizes, extracts allocated resources,
estimates the channel from appropriate references, combines/equalizes,
demodulates, decodes, checks CRC, and delivers the result. PRACH, PUCCH,
PUSCH, and SRS have different processing rules.

## 10. Sensing DSP: from waveform to target information

```text
Known transmitted symbols/waveform, timing, beam and calibration
                              |
                              v
Received samples -> alignment -> leakage/clutter handling -> echo extraction
                              |
                              v
            delay/range + Doppler/motion + angle processing
                              |
                              v
                  detection -> association -> tracking
                              |
                              v
          target estimates, uncertainty and environment information
```

These stages can be reordered or combined, and need not be implemented as
three independent estimators. Performance depends on waveform, geometry,
sampling, reference knowledge, and hardware.

### 10.1 Reference reconstruction and echo model

The sensing receiver needs a known or reconstructable transmit reference:
symbols or sequences, resource allocation, timing, beam/precoder state, and
relevant RF calibration. A gNB can know its own transmitted data, but the
actual waveform can still differ from the ideal because of RF distortion or
calibration error. Unknown data or ambiguous simultaneous waveforms require
appropriate separation or estimation; they are not automatically usable.

A simplified single-channel model is:

```text
y(t) = h_leak(t) * x(t)
     + sum_i alpha_i x(t - tau_i) exp(j 2 pi f_D,i t)
     + clutter(t) + noise_and_interference(t)
```

Here `x` is the transmit reference, `*` is convolution, `tau_i` is an echo
delay, `f_D,i` is Doppler, and `alpha_i` is a reflection coefficient. A real
multi-antenna system also includes propagation, array response, beam weights,
and RF effects.

### 10.2 Leakage, clutter, and dynamic range

The direct transmit signal and static reflections can be far stronger than a
target echo. Depending on the architecture, mitigation can include antenna
isolation, duplexing, RF cancellation, digital subtraction, filtering, and
clutter suppression. Digital cancellation cannot recover an echo if leakage
has already saturated the RF front end or ADC. Suppressing static clutter can
also remove slow or stationary targets; design depends on sensing goals.

### 10.3 Delay and range

The receiver compares the echo against the transmit reference, for example
using matched filtering or suitable OFDM-domain processing. For a monostatic
system, round-trip delay `tau` gives approximately:

```text
R = c tau / 2
range resolution ≈ c / (2 B)
```

`c` is the speed of light and `B` is effective sensing bandwidth. For
100 MHz effective bandwidth, the nominal resolution is approximately 1.5 m.
Resolution is the ability to distinguish nearby targets, not a guarantee of
1.5 m individual measurement error. Actual performance depends on occupied
resources, SNR, processing, windowing, and multipath. Bistatic range uses
the transmitter-target-receiver path geometry instead of this simple formula.

### 10.4 Doppler and radial velocity

Phase changes across observations can reveal Doppler. For a monostatic
approximation:

```text
v_radial ≈ lambda f_D / 2
```

`lambda` is wavelength and `f_D` is Doppler frequency. The sign depends on
the convention used. Coherent observation time, sampling pattern, oscillator
stability, and target motion affect Doppler resolution and ambiguity. In
bistatic sensing, the velocity relationship depends on geometry.

### 10.5 Direction and spatial estimation

Relative phases and amplitudes across receive channels, or measurements from
multiple beams, can estimate direction. Possible approaches include codebook
scanning, digital beam scanning, and model-based array estimation such as
MUSIC. Accuracy depends on array geometry, calibration, SNR, multipath, and
the spatial information preserved by analog combining. Near-field processing
may jointly account for direction and distance.

### 10.6 Detection, tracking, and new information

The processor can detect candidate reflections, associate them across time,
estimate position/velocity, quantify uncertainty, and predict future states.
A conceptual output is:

```text
Track:             track-7
Direction:         25 degrees
Range:             40 m
Radial velocity:   -3 m/s
Uncertainty:       direction/range/velocity uncertainty
Timestamp:         measurement time
```

These values and fields are illustrative, not a standardized report. A
detected object is not automatically a particular UE, and radial velocity is
not the complete 3D velocity.

## 11. Closed-loop ISAC control

```text
 MEASURE
 UE beam/CSI feedback + uplink channel estimates + sensing observations
                                  |
                                  v
 ESTIMATE AND PREDICT
 channels, targets, motion, blockage/reflectors where inferable, uncertainty
                                  |
                                  v
 OPTIMIZE AND SCHEDULE
 select users, resources, candidate beams, power and sensing opportunities
                                  |
                                  v
 APPLY
 configure PHY digital precoders/combiners and radio TX/RX beam settings
                                  |
                                  v
 TRANSMIT / RECEIVE / MEASURE AGAIN
```

Potential new ISAC inputs include target direction, range, motion, prediction
uncertainty, sensing priority, and possible environmental reflectors or
blockers. They can support beam selection, tracking, or sensing scheduling.
Communication channel estimates and target estimates are related but not
automatically interchangeable: calibration, geometry, delay, and multipath
matter. If confidence is low, the system may need new measurements or a
beam sweep rather than trusting a prediction.

### Example: a moving UE and a separate object

1. The gNB transmits to a UE and, if its waveform and beam are suitable,
   illuminates a sensing region.
2. The UE measures its effective channel and reports configured information.
3. The gNB estimates an object's direction and motion from echoes.
4. A tracker predicts the object's position and uncertainty.
5. The controller proposes a future sensing beam or sensing resources.
6. The scheduler accounts for communication QoS, control reliability, and
   resource conflicts; the PHY applies the chosen beam configuration.

If UE and target directions differ, sensing may require another beam or time
allocation. An implementation can choose a reliable control beam and narrower
data/sensing beams where appropriate, but no universal rule requires a
different or wider beam for each channel.

## 12. Practical limitations and design checks

- **Latency and mobility:** channel/target estimates age while feedback,
  scheduling, optimization, and radio reconfiguration take time.
- **Calibration and synchronization:** array phase, RF-chain response, clock
  offset, and timing errors affect both beam accuracy and sensing. Distributed
  sensing adds inter-node synchronization and geometry requirements.
- **Ambiguity and interference:** sparse or irregular time-frequency
  resources, multipath, other users, and other cells can create ambiguities
  or false detections. Check waveform-dependent unambiguous range and Doppler.
- **Hardware limits:** finite phase resolution, RF-chain count, PA
  nonlinearity, ADC dynamic range, beam-switch time, and power constraints
  restrict feasible designs.
- **Safety and privacy:** sensing deployment must account for applicable
  radio, privacy, and data-handling requirements.
- **Evaluation:** measure communication rate/reliability and sensing
  detection, accuracy, update latency, and uncertainty together; no single
  beamforming metric captures the full trade-off.

## 13. Further reading

- [3GPP TS 38.211: NR physical channels and modulation](https://www.3gpp.org/DynaReport/38211.htm).
- [3GPP TS 38.212: NR multiplexing and channel coding](https://www.3gpp.org/DynaReport/38212.htm).
- [3GPP TS 38.213: NR physical layer procedures for control](https://www.3gpp.org/DynaReport/38213.htm).
- [3GPP TS 38.214: NR physical layer procedures for data](https://www.3gpp.org/DynaReport/38214.htm).
- [3GPP TS 38.331: NR radio resource control](https://www.3gpp.org/DynaReport/38331.htm).
- The ISAC and beamforming papers already included in this repository.
