# Cospas-Sarsat C/S T.001 & T.018 Distress Beacon Technical Architecture

## 1\. Executive Summary & Regulatory Framework

The International Cospas-Sarsat Programme is a satellite-aided search and rescue (SAR) initiative established by Canada, France, the United States, and the former Soviet Union. It detects and locates distress signals emitted by emergency beacons operating in the 406.0–406.1 MHz frequency band.

### Key Architectural Concepts:

1. **First Generation Beacon (FGB) Architecture (C/S T.001)**:  
   - Burst Transmission: Transmits 440 ms (Short Message 112 bits) or 520 ms (Long Message 144 bits) bursts every 50 seconds (\$\\pm 5\\text{ s}\$).  
   - **Bit Synchronization**: 15 bits of unmodulated carrier preamble (`111111111111111`).  
   - **Frame Synchronization**: 9 bits (`000101111` for normal distress, `011010000` for self-test frames).  
   - **Format Flag**: Bit 25 (0 \= 112-bit Short, 1 \= 144-bit Long).  
   - **15-Hex UIN**: Derived from bits 26 through 85 (60 bits \= 15 hexadecimal digits). Identifies the aircraft tail number, ship MMSI, or unique beacon serial number registered in national SAR databases (e.g. NOAA RGDB).  
2. **Dual BCH Error-Correcting Code Generation**:  
   - **BCH-1 (21 Parity Bits)**: Protects bits 25 through 85 using generator polynomial: \$\$g\_1(x) \= x^{21} \+ x^{20} \+ x^{18} \+ x^{16} \+ x^{14} \+ x^{13} \+ x^{11} \+ x^6 \+ x^5 \+ x^3 \+ x \+ 1 \\quad (0x13180B)\$\$  
   - **BCH-2 (12 Parity Bits)**: Protects supplementary GNSS location bits 107 through 132 using generator polynomial: \$\$g\_2(x) \= x^{12} \+ x^{10} \+ x^8 \+ x^5 \+ x^4 \+ x^3 \+ 1 \\quad (0x1539)\$\$  
3. **Second Generation Beacon (SGB) Evolution (C/S T.018)**:  
   - 250-bit variable payload transmissions using Direct Sequence Spread Spectrum (DSSS) and Offset QPSK (OQPSK) modulation.  
   - Return Link Service (RLS): Receives confirmation acknowledgments back from Galileo satellites to confirm rescue coordination.  
4. **Regulatory Mandates**:  
   - Aviation: ICAO GADSS Autonomous Distress Tracking (ADT) / FAA TSO-C126c ELTs.  
   - Maritime: IMO SOLAS Chapter IV mandatory EPIRB installations.