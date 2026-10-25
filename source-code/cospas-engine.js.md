/\*\*  
 \* Cospas-Sarsat C/S T.001 & C/S T.018 406 MHz Distress Beacon Core Protocol Engine  
 \* Implements 112-bit / 144-bit transmission framing, 15-Hex UIN computation,  
 \* BCH-1 (21-bit) and BCH-2 (12-bit) error-correcting codes, and GPS coordinate encoding.  
 \*/

// Cospas-Sarsat Frame Sync Codes (Bits 16–24)  
const FRAME\_SYNC\_CODES \= {  
  NORMAL: 0x02F, // 000101111 (Default for operational distress transmission)  
  TEST:   0x0D0  // 011010000 (Self-test transmission flag)  
};

// Beacon Types  
const BEACON\_TYPES \= {  
  ELT: 'ELT',   // Emergency Locator Transmitter (Aviation)  
  EPIRB: 'EPIRB', // Emergency Position Indicating Radio Beacon (Maritime)  
  PLB: 'PLB'    // Personal Locator Beacon (Land / CSAR)  
};

// Standard Country Codes (ITU Maritime Identification Digits \- MID)  
const COUNTRY\_CODES \= {  
  USA: 366,  
  UK:  232,  
  UAE: 470,  
  FRANCE: 227,  
  GERMANY: 211,  
  AUSTRALIA: 503  
};

class COSPASEngine {  
  constructor(countryCode \= COUNTRY\_CODES.USA, beaconType \= BEACON\_TYPES.ELT) {  
    this.countryCode \= countryCode; // 10-bit integer (0–1023)  
    this.beaconType \= beaconType;  
    this.serialNumber \= 1042;  
  }

  /\*\*  
   \* Polynomial division for BCH error-correcting code  
   \* @param {Array\<number\>} dataBits \- Array of 0 and 1 bits  
   \* @param {number} poly \- Generator polynomial integer  
   \* @param {number} polyDegree \- Degree of generator polynomial  
   \*/  
  static computeBch(dataBits, poly, polyDegree) {  
    // Left-shift data bits by polyDegree (append zeroes)  
    const dividend \= \[...dataBits, ...new Array(polyDegree).fill(0)\];  
      
    // Convert poly to bit array  
    const polyBits \= \[\];  
    for (let i \= polyDegree; i \>= 0; i--) {  
      polyBits.push((poly \>\> i) & 1);  
    }

    // Perform modulo-2 polynomial division  
    for (let i \= 0; i \< dataBits.length; i++) {  
      if (dividend\[i\] \=== 1\) {  
        for (let j \= 0; j \<= polyDegree; j++) {  
          dividend\[i \+ j\] ^= polyBits\[j\];  
        }  
      }  
    }

    // Return the remainder (last polyDegree bits)  
    return dividend.slice(dataBits.length);  
  }

  /\*\*  
   \* Computes BCH-1 (21 parity bits over bits 25 to 85\)  
   \* Generator Polynomial: x^21 \+ x^20 \+ x^18 \+ x^16 \+ x^14 \+ x^13 \+ x^11 \+ x^6 \+ x^5 \+ x^3 \+ x \+ 1 (0x33180B or 0x13180B)  
   \* Standard C/S T.001 value: 0x13180B (deg 21\)  
   \*/  
  static calculateBch1(bits25to85) {  
    const poly \= 0x13180B; // 21-degree generator  
    return COSPASEngine.computeBch(bits25to85, poly, 21);  
  }

  /\*\*  
   \* Computes BCH-2 (12 parity bits over bits 107 to 132\)  
   \* Generator Polynomial: x^12 \+ x^10 \+ x^8 \+ x^5 \+ x^4 \+ x^3 \+ 1 (0x1539)  
   \*/  
  static calculateBch2(bits107to132) {  
    const poly \= 0x1539; // 12-degree generator  
    return COSPASEngine.computeBch(bits107to132, poly, 12);  
  }

  /\*\*  
   \* Computes 15-Hexadecimal Unique Identifier (UIN) from bits 26 through 85 (60 bits \= 15 hex characters)  
   \*/  
  static compute15HexUin(bits26to85) {  
    let hexStr \= '';  
    for (let i \= 0; i \< 60; i \+= 4\) {  
      let nibble \= (bits26to85\[i\] \<\< 3\) \| (bits26to85\[i \+ 1\] \<\< 2\) \| (bits26to85\[i \+ 2\] \<\< 1\) \| bits26to85\[i \+ 3\];  
      hexStr \+= nibble.toString(16).toUpperCase();  
    }  
    return hexStr;  
  }

  /\*\*  
   \* Encodes standard latitude/longitude coordinates into 144-bit Long Message location fields  
   \*/  
  static encodeCoordinates(latDeg, lonDeg) {  
    // Latitude: Bit 65 (Sign: 0=N, 1=S), Bits 66-72 (Degrees 0-90), Bits 73-74 (Minutes in 15-min increments 0, 15, 30, 45\)  
    const latSign \= latDeg \>= 0 ? 0 : 1;  
    const latAbs \= Math.abs(latDeg);  
    const latDegrees \= Math.floor(latAbs) & 0x7F;  
    const latMinutesQuarter \= Math.floor((latAbs \- Math.floor(latAbs)) \* 4\) & 0x03;

    // Longitude: Bit 75 (Sign: 0=E, 1=W), Bits 76-83 (Degrees 0-180), Bits 84-85 (Minutes in 15-min increments)  
    const lonSign \= lonDeg \>= 0 ? 0 : 1;  
    const lonAbs \= Math.abs(lonDeg);  
    const lonDegrees \= Math.floor(lonAbs) & 0xFF;  
    const lonMinutesQuarter \= Math.floor((lonAbs \- Math.floor(lonAbs)) \* 4\) & 0x03;

    return {  
      latBits: \[  
        latSign,  
        ...COSPASEngine.numToBits(latDegrees, 7),  
        ...COSPASEngine.numToBits(latMinutesQuarter, 2\)  
      \],  
      lonBits: \[  
        lonSign,  
        ...COSPASEngine.numToBits(lonDegrees, 8),  
        ...COSPASEngine.numToBits(lonMinutesQuarter, 2\)  
      \]  
    };  
  }

  static numToBits(num, length) {  
    const bits \= \[\];  
    for (let i \= length \- 1; i \>= 0; i--) {  
      bits.push((num \>\> i) & 1);  
    }  
    return bits;  
  }

  /\*\*  
   \* Generates a complete 144-bit Long Message Cospas-Sarsat Distress Frame  
   \*/  
  generateLongMessage(latDeg \= 25.2048, lonDeg \= 55.2708, isTest \= false) {  
    // 1\. Bit Sync (Bits 1–15): 15 bits of 1s  
    const bitSync \= new Array(15).fill(1);

    // 2\. Frame Sync (Bits 16–24): 9 bits  
    const frameSync \= isTest ? \[0,1,1,0,1,0,0,0,0\] : \[0,0,0,1,0,1,1,1,1\];

    // 3\. Format Flag (Bit 25): 1 \= Long message  
    const formatFlag \= \[1\];

    // 4\. Protocol Flag (Bit 26): 0 \= Standard Location Protocol  
    const protocolFlag \= \[0\];

    // 5\. Country Code (Bits 27–36): 10-bit ITU MID  
    const countryBits \= COSPASEngine.numToBits(this.countryCode, 10);

    // 6\. Protocol Identifier (Bits 37–40): e.g. Standard Location Protocol \= 0011 (3)  
    const protoIdBits \= \[0, 0, 1, 1\];

    // 7\. Type of Beacon & Serial Number (Bits 41–64): 24 bits  
    const serialBits \= COSPASEngine.numToBits(this.serialNumber, 24);

    // 8\. Location coordinates (Bits 65–85): 21 bits  
    const coords \= COSPASEngine.encodeCoordinates(latDeg, lonDeg);  
    const locBits \= \[...coords.latBits, ...coords.lonBits\];

    // Protected data bits 25 through 85 (61 bits)  
    const bits25to85 \= \[  
      ...formatFlag,  
      ...protocolFlag,  
      ...countryBits,  
      ...protoIdBits,  
      ...serialBits,  
      ...locBits  
    \];

    // 9\. Compute 15-Hex UIN over bits 26 through 85 (60 bits)  
    const uin15Hex \= COSPASEngine.compute15HexUin(bits25to85.slice(1));

    // 10\. Compute BCH-1 (Bits 86–106): 21 bits  
    const bch1Bits \= COSPASEngine.calculateBch1(bits25to85);

    // 11\. Supplementary Data (Bits 107–132): 26 bits (Encoded offset delta coordinates, RLS flag)  
    const suppBits \= \[  
      1, 1, 0, // Encoded delta minutes  
      0, 1, 0, 1,  
      0, 0, 1, 1,  
      1, 0, 1, 0,  
      1, 0, 1, 1, 0, 0, 1, 1, 0, 1, 0  
    \];

    // 12\. Compute BCH-2 (Bits 133–144): 12 bits  
    const bch2Bits \= COSPASEngine.calculateBch2(suppBits);

    // Full 144-bit frame assembly  
    const fullBits \= \[  
      ...bitSync,    // 15  
      ...frameSync,  // 9  
      ...bits25to85, // 61  
      ...bch1Bits,   // 21  
      ...suppBits,   // 26  
      ...bch2Bits    // 12  
    \]; // Total \= 144 bits

    // Convert to hex representation  
    let hexStream \= '';  
    for (let i \= 0; i \< fullBits.length; i \+= 4\) {  
      let nibble \= (fullBits\[i\] \<\< 3\) \| (fullBits\[i+1\] \<\< 2\) \| (fullBits\[i+2\] \<\< 1\) \| fullBits\[i+3\];  
      hexStream \+= nibble.toString(16).toUpperCase();  
    }

    return {  
      messageType: '144\_BIT\_LONG\_MESSAGE',  
      totalBits: fullBits.length,  
      uin15Hex: uin15Hex,  
      countryCode: this.countryCode,  
      beaconType: this.beaconType,  
      isTestTransmission: isTest,  
      coordinates: { lat: latDeg, lon: lonDeg },  
      bch1BitsCount: bch1Bits.length,  
      bch2BitsCount: bch2Bits.length,  
      hexRepresentation: hexStream,  
      bits: fullBits  
    };  
  }  
}

if (typeof module \!== 'undefined' && module.exports) {  
  module.exports \= {  
    COSPASEngine,  
    FRAME\_SYNC\_CODES,  
    BEACON\_TYPES,  
    COUNTRY\_CODES  
  };  
}  
