# World Server Protocol

After the [Login Service Protocol][login_service_protocol] send informations
regarding one server (or more), the client WAR.exe will connect to a World
server.

## Packet format

### Structur declaration

Inside some packets some buffers are sent, using the following structur :

    /* WAR_B_BUFFER */
    +0x00   :   SIZE_DATA           [BYTE]
    +0x01   :   DATA                [BYTE] * SIZE_DATA

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

* RVA : 0x004AF5EE
* VA : 0x00AF5EE

PAT signature (for sigmake IDA flair) :

    558BEC81EC04010000568BF033C03945080F84CA00000039450C0F84C1000000 C6 43C9 00E6 :0000 WAR_RC4_Encrypt

### WAR_RC4_Decrypt

Function in the client used to decrypt message from the server

    004AF6D4     ; signed int __usercall WAR_RC4_Decrypt<eax>(BYTE *Key<eax>, BYTE *Data, unsigned int SizeData)

* RVA : 0x004AF6D4
* VA : 0x000AF6D4

PAT signature (for sigmake IDA flair) :

    558BEC81EC040100005333DB395D08568BF00F84C1000000395D0C0F84B80000 BF 71CA 00DF :0000 WAR_RC4_Decrypt

## Opcodes list received

### 0x04

    004B2E63 SendPacket_0x04 proc near

Packet data :

    +0x00   :   SESSION_ID          [WORD]
    +0x02   :   UNK_WORD_00         [WORD]

### 0x0B

    004B2F33 PrepareSendPacket_0x0B proc near

Packt data :

    +0x00   :   UNK_DWORD_00        [DWORD]
    +0x04   :   UNK_DWORD_01        [DWORD]
    +0x08   :   UNK_DWORD_02        [DWORD]
    +0x0C   :   UNK_DATA            [WORD] * 3
    +0x012  :   UNK_WORD_00         [WORD]

The server must answer with opcode 0x81.

### 0x0F

...

    004B29A8     PrepareSendPacket_0x0F_01 proc near

...

    004B27CF     PrepareSendPacket_0x0F_02 proc near

Packet data :

    +0x00   :   UNK_BYTE_00         [BYTE]
    +0x01   :   UNK_BYTE_01         [BYTE]
    +0x02   :   MAJOR_VERSION       [BYTE]
    +0x03   :   MINOR_VERSION       [BYTE]
    +0x04   :   REVISION_VERSION    [BYTE]
    +0x05   :   UNK_BYTE_02         [BYTE]
    +0x06   :   UNK_WORD_00         [WORD]
    +0x08   :   PROTOCOL_VERSION    [DWORD]
    +0x0C   :   SESSION             [BYTE] * 101
    +0x71   :   USERNAME            [BYTE] * 21
    +0x86   :   SIZE_XML            [WORD]
    +0x88   :   XML_DATA            [BYTE] * SIZE_XML

### XML DATA

*TODO, not really important*

The server must answer with opcode 0x82.

### 0x13

...

    004B162A SendPacket_0x13 proc near

Packet data :

    +0x00   :   UNK_BYTE_00         [BYTE]

The server must answer with opcode 0x13.

### 0x54

This packet can be send from two functions.

Packet data :

    +0x00   :   COMMAND             [WORD]
    +0x02   :   UNK_BYTE_00         [BYTE]

#### 0x2D58

    004B19C6 SendPacket_0x54_0x2D58 proc near

The server must answer with opcode 0x56.

#### 0x2D53

    004B1A64 SendPacket_0x54_0x2D53 proc near

The server must answer with opcode 0x55.


### 0x5C

First packet received from the client after connection.
This packet can be send from two functions.

#### 0x5C (First)

    004B17D7     SendPacket_0x5C_1 proc near

* RVA    : 0x004B17D7
* VA     : 0x000B17D7

Packet data :

    +0x00   :   KEY_PRESENT         [BYTE]
    +0x01   :   UNK_BYTE_00         [BYTE]
    +0x02   :   MAJOR_VERSION       [BYTE]
    +0x03   :   MINOR_VERSION       [BYTE]
    +0x04   :   REVISION_VERSION    [BYTE]
    +0x05   :   UNK_BYTE_01         [BYTE]

The server must answer with opcode 0x8A.

### 0x5C (Second)

    004B2B84     SendPacket_0x5C_2 proc near

* RVA    : 0x004B2B84
* VA     : 0x000B2B84

Packet data :

    +0x00   :   KEY_PRESENT         [BYTE]
    +0x01   :   UNK_BYTE_00         [BYTE]
    +0x02   :   MAJOR_VERSION       [BYTE]
    +0x03   :   MINOR_VERSION       [BYTE]
    +0x04   :   REVISION_VERSION    [BYTE]
    +0x05   :   UNK_BYTE_01         [BYTE]
    +0x06   :   WAR_RC4_KEY         [BYTE] * 256

The server doesn't need to answer to this.

### 0x68

    004B3ABD SendPacket_0x68 proc near

Packet data :

    +0x00   :   CHAR_NAME           [BYTE] * 24
    +0x18   :   PADDING             [DWORD]
    +0x1C   :   PADDING             [WORD]
    +0x1E   :   USER_NAME           [BYTE] * 20
    +0x32   :   PADDING             [DWORD]

The server must answer with opcode 0x6A.

### 0x91

    004C6DFA SendPacket_0x91 proc near

Packet data :

The server must answer with opcode 0x58.

### 0xB8

    004B2C27 PrepareSendPacket_0xB8 proc near

Packet data :

    +0x00   :   UNK_WORD_00         [WORD]
    +0x02   :   UNK_BYTE_00         [BYTE]
    +0x03   :   UNK_BYTE_01         [BYTE]
    +0x04   :   UNK_DATA_00         [BYTE] * 24
    +0x1C   :   NS_PORT             [WORD]
    +0x1E   :   LANGAGE             [BYTE] * 6
    +0x24   :   UNK_DWORD_00        [DWORD]
    +0x28   :   UNK_DWORD_01        [DWORD]
    +0x2C   :   UNK_DWORD_02        [DWORD]

The server must answer with opcode 0x80.

## Opcodes list send

### 0x13

Answer to packet 0x13.

    004C3300 mov     [ebp+var_10], offset aF_request_char ; "F_REQUEST_CHAR_TEMPLATES"

...

    004C8AA5 Handle_0x13 proc near

Packet data :

    +0x00   :   NB_AVAILABLETEMPLATES_NAMES     [DWORD]
    +0x04   :   TEMPLATES_NAMES                 [WAR_BUFFER] * NB_AVAILABLETEMPLATES_NAMES
    +0x..   :   NB_AVAILABLETEMPLATES_CLASSES   [DWORD]
    +0x..   :   TEMPLATES_CLASSES               [DWORD] * NB_AVAILABLETEMPLATES_CLASSES
    +0x..   :   NB_AVAILABLETEMPLATES_RACES     [DWORD]
    +0x..   :   TEMPLATES_RACES                 [DWORD] * NB_AVAILABLETEMPLATES_RACES
    +0x..   :   NB_AVAILABLETEMPLATES_GENDERS   [DWORD]
    +0x..   :   TEMPLATES_GENDERS               [DWORD] * NB_AVAILABLETEMPLATES_GENDERS

### 0x55

Answer to packet 0x54 with command 0x2D53

    004C8CC1 Handle_0x55 proc near

Packet data :

    +0x00   :   UNK_DWORD_00        [DWORD]

### 0x56

Answer to packet 0x54 with command 0x2D58

No function handler.

    .text:004C3C23                 movzx   eax, byte ptr [edi]  // Buffer
    .text:004C3C26                 push    eax
    .text:004C3C27                 push    56h

Packet data :

    +0x00   :   UNK_BYTE_00         [BYTE]

### 0x6A

Answer to packet 0x68.
Check if the character name is valid and not used.

No function handler, only check the byte value at offset 0x32, WTF

    .text:004C3E58                 xor     eax, eax
    .text:004C3E5A                 cmp     [edi+32h], al
    .text:004C3E5D                 setz    al

Packet data :

    +0x00   :   DATA_OSEF           [BYTE] * 0x31
    +0x32   :   NAME_VALID          [BYTE]

### 0x80

Answer to packet 0xB8.

    004C8A86 Handle_0x80 proc near

Packet data :

    +0x00   :   SESSION_ID          [WORD]


### 0x82

Answer to packet 0x0F.

    004C88F9     Handle_0x82 proc near
    ...
    004C8954 mov     [ebp+var_C], offset aS_connected ; "S_CONNECTED"

Packet data :

    +0x00   :   UNK_BYTE_00         [BYTE]
    +0x01   :   UNK_BYTE_01         [BYTE]
    +0x02   :   UNK_BYTE_02         [BYTE]
    +0x03   :   UNK_BYTE_03         [BYTE]
    +0x04   :   PROTOCOL_VERSION    [DWORD]
    +0x08   :   SERVER_ID           [BYTE]
    +0x09   :   UNK_BYTE_04         [BYTE]
    +0x0A   :   UNK_BYTE_05         [BYTE]
    +0x0B   :   UNK_BYTE_06         [BYTE]
    +0x0C   :   TRANSFER_FLAG       [BYTE]
    +0x0D   :   USERNAME            [WAR_B_BUFFER]
    +0x..   :   SERVER_NAME         [WAR_B_BUFFER]
    +0x..   :   UNK_BYTE_07         [BYTE]  // nb iteration to read a BYTE
    +0x..   :   UNK_DATA            [BYTE] * UNK_BYTE_07

Protocol version must be equal to 0xEB8DB21.

### 0x8A

Answer to packet 0x5C.

    004C8883     Handle_0x8A proc near

* RVA    : 0x004C8883
* VA     : 0x000C8883

Packet data :

    +0x00   :   SEND_KEY            [BYTE]

SEND_KEY must be equal to 1, and the client will send opcode 0x5C again
containing the key for encrypt/decrypt.

[login_service_protocol]:./LoginServiceProtocol.md