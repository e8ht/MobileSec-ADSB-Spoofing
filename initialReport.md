
# ADS-B Spoofing and Anomaly Detection: Project Report

## 1. Introduction

The Automatic Dependent Surveillance–Broadcast (ADS-B) protocol is the backbone of modern air traffic surveillance, used globally by commercial, private, and military aircraft to continuously broadcast their position, altitude, velocity, and identity. Despite its central role in aviation safety, ADS-B was designed without any form of authentication or encryption. Any message transmitted by an aircraft can be received, decoded, and in principle fabricated by anyone with inexpensive, commercially available Software Defined Radio (SDR) hardware. This project investigates that vulnerability directly: we built a complete receive pipeline capable of capturing and decoding real ADS-B traffic, and we use that pipeline as the foundation for a planned spoofing-injection and anomaly-detection system.

The objective of this project is twofold. First, to demonstrate, hands-on, that ADS-B's lack of authentication is not merely a theoretical weakness but a practically exploitable one, using only consumer-grade hardware. Second, to build a lightweight, rule-based detector capable of distinguishing legitimate aircraft broadcasts from fabricated or anomalous ones, drawing on detection strategies described in published academic research. The deliverable for this project is a live demonstration: real aircraft received and decoded in real time, a spoofed/injected message introduced into the data stream, and a working detector that correctly flags the anomaly, supported by a written report describing the protocol's security weaknesses, our methodology, and our results.

This project was conducted as part of a mobile security course, and its scope was deliberately designed around hardware the team already owned (a HackRF One SDR with a PortaPack H4M standalone interface and a dedicated 1090MHz antenna), the explicit requirement for a hands-on demonstration rather than a literature review, and the practical constraint that only one team member had physical access to the RF hardware, with the remainder of the team distributed internationally and contributing on the software/analysis side.



## 2. Background

### 2.1 The ADS-B Protocol

ADS-B operates primarily on 1090MHz using a scheme called the Extended Squitter, in which aircraft periodically and autonomously broadcast short digital messages without being queried by a ground station. These broadcasts are classified by a 5-bit Downlink Format (DF) field at the start of each message. Long messages (112 bits, 28 hex characters) with DF value 17 carry Extended Squitter ADS-B data: the message type of primary interest to this project, since it contains an aircraft's identity (ICAO 24-bit address and callsign), airborne position (latitude/longitude via Compact Position Reporting, or CPR, encoding), velocity, and altitude. Shorter 56-bit messages (DF values such as 0, 4, 5, and 11) correspond to older Mode S surveillance replies such as altitude and identity replies, and were also observed in our captures but are of secondary interest here.

Because ADS-B messages are unencrypted, unauthenticated, and conform to a fully public specification, any party with an SDR capable of tuning to 1090MHz can both receive and, if they choose to transmit, fabricate these messages. This is the central security weakness that motivates our project.

### 2.2 Related Work

The vulnerability of ADS-B to spoofing is well documented in the academic literature, and our detection approach draws directly on this prior work. Luo and Chan's SODA system uses a two-stage Deep Neural Network to classify incoming messages and their transmitting aircraft based on physical-layer features such as IQ samples and signal phase, achieving a 99.34% detection probability for ground-based spoofing attacks with a false alarm rate of only 0.43%. Jansen et al. take a complementary, non-invasive approach in their NDSS paper, using crowdsourced data from geographically distributed ground sensors to cross-validate aircraft reports and detect GPS spoofing, ADS-B spoofing, and Sybil attacks through consistency checks rather than deep signal analysis. Pirolley et al. similarly apply a deep-learning classifier focused specifically on low-altitude traffic anomaly detection, achieving a 96.2% success rate in identifying spoofed aircraft around Toulouse.

Our project does not attempt to replicate the deep-learning architectures used in this prior work, which require large labeled training datasets and specialized infrastructure beyond the scope of a single-semester, single-hardware-owner project. Instead, we adopt the same conceptual foundation that spoofed or anomalous ADS-B messages can be identified by inconsistencies in their content, independent of any cryptographic verification, and implement it as a lightweight, rule-based detector operating on decoded message fields (position jumps, duplicate identifiers, and implausible kinematics) rather than raw physical-layer signal features.



## 3. Methodology

### 3.1 Hardware and Initial Validation

The receive hardware consists of a HackRF One SDR, a PortaPack H4M add-on board running the open-source Mayhem firmware (providing a standalone touchscreen interface), and a dedicated 1090MHz antenna connected via an SMA male-to-male adapter. Initial hardware validation was performed in two stages. First, the device was powered on and tested standalone, using the PortaPack's built-in ADS-B receiver application, which immediately decoded live aircraft (e.g., flight CCA471, and subsequently N7799T at 20,950ft, heading 069, over Bangkok) directly on the device screen, confirming that the antenna, HackRF, and PortaPack firmware were all functioning correctly before any computer-side software was introduced.

Second, the HackRF was connected to a laptop running a Kali Linux virtual machine for computer-based capture and analysis. This required switching the PortaPack out of its standalone Mayhem mode into native HackRF USB mode, after which the device was correctly enumerated by the host:


```
sudo apt install hackrf -y
```

<img width="862" height="342" alt="image" src="https://github.com/user-attachments/assets/475c2d8e-ce55-4252-902c-6d8ca1329a88" />


```
lsusb
```
//Bus 001 Device 003: ID 1d50:6089 OpenMoko, Inc. Great Scott Gadgets HackRF One SDR

<img width="621" height="135" alt="image" src="https://github.com/user-attachments/assets/2668eb79-f382-4903-9f44-25c590197a1f" />

```
hackrf_info
```

<img width="616" height="238" alt="image" src="https://github.com/user-attachments/assets/e58b8785-e9c0-4e21-93e1-d55166d168fa" />

//Found HackRF, Board ID: 2 (HackRF One), Firmware Version: v2.3.2


### 3.2 Software Pipeline Setup

We selected `dump1090` (FlightAware fork) as our decoding software. Because this software natively supports RTL-SDR dongles but not HackRF directly, HackRF support was added via SoapySDR, a hardware abstraction layer:

```
sudo apt install git build-essential libusb-1.0-0-dev -y
git clone https://github.com/flightaware/dump1090.git
cd dump1090
```
<img width="841" height="344" alt="image" src="https://github.com/user-attachments/assets/1f826d2c-5add-489f-ae46-6d4b900bcb8a" />



```
sudo apt install libsoapysdr-dev soapysdr-module-hackrf soapysdr-tools -y
```
<img width="857" height="252" alt="image" src="https://github.com/user-attachments/assets/3b13d569-c7be-4e0c-9ab1-de7c9bec8ad2" />

```
SoapySDRUtil --find
```

<img width="465" height="221" alt="image" src="https://github.com/user-attachments/assets/aed449b8-f3d5-4544-93b8-3718b1b2cfb5" />

// confirms HackRF is visible to SoapySDR


```
make clean
make SOAPYSDR=yes LDLIBS="-lSoapySDR"   # LDLIBS (not LIBS) was required to resolve a
                                          # linker ordering error with ld.bfd
```
<img width="926" height="233" alt="image" src="https://github.com/user-attachments/assets/913e3782-d752-412c-a288-726540bbe095" />

Two non-obvious issues were encountered and resolved during this step, both worth noting as part of our hands-on methodology: first, the standard `make SOAPYSDR=yes` invocation produced `undefined reference` linker errors despite `pkg-config` correctly reporting the SoapySDR library which was resolved by passing the link flag via the `LDLIBS` variable rather than `LIBS`, respecting `ld.bfd`'s requirement that libraries be specified after the object files that reference them. Second, the `--device-type` flag required for HackRF operation is `soapy`, not the more intuitively named `soapysdr`.



### 3.3 Data Collection

With the pipeline functional, live ADS-B traffic was captured in interactive mode to confirm correct decoding:

```
./dump1090 --device-type soapy --device "driver=hackrf" --interactive
```

A baseline dataset of real aircraft traffic was then logged to disk for offline analysis:

```
./dump1090 --device-type soapy --device "driver=hackrf" --raw > baseline_capture.txt
```
<img width="935" height="301" alt="image" src="https://github.com/user-attachments/assets/76b39723-abe8-4dd5-ba8d-cc0ccd79d50a" />


A single capture session over Bangkok yielded 2,331 raw message lines, each in the standard `*<hex>;` format. Message length was used to distinguish message types: 28-hex-character (112-bit) messages correspond to long Mode S frames, including DF17 Extended Squitter ADS-B messages, while 14-hex-character (56-bit) messages correspond to shorter surveillance replies (e.g., DF0, DF4, DF5, DF11) that lack position/identity payloads and were excluded from further analysis.

<img width="533" height="63" alt="image" src="https://github.com/user-attachments/assets/11700baa-9dc9-4c43-b19a-6431e80f7ffd" />





### 3.4 Message Decoding

Raw hex messages were decoded using the `pyModeS` library, which parses Mode S/ADS-B frames into structured fields (Downlink Format, ICAO address, ADS-B type code, position, velocity, and identification data) rather than requiring the team to hand-implement the CPR position-decoding algorithm and checksum validation:

Fire up python virtual environment and install pyModeS

```
python3 -m venv 1090-capture

source 1090-capture/bin/activate

pip install pyModeS

```

<img width="460" height="212" alt="image" src="https://github.com/user-attachments/assets/bcff5804-c461-4734-a78c-d53dd85a5ece" />



fire up python interactive shell
```
python3
```

```python
import pyModeS as pms

with open("baseline_capture.txt") as f:
    for line in f:
        msg = line.strip().strip("*;")
        if len(msg) == 28:
            try:
            # pyModeS 3 returns a dictionary-like object with all decoded fields
                res = pms.decode(msg)
                if res.get("df") == 17:
                    icao = res.get("icao")
                    tc = res.get("typecode")
                    print(icao, tc, msg)
            except Exception:
                # Skip messages that fail decoding/checksum validation
                continue  # skip messages that fail checksum/decoding validation
```

<img width="775" height="503" alt="image" src="https://github.com/user-attachments/assets/0c30e091-0557-487b-a133-fe0744ae689e" />



This confirmed successful decoding of real DF17 messages from multiple distinct aircraft (identified by ICAO addresses such as `8832B5` and `880852`), with ADS-B type codes spanning identification (type code 4), surface/airborne position (type codes 11 and 19 observed; type codes 9–18 denote airborne position with barometric altitude, 5–8 surface position), and airborne velocity (type code 19) messages, consistent with expected real-world traffic composition.




### 3.5 Planned Next Steps

Building on this validated baseline, the remaining methodology consists of three stages. 
First, construction of a synthetic message injector that generates fabricated DF17 messages, duplicate ICAO addresses, physically implausible position jumps, and inconsistent kinematic data -- and interleaves them into the decoded message stream at the software level, avoiding any actual RF transmission of fabricated aircraft data for safety and regulatory reasons. 
Second, development of a rule-based detector, informed by the feature classes used in the SODA and related detection literature, that flags messages violating physical plausibility constraints (maximum realistic speed/altitude change between consecutive reports from the same aircraft) or logical consistency constraints (a single ICAO address reporting two incompatible positions within an implausibly short time window). 
Third, validation of the detector against both the real baseline dataset, to confirm a low false-positive rate on legitimate traffic, and the synthetic injection dataset, to confirm correct identification of fabricated aircraft, forming the live demonstration component of this project.





## References

1. Luo, J. & Chan, K. Detecting ADS-B Spoofing Attacks Using Deep Neural Networks. IEEE CNS 2019. https://arxiv.org/pdf/1904.09969
2. Jansen, K., Niu, L., Xue, N., Martinovic, I., & Pöpper, C. Trust the Crowd: Wireless Witnessing to Detect Attacks on ADS-B-Based Air-Traffic Surveillance. NDSS. https://www.ndss-symposium.org/ndss-paper/trust-the-crowd-wireless-witnessing-to-detect-attacks-on-ads-b-based-air-traffic-surveillance
3. Pirolley, M., Couturier, R., Salomon, M., & Ambert, F. (2023). ADS-B Anomaly Detection in the Surveillance of Low-Altitude Aircrafts. Journal of Open Aviation Science, Vol. 1. https://journals.open.tudelft.nl/joas/article/download/7200/5891/27323
