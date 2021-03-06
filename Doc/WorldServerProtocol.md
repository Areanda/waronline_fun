# World Server Protocol

After the [Login Service Protocol][login_service_protocol] send informations
regarding one server (or more), the client WAR.exe will connect to a World
server.

## Packet format

### Structur declaration

Inside some packets some buffers are sent, using the following structur :

#### WAR_B_BUFFER

    /* WAR_B_BUFFER */
    +0x00   :   SIZE_DATA           [BYTE]
    +0x01   :   DATA                [BYTE] * SIZE_DATA

#### WAR_PLAYER

    /* WAR_PLAYER */
    +0x000  :   NICKNAME            [BYTE] * 48
    +0x030  :   LEVEL               [BYTE]
    +0x031  :   CAREER              [BYTE]      // 0x1A == Shaman
    +0x032  :   REALM               [BYTE]      // ORDER = 1 ; DESTRU = 2
    +0x033  :   GENDER              [BYTE]      //
    +0x034  :   UNK_WORD_00         [WORD]      // Crash game :(
    +0x036  :   ZONE_ID             [WORD]      //
    +0x038  :   UNK_DATA_00         [BYTE] * 12
    +0x044  :   NS_INVENTORY        [WAR_INVENTORY] * 16
    +0x0C4  :   NS_OTHER_INVENTORY  [WAR_INVENTORY] * 5
    +0x0EC  :   UNK_DATA_01         [BYTE] * 0x14
    +0x100  :   ...
    ...
    +0x11C  :   ...

#### WAR_INVENTORY

### Received FROM client

All packet received from the client follow this struct :

    +0x00   :   SIZE_DATA           [WORD] // minus HEADER_SIZE and CRC

The server must recv/read SIZE_DATA + 8 + 2.

* 8 for sizeof (HEADER_SIZE).
* 2 for size of CRC aka sizeof (WORD).

Packet content struct :

    /* HEADER       */
    +0x00   :   SEQUENCE_PACKET     [WORD]
    +0x02   :   SESSION_ID_PACKET   [WORD]
    +0x04   :   UNK_WORD_00         [WORD]
    +0x06   :   UNK_BYTE_00         [BYTE]
    +0x07   :   OPCODE_PACKET       [BYTE]
    /* END HEADER   */
    +0x08   :   DATA_PACKET         [BYTE] * SIZE_DATA
    +0x..   :   CRC                 [WORD]

### Sent TO client

All packet sent by the server to the client follow this struct :

    +0x00   :   SIZE_PACKET         [WORD] // minus OPCODE
    +0x02   :   OPCODE              [BYTE]
    +0x03   :   DATA                [BYTE] * SIZE_PACKET

## Packet encryption

The client send the key of 0x100 (256) bytes length in the opcode 0x5C (*TODO
put ref to other doc*).

They use a really weird implementation of RC4 (here in python) :

    def WAR_RC4(data, key, encrypt = True):
        j = 0
        i = 0
        out_first_half = []
        out_second_half = []
        half_len = (len(data) / 2)
        S = []
        for val in key:
            S.append(ord(val))
        for char in data[half_len:]:
            i = (i + 1) % 256
            j = (j + S[i]) % 256
            S[i] , S[j] = S[j] , S[i]
            c = ord(char) ^ S[(S[i] + S[j]) % 256]
            out_second_half.append(chr(c))
            if encrypt == True:
                j = (j + ord(char)) % 256
            else:
                j = (j + c) % 256
        for char in data[:half_len]:
            i = (i + 1) % 256
            j = (j + S[i]) % 256
            S[i] , S[j] = S[j] , S[i]
            c = ord(char) ^ S[(S[i] + S[j]) % 256]
            out_first_half.append(chr(c))
            if encrypt == True:
                j = (j + ord(char)) % 256
            else:
                j = (j + c) % 256
        return ''.join(out_first_half) + ''.join(out_second_half)

### WAR_RC4_Encrypt

Function in the client used to encrypt message for the server

    004AF5EE     ; signed int __usercall WAR_RC4_Encrypt<eax>(const void *Key<eax>, char *Data, unsigned int SizeData)

* VA    : 0x004AF5EE
* RVA   : 0x000AF5EE

PAT signature (for sigmake IDA flair) :

    558BEC81EC04010000568BF033C03945080F84CA00000039450C0F84C1000000 C6 43C9 00E6 :0000 WAR_RC4_Encrypt

### WAR_RC4_Decrypt

Function in the client used to decrypt message from the server

    004AF6D4     ; signed int __usercall WAR_RC4_Decrypt<eax>(BYTE *Key<eax>, BYTE *Data, unsigned int SizeData)

* VA    : 0x004AF6D4
* RVA   : 0x000AF6D4

PAT signature (for sigmake IDA flair) :

    558BEC81EC040100005333DB395D08568BF00F84C1000000395D0C0F84B80000 BF 71CA 00DF :0000 WAR_RC4_Decrypt

[login_service_protocol]:./LoginServiceProtocol.md
