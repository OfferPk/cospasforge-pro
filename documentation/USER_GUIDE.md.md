# COSPASForge Pro User Guide & Developer Manual

## 1\. Quickstart Guide

1. Launch `index.html` in your web browser.  
2. In the "Emergency Beacon Transmitter Config" panel, select your Beacon Application Type (e.g. `ELT - Aviation ICAO GADSS`).  
3. Select your national administration Country Code (e.g. `366 - United States` or `470 - UAE`).  
4. Set the Beacon Serial Number (e.g. `1042`).  
5. Enter the distress GNSS coordinates: Latitude (`25.2048° N`) and Longitude (`55.2708° E`).  
6. Toggle "Self-Test Transmission Frame" to evaluate the test synchronization word (`0x0D0`).  
7. Click **Generate 144-Bit Distress Frame** to assemble the 144-bit burst, compute the 15-Hex UIN, calculate both BCH-1 (21 bits) and BCH-2 (12 bits) remainders, and inspect the binary stream.  
8. Click **Run Normative Audit** to evaluate all 10 C/S T.001 / C/S T.018 compliance checks.

## 2\. Standalone Headless Python CLI

Run automated regression testing and compliance verification:

python3 cospas-cli.py \--audit \--country 366 \--serial 1042 \--lat 25.2048 \--lon 55.2708