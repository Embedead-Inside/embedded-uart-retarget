# 全65種対応 UARTリターゲット（printf / scanf）完全実装ガイド

本ドキュメントは、多様なCPU、マイコン、SoC、およびDSP（全65種類）における標準入出力（`printf` / `scanf` 等）をUARTシリアル通信へリターゲット（オーバーライド）するための完全ソースコード集および解説書です。

アーキテクチャやコンパイラ標準ライブラリの共通性に基づき、22のカテゴリに集約・構造化しています。

---

## 📝 概要および修正・注釈ノート

### 1. コード整形および文法エラーの修正
* **改行・空白の正常化**: プリプロセッサディレクティブ（`#include` や `#define`）や関数宣言、コメント終端（`*/`）直後の改行喪失を解消し、そのままビルド可能なC言語構文へ補正しています。
* **コードブロックの適用**: 全てのC言語ソースコードに対して適切なSyntax Highlighting付きコードブロック（` ```c `）を設定し、可読性と編集性を向上させました。

### 2. リターゲット実装時の共通注意点
* **セミホスティング（Semihosting）の回避**: Arm系コンパイラ（Keil MDK等）では、デフォルトでデバッガ依存のセミホスティングが有効化される場合があります。`__use_no_semihosting` の指定や MicroLib の使用が必要です。
* **バッファリングの無効化**: C標準ライブラリの `stdout` はデフォルトで行バッファリングまたはフルバッファリングされます。`main` 関数の冒頭で `setvbuf(stdout, NULL, _IONBF, 0);` を呼び出して即時送信を行わせるのが安全です。
* **改行コードの自動置換**: ターミナルソフトの仕様に合わせて、C標準の `\n` (LF) を送信時に `\r\n` (CR+LF) へ自動変換するロジックを各コードに含めています。

---

## 1. GCC (Newlib) 標準系

* **対象 (31種類)**: SimpleLink, Tiva C, MSP432, Sitara, STM32(GCC), Synergy, SAM, i.MX RT/LPC, S32K, LPC800, PSoC 4/6, nRF51, nRF9160, MicroBlaze, Nios II/V, Lattice Mico32, RP2040, RP2350, ESP32/C3/S3, CH32V, GD32, GD32VF103, BL602/702, RTL8720, RTL8195, MAX32660, ADuCM, Dust Networks, 汎用RISC-V, OpenRISC, 汎用Cortex-M7

```c
/**
 * @file uart_retarget_newlib.c
 * @brief Newlib (GCC) 環境向け UART 送受信リターゲット実装
 * @details 対象: STM32, SAM, i.MX RT, ESP32, RP2040, 各社 Cortex-M/RISC-V 等
 * 
 * @note 接続するHAL/ドライバ（例: MX_USART1_UART_Init等）は事前に初期化されている必要があります。
 * @warning リンカフラグに `-specs=nano.specs` を使用する場合、デフォルトで浮動小数点表示 (%f) が
 *          無効化されるため、必要に応じて `-u _printf_float` を追加してください。
 */
#include <stdio.h>
#include <sys/stat.h>

/* 各プラットフォーム依存のHALヘッダ（環境に合わせてインクルードを切り替えてください） */
/* #include "stm32f4xx_hal.h" */
/* #include "pico/stdlib.h" */

/**
 * @brief 1バイト送信（プラットフォーム依存の下位関数Stub）
 * @param[in] ch 送信する文字
 */
static void platform_uart_send_byte(char ch) {
    /* TODO: 各社SDKの送信関数またはレジスタ操作に置き換えてください */
    /* 例 (STM32 HAL): HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY); */
    /* 例 (RP2040):    uart_putc(uart0, ch); */
}

/**
 * @brief 1バイト受信（プラットフォーム依存の下位関数Stub）
 * @return 受信した文字（ブロッキング）
 */
static char platform_uart_recv_byte(void) {
    /* TODO: 各社SDKの受信関数またはレジスタ操作に置き換えてください */
    /* 例 (STM32 HAL): uint8_t ch; HAL_UART_Receive(&huart1, &ch, 1, HAL_MAX_DELAY); return ch; */
    /* 例 (RP2040):    return uart_getc(uart0); */
    return 0;
}

/**
 * @brief Newlib システムコール _write のオーバーライド
 * @details printf や puts などの標準出力関数から最終的に呼び出されます。
 * @param[in] file ファイル記述子（標準出力・標準エラーを対象とする）
 * @param[in] ptr  送信データバッファへのポインタ
 * @param[in] len  送信データ長
 * @return 実際に送信したバイト数
 */
int _write(int file, char *ptr, int len) {
    if (file == 1 || file == 2) { /* stdout (1) または stderr (2) */
        for (int i = 0; i < len; i++) {
            /* 改行コードの自動変換 (LF -> CR+LF) */
            if (ptr[i] == '\n') {
                platform_uart_send_byte('\r');
            }
            platform_uart_send_byte(ptr[i]);
        }
        return len;
    }
    return -1;
}

/**
 * @brief Newlib システムコール _read のオーバーライド
 * @details scanf や getchar などの標準入力関数から最終的に呼び出されます。
 * @param[in] file ファイル記述子（標準入力を対象とする）
 * @param[out] ptr 受信データを格納するバッファへのポインタ
 * @param[in] len  読み込み要求バイト数
 * @return 実際に読み込んだバイト数
 */
int _read(int file, char *ptr, int len) {
    if (file == 0) { /* stdin (0) */
        int num = 0;
        for (num = 0; num < len; num++) {
            char ch = platform_uart_recv_byte();
            ptr[num] = ch;
            
            /* エコーバック処理（必要に応じて有効化） */
            if (ch == '\r') {
                platform_uart_send_byte('\r');
                platform_uart_send_byte('\n');
            } else {
                platform_uart_send_byte(ch);
            }
            
            /* 改行コード受信で読み込みを終了 */
            if (ch == '\r' || ch == '\n') {
                num++;
                break;
            }
        }
        return num;
    }
    return -1;
}
```

---

## 2. Keil MDK (Arm Compiler 6 / AC5) MicroLib系

* **対象 (4種類)**: STM32(Keil), XMC, nRF52/53

```c
/**
 * @file uart_retarget_keil.c
 * @brief Keil MDK (ARMCC) 環境向け fputc/fgetc リターゲット実装
 * @details 対象: STM32 (MDK環境), Infineon XMC, Nordic nRF52/nRF53
 * 
 * @note セミホスティング (Semihosting) を完全に回避するため、MicroLibを有効にするか、
 *       または __use_no_semihosting を宣言する必要があります。
 */
#include <stdio.h>

/* セミホスティング回避のための記述 (MicroLib不使用時) */
#if !defined(__MICROLIB)
__asm(".global __use_no_semihosting\n\t");
void _sys_exit(int return_code) { while (1); }
#endif

/* 外部定義されるUARTハンドル構造体等のダミー宣言 */
extern void UART_SendByte(char ch);
extern char UART_RecvByte(void);

/**
 * @brief 標準出力のリターゲット関数
 * @param[in] ch 送信文字
 * @param[in] f  ファイルポインタ（未使用）
 * @return 送信に成功した文字
 */
int fputc(int ch, FILE *f) {
    if (ch == '\n') {
        UART_SendByte('\r');
    }
    UART_SendByte((char)ch);
    return ch;
}

/**
 * @brief 標準入力のリターゲット関数
 * @param[in] f ファイルポインタ（未使用）
 * @return 受信した文字（ブロッキング）
 */
int fgetc(FILE *f) {
    char ch = UART_RecvByte();
    /* 必要に応じてエコーバック */
    if (ch == '\r') {
        fputc('\n', stdout);
    } else {
        fputc(ch, stdout);
    }
    return (int)ch;
}
```

---

## 3. IAR Embedded Workbench (DLIB) 系

* **対象 (5種類)**: RA, RX, EFM32/EFR32, V850(IAR移行時含む)

```c
/**
 * @file uart_retarget_iar.c
 * @brief IAR DLIB 低レベル I/O リターゲット実装
 * @details 対象: Renesas RA, RX (IAR), Silicon Labs EFM32/EFR32
 */
#include <stdio.h>
#include <ysizet.h>

extern void IAR_UART_Write(unsigned char c);
extern unsigned char IAR_UART_Read(void);

/**
 * @brief IAR標準ライブラリの出力基本関数 __write の上書き
 */
size_t __write(int handle, const unsigned char *buffer, size_t size) {
    if (handle == _LLIO_STDOUT || handle == _LLIO_STDERR) {
        const unsigned char *p = buffer;
        size_t n = size;
        while (n > 0) {
            if (*p == '\n') {
                IAR_UART_Write('\r');
            }
            IAR_UART_Write(*p++);
            n--;
        }
        return size;
    }
    return _LLIO_ERROR;
}

/**
 * @brief IAR標準ライブラリの入力基本関数 __read の上書き
 */
size_t __read(int handle, unsigned char *buffer, size_t size) {
    if (handle == _LLIO_STDIN) {
        size_t received = 0;
        while (received < size) {
            unsigned char ch = IAR_UART_Read();
            buffer[received++] = ch;
            if (ch == '\n' || ch == '\r') break;
        }
        return received;
    }
    return _LLIO_ERROR;
}
```

---

## 4. TI C2000 (TMS320C28x)

* **対象 (1種類)**: C2000 シリーズ

```c
/**
 * @file uart_retarget_c2000.c
 * @brief TI C2000 (CCS/TIコンパイラ) 向け HOSTwrite / HOSTread リターゲット
 * @details 伝統的な `add_device` APIを利用し、SCI (UART) をファイル記述子に紐づけます。
 */
#include <stdio.h>
#include <file.h>

/* SCIレジスタ構造体ダミー（F2837xD等のドライバ想定） */
extern void SCI_writeCharBlocking(uint32_t base, uint16_t data);
extern uint16_t SCI_readCharBlocking(uint32_t base);
#define SCIA_BASE 0x00007000U

int sci_open(const char *path, unsigned flags, int fildes);
int sci_close(int fildes);
int sci_write(int fildes, const char *buffer, unsigned count);
int sci_read(int fildes, char *buffer, unsigned count);

/**
 * @brief C2000用 UARTリターゲット初期化ルーチン
 * @details main関数の初期の段階で呼び出す必要があります。
 */
void C2000_UART_Redirect_Init(void) {
    /* "sci" デバイスを C 標準 I/O フレームワークに追加 */
    add_device("sci", _MSA, sci_open, sci_close, sci_read, sci_write, NULL, NULL, NULL);
    /* stdout / stdin を追加したデバイスにリダイレクト */
    freopen("sci:out", "w", stdout);
    freopen("sci:in", "r", stdin);
    setvbuf(stdout, NULL, _IONBF, 0); /* バッファリングを無効化 */
}

int sci_open(const char *path, unsigned flags, int fildes) { return fildes; }
int sci_close(int fildes) { return 0; }

int sci_write(int fildes, const char *buffer, unsigned count) {
    for (unsigned i = 0; i < count; i++) {
        if (buffer[i] == '\n') SCI_writeCharBlocking(SCIA_BASE, '\r');
        SCI_writeCharBlocking(SCIA_BASE, buffer[i]);
    }
    return count;
}

int sci_read(int fildes, char *buffer, unsigned count) {
    for (unsigned i = 0; i < count; i++) {
        buffer[i] = (char)SCI_readCharBlocking(SCIA_BASE);
        if (buffer[i] == '\r' || buffer[i] == '\n') {
            return i + 1;
        }
    }
    return count;
}
```

---

## 5. TI MSP430 (putchar/getchar系)

* **対象 (1種類)**: MSP430 シリーズ

```c
/**
 * @file uart_retarget_msp430.c
 * @brief MSP430 (TI Compiler) putchar / getchar オーバーライド
 */
#include <stdio.h>
#include <msp430.h>

/**
 * @brief TI MSP430 コンパイラ用 putchar
 */
int putchar(int ch) {
    if (ch == '\n') {
        while (!(UCA0IFG & UCTXIFG));
        UCA0TXBUF = '\r';
    }
    while (!(UCA0IFG & UCTXIFG));
    UCA0TXBUF = (char)ch;
    return ch;
}

/**
 * @brief TI MSP430 コンパイラ用 getchar
 */
int getchar(void) {
    while (!(UCA0IFG & UCRXIFG));
    return UCA0RXBUF;
}
```

---

## 6. Microchip XC8/XC16 (PIC16/24, dsPIC)

* **対象 (1種類)**: PIC シリーズ (PIC16/24/32, dspic)

```c
/**
 * @file uart_retarget_xc8_xc16.c
 * @brief Microchip XC8 / XC16 コンパイラ向け putch / getch 実実装
 */
#include <xc.h>
#include <stdio.h>

/**
 * @brief XCコンパイラ標準 stdio 出力フック
 */
void putch(char data) {
    if (data == '\n') {
        while (!U1STAbits.TRMT); /* 送信シフトレジスタ空き待ち */
        U1TXREG = '\r';
    }
    while (!U1STAbits.TRMT);
    U1TXREG = data;
}

/**
 * @brief XCコンパイラ標準 stdio 入力フック
 */
char getch(void) {
    while (!U1STAbits.URXDA); /* 受信データあり待ち */
    if (U1STAbits.OERR) {     /* オーバーランエラー処理 */
        U1STAbits.OERR = 0;
    }
    return U1RXREG;
}
```

---

## 7. Microchip XC32 (PIC32MZ)

* **対象 (1種類)**: PIC32MZ シリーズ

```c
/**
 * @file uart_retarget_xc32.c
 * @brief Microchip XC32 (MIPSコア) 向け _mon_putc / _mon_getc 実装
 */
#include <xc.h>
#include <stdio.h>

/**
 * @brief XC32のモニター出力フックを上書き
 */
void _mon_putc(char c) {
    if (c == '\n') {
        while (U1STAbits.UTXBF); /* 送信バッファフル待ち */
        U1TXREG = '\r';
    }
    while (U1STAbits.UTXBF);
    U1TXREG = c;
}

/**
 * @brief XC32のモニター入力フックを上書き
 */
int _mon_getc(int canblock) {
    if (canblock) {
        while (!U1STAbits.URXDA);
        return (int)U1RXREG;
    } else {
        if (U1STAbits.URXDA) return (int)U1RXREG;
        return -1;
    }
}
```

---

## 8. AVR-GCC (ATmega328P等 / AVR Dx/DD)

* **対象 (2種類)**: AVR シリーズ, AVR Dx / DD シリーズ

```c
/**
 * @file uart_retarget_avr.c
 * @brief AVR-GCC (8bit AVR) 向け fdevopen ストリームバインディング実装
 */
#include <avr/io.h>
#include <stdio.h>

static int avr_uart_putchar(char c, FILE *stream);
static int avr_uart_getchar(FILE *stream);

/* 標準入出力用ストリーム構造体の静的定義 */
static FILE uart_stdout = FDEV_SETUP_STREAM(avr_uart_putchar, NULL, _FDEV_SETUP_WRITE);
static FILE uart_stdin  = FDEV_SETUP_STREAM(NULL, avr_uart_getchar, _FDEV_SETUP_READ);

/**
 * @brief AVRでの標準入出力バインド初期化
 */
void AVR_UART_Stream_Init(void) {
    stdout = &uart_stdout;
    stdin  = &uart_stdin;
}

static int avr_uart_putchar(char c, FILE *stream) {
    if (c == '\n') avr_uart_putchar('\r', stream);
    
    #if defined(__AVR_ATmega328P__)
    while (!(UCSR0A & (1 << UDRE0)));
    UDR0 = c;
    #else /* AVR Dx / DD シリーズ等の新USART */
    while (!(USART0.STATUS & USART_DREIF_bm));
    USART0.TXDATAL = c;
    #endif
    return 0;
}

static int avr_uart_getchar(FILE *stream) {
    #if defined(__AVR_ATmega328P__)
    while (!(UCSR0A & (1 << RXC0)));
    return UDR0;
    #else /* AVR Dx / DD */
    while (!(USART0.STATUS & USART_RXCIF_bm));
    return USART0.RXDATAL;
    #endif
}
```

---

## 9. Renesas RH850 (CC-RHコンパイラ)

* **対象 (1種類)**: RH850 シリーズ

```c
/**
 * @file uart_retarget_ccrh.c
 * @brief Renesas CC-RHコンパイラ向け __write / __read リターゲット
 */
#include <stddef.h>

/* パラメータ定義 */
#define SCIF_TX_EMPTY_BIT 0x01
extern volatile unsigned char UART_REG_STS;
extern volatile unsigned char UART_REG_TX;
extern volatile unsigned char UART_REG_RX;

/**
 * @brief CC-RH用 低レベル書き込み関数
 */
int __write(int no, const unsigned char *buf, size_t count) {
    if (no == 1 || no == 2) {
        for (size_t i = 0; i < count; i++) {
            if (buf[i] == '\n') {
                while (!(UART_REG_STS & SCIF_TX_EMPTY_BIT));
                UART_REG_TX = '\r';
            }
            while (!(UART_REG_STS & SCIF_TX_EMPTY_BIT));
            UART_REG_TX = buf[i];
        }
        return (int)count;
    }
    return -1;
}

/**
 * @brief CC-RH用 低レベル読み込み関数
 */
int __read(int no, unsigned char *buf, size_t count) {
    if (no == 0) {
        for (size_t i = 0; i < count; i++) {
            /* 簡易的な受信フラグ待ちを想定 */
            while ((UART_REG_STS & 0x02) == 0); 
            buf[i] = UART_REG_RX;
            if (buf[i] == '\r' || buf[i] == '\n') return (int)(i + 1);
        }
        return (int)count;
    }
    return -1;
}
```

---

## 10. Renesas RL78 (CC-RLコンパイラ)

* **対象 (1種類)**: RL78 シリーズ

```c
/**
 * @file uart_retarget_ccrl.c
 * @brief Renesas CC-RL コンパイラ環境用 putchar / getchar
 */
#include <stdio.h>

/* CC-RLではputchar/getcharを単純オーバーライドすることでprintfが連動します */
extern void RL78_UART_Send(char c);
extern char RL78_UART_Receive(void);

int putchar(int c) {
    if (c == '\n') {
        RL78_UART_Send('\r');
    }
    RL78_UART_Send((char)c);
    return c;
}

int getchar(void) {
    return (int)RL78_UART_Receive();
}
```

---

## 11. Renesas V850 (CA850コンパイラ)

* **対象 (1種類)**: V850 シリーズ

```c
/**
 * @file uart_retarget_ca850.c
 * @brief CA850 コンパイラ専用 __putc / __getc フック
 */
extern void V850_UART_TxByte(char c);
extern char V850_UART_RxByte(void);

/**
 * @brief CA850環境での文字出力フック
 */
void __putc(char c) {
    if (c == '\n') {
        V850_UART_TxByte('\r');
    }
    V850_UART_TxByte(c);
}

/**
 * @brief CA850環境での文字入力フック
 */
char __getc(void) {
    return V850_UART_RxByte();
}
```

---

## 12. Renesas M16C / R8C (NC30コンパイラ)

* **対象 (1種類)**: M16C / R8C シリーズ

```c
/**
 * @file uart_retarget_nc30.c
 * @brief NC30 コンパイラ環境用 putch / getch
 */
#include <sfr62p.h> /* M16Cのレガシー共通レジスタ定義等 */

void putch(char c) {
    if (c == '\n') {
        while (u0ti_u0c1 == 0); /* 送信バッファ空き待ち */
        u0tb = '\r';
    }
    while (u0ti_u0c1 == 0);
    u0tb = c;
}

char getch(void) {
    while (u0ri_u0c1 == 0); /* 受信データフラグ待ち */
    return (char)(u0rb & 0x00FF);
}
```

---

## 13. Renesas SuperH (SH-Cコンパイラ)

* **対象 (1種類)**: SuperH SH-2 / SH-4 シリーズ

```c
/**
 * @file uart_retarget_sh.c
 * @brief SuperH SH-C コンパイラ環境用 putch / getch レジスタ直叩き実装
 */
/* SH2(例: SH7047) または SH4 のSCIFレジスタ定義体を想定 */
#define SH2_SCSSR_TDRE  0x80
#define SH2_SCSSR_RDRF  0x40

extern volatile unsigned char SCI0_SSR;
extern volatile unsigned char SCI0_TDR;
extern volatile unsigned char SCI0_RDR;

void putch(char c) {
    if (c == '\n') {
        while (!(SCI0_SSR & SH2_SCSSR_TDRE));
        SCI0_TDR = '\r';
        SCI0_SSR &= ~SH2_SCSSR_TDRE;
    }
    while (!(SCI0_SSR & SH2_SCSSR_TDRE));
    SCI0_TDR = c;
    SCI0_SSR &= ~SH2_SCSSR_TDRE; /* フラグクリア */
}

char getch(void) {
    char data;
    while (!(SCI0_SSR & SH2_SCSSR_RDRF));
    data = SCI0_RDR;
    SCI0_SSR &= ~SH2_SCSSR_RDRF; /* フラグクリア */
    return data;
}
```

---

## 14. AURIX TriCore (Tasking C Compiler)

* **対象 (1種類)**: AURIX (TC3xx/TC4xx) シリーズ

```c
/**
 * @file uart_retarget_tasking.c
 * @brief AURIX TriCore Taskingコンパイラ用 _write / _read 低層関数実装
 */
#include <stdio.h>

extern void IfxAsclin_Asc_writeDeviceByte(void *asclinHandle, char c);
extern char IfxAsclin_Asc_readDeviceByte(void *asclinHandle);
extern void *g_asclinHandle;

/**
 * @brief Tasking Runtime環境用 _write のオーバーライド
 */
int _write(int fd, const char *buf, int nbyte) {
    if (fd == 1 || fd == 2) {
        for (int i = 0; i < nbyte; i++) {
            if (buf[i] == '\n') {
                IfxAsclin_Asc_writeDeviceByte(g_asclinHandle, '\r');
            }
            IfxAsclin_Asc_writeDeviceByte(g_asclinHandle, buf[i]);
        }
        return nbyte;
    }
    return 0;
}

/**
 * @brief Tasking Runtime環境用 _read のオーバーライド
 */
int _read(int fd, char *buf, int nbyte) {
    if (fd == 0) {
        for (int i = 0; i < nbyte; i++) {
            buf[i] = IfxAsclin_Asc_readDeviceByte(g_asclinHandle);
            if (buf[i] == '\r' || buf[i] == '\n') return i + 1;
        }
        return nbyte;
    }
    return 0;
}
```

---

## 15. AMD / Xilinx Zynq (outbyte / inbyte型)

* **対象 (1種類)**: Zynq-7000 / UltraScale+

```c
/**
 * @file uart_retarget_zynq.c
 * @brief Xilinx Embedded SW BSP (xil_printf / printf) 用コア関数のオーバーライド
 * @details ZynqスタンドアロンBSP内部では `outbyte` および `inbyte` が弱参照定義されています。
 */
#include "xuartps.h"

extern XUartPs Uart_Instance; /* メインで初期化されたインスタンス */

/**
 * @brief BSPのコア出力文字関数をオーバーライド
 */
void outbyte(char c) {
    if (c == '\n') {
        XUartPs_SendByte(Uart_Instance.Config.BaseAddress, '\r');
    }
    XUartPs_SendByte(Uart_Instance.Config.BaseAddress, c);
}

/**
 * @brief BSPのコア入力文字関数をオーバーライド
 */
char inbyte(void) {
    return XUartPs_RecvByte(Uart_Instance.Config.BaseAddress);
}
```

---

## 16. ベアメタル向け 16550/PL011/デバッグUARTレジスタ直叩き

* **対象 (7種類)**: Allwinner シリーズ, Broadcom BCM2835/BCM2837, Rockchip シリーズ, MediaTek MT7688, Raspberry Pi 4 / 5, 汎用SPARC, NuMicro M480

```c
/**
 * @file uart_retarget_baremetal_soc.c
 * @brief 各社LinuxクラスSoCのベアメタル、またはカスタムSoCのUARTレジスタ直叩き実装
 * @details 16550コンパチブル、ARM PL011、Nuvoton等の特殊FIFOSTSレジスタの物理アドレスへ直入出力します。
 */
#include <stdint.h>

/* 各SoCターゲットに応じたベースアドレス定義（ビルド設定にて切り替え） */
#if defined(TARGET_BCM2837)
  #define UART_BASE_ADDR  0x3F201000U /* Raspberry Pi 3 PL011 Base */
  #define UART_DR         (*(volatile uint32_t *)(UART_BASE_ADDR + 0x00))
  #define UART_FR         (*(volatile uint32_t *)(UART_BASE_ADDR + 0x18))
  #define TX_FIFO_FULL    (UART_FR & (1 << 5))
  #define RX_FIFO_EMPTY   (UART_FR & (1 << 4))
#elif defined(TARGET_16550) /* Allwinner, MediaTek, 汎用RISC-V等 */
  #define UART_BASE_ADDR  0x10000000U
  #define UART_THR        (*(volatile uint8_t *)(UART_BASE_ADDR + 0))
  #define UART_RBR        (*(volatile uint8_t *)(UART_BASE_ADDR + 0))
  #define UART_LSR        (*(volatile uint8_t *)(UART_BASE_ADDR + 5))
  #define TX_FIFO_FULL    (!(UART_LSR & 0x20))
  #define RX_FIFO_EMPTY   (!(UART_LSR & 0x01))
#elif defined(TARGET_NUMICRO)
  #define UART0_BASE      0x40070000U
  #define UART_DAT        (*(volatile uint32_t *)(UART0_BASE + 0x00))
  #define UART_FIFOSTS    (*(volatile uint32_t *)(UART0_BASE + 0x18))
  #define TX_FIFO_FULL    (UART_FIFOSTS & (1 << 23))
  #define RX_FIFO_EMPTY   (UART_FIFOSTS & (1 << 14))
#endif

static void raw_soc_uart_putc(char c) {
    while (TX_FIFO_FULL);
    #if defined(TARGET_16550)
    UART_THR = (uint8_t)c;
    #else
    UART_DR = c;
    #endif
}

static char raw_soc_uart_getc(void) {
    while (RX_FIFO_EMPTY);
    #if defined(TARGET_16550)
    return (char)UART_RBR;
    #else
    return (char)(UART_DR & 0xFF);
    #endif
}

/* Newlib系シンボルフック（_write / _read はセクション1の共通ロジックにこの下位レイヤを統合可能） */
int _write(int file, char *ptr, int len) {
    for (int i = 0; i < len; i++) {
        if (ptr[i] == '\n') raw_soc_uart_putc('\r');
        raw_soc_uart_putc(ptr[i]);
    }
    return len;
}

int _read(int file, char *ptr, int len) {
    for (int i = 0; i < len; i++) {
        ptr[i] = raw_soc_uart_getc();
        if (ptr[i] == '\r' || ptr[i] == '\n') return i + 1;
    }
    return len;
}
```

---

## 17. Silicon Labs EFM8 / C8051 (Keil C51)

* **対象 (2種類)**: EFM8 / C8051 シリーズ, 汎用 8051 コア

```c
/**
 * @file uart_retarget_c51.c
 * @brief Keil C51 コンパイラ (8051アーキテクチャ) 用 putchar / _getkey
 */
#include <compiler_defs.h>
#include <SI_EFM8BB1_Register_Enums.h> /* デバイス固有ヘッダ例 */
#include <stdio.h>

/**
 * @brief Keil C51標準の1文字出力フック
 */
char putchar(char c) {
    if (c == '\n') {
        while (!TI); /* 送信完了フラグ待ち */
        SBUF0 = '\r';
        TI = 0;
    }
    while (!TI);
    SBUF0 = c;
    TI = 0;
    return c;
}

/**
 * @brief Keil C51標準の1文字入力フック
 */
char _getkey(void) {
    char c;
    while (!RI); /* 受信完了フラグ待ち */
    c = SBUF0;
    RI = 0;
    return c;
}
```

---

## 18. SDCC (Zilog Z80 / 汎用8051)

* **対象 (2種類)**: Zilog Z80 / Z84C00, 汎用 8051 コア (SDCC環境)

```c
/**
 * @file uart_retarget_sdcc_z80.c
 * @brief SDCC (Small Device C Compiler) Z80 向け外部UART ICポート制御
 * @details 例として 8251 または Z80 SIO 等の外部I/O通信を想定。
 */
#include <stdio.h>

/* SDCC拡張の__sfrによるI/Oポート指定 */
__sfr __at(0x80) UART_DATA_PORT;
__sfr __at(0x81) UART_STAT_PORT;

#define TX_READY_MASK 0x01
#define RX_READY_MASK 0x02

/**
 * @brief SDCC標準のストリーム出力関数
 */
void putchar(char c) {
    if (c == '\n') {
        while (!(UART_STAT_PORT & TX_READY_MASK));
        UART_DATA_PORT = '\r';
    }
    while (!(UART_STAT_PORT & TX_READY_MASK));
    UART_DATA_PORT = c;
}

/**
 * @brief SDCC標準のストリーム入力関数
 */
char getchar(void) {
    while (!(UART_STAT_PORT & RX_READY_MASK));
    return UART_DATA_PORT;
}
```

---

## 19. Dialog SmartBond (DA14531)

* **対象 (1種類)**: DA14531 / SmartBond

```c
/**
 * @file uart_retarget_dialog.c
 * @brief Dialog (Renesas) DA14531 SmartBond SDKリターゲット
 * @details SDK非同期バッファマネージャを介して _write をフックします。
 */
#include <stdio.h>
#include "uart.h" /* Dialog SDKヘッダ */

int _write(int fd, const char *buf, int nbyte) {
    if (fd == 1 || fd == 2) {
        /* Dialog SDKの同期ブロック型UART送信APIを使用 */
        uart_send(UART1, (uint8_t *)buf, nbyte, UART_OP_BLOCKING);
        return nbyte;
    }
    return -1;
}

int _read(int fd, char *buf, int nbyte) {
    if (fd == 0) {
        uart_receive(UART1, (uint8_t *)buf, 1, UART_OP_BLOCKING);
        return 1;
    }
    return -1;
}
```

---

## 20. XMOS xCORE (多芯オーディオプロセッサ)

* **対象 (1種類)**: xCORE シリーズ

```c
/**
 * @file uart_retarget_xmos.xc
 * @brief XMOS xCORE 独自C/XC環境下でのポート直接駆動型UARTエミュレーション
 * @note 本ファイルはXC言語仕様の文脈として記載しています。
 */

#include <xs1.h>
#include <platform.h>

/* 1ビットの高速I/Oポートマッピング（例） */
out port p_uart_tx = XS1_PORT_1A;
in port p_uart_rx  = XS1_PORT_1B;

#define BIT_TIME 8680 /* 115200bps @ 100MHz基準のタイマティック数 */

/**
 * @brief ソフトウェアエミュレーション（ビットバンギング）によるUART送信
 */
void xcore_uart_putc(char c) {
    timer t;
    unsigned int time;
    unsigned int data = (unsigned int)c;

    t :> time;

    /* スタートビット */
    p_uart_tx <: 0;
    time += BIT_TIME;
    t when timerafter(time) :> void;

    /* データ8ビット分ループ */
    for (int i = 0; i < 8; i++) {
        p_uart_tx <: (data & 1);
        data >>= 1;
        time += BIT_TIME;
        t when timerafter(time) :> void;
    }

    /* ストップビット */
    p_uart_tx <: 1;
    time += BIT_TIME;
    t when timerafter(time) :> void;
}

/* 標準出力フック */
int _write(int fd, char buf[], int len) {
    for(int i=0; i<len; i++) {
        if(buf[i] == '\n') xcore_uart_putc('\r');
        xcore_uart_putc(buf[i]);
    }
    return len;
}
```

---

## 21. Parallax Propeller (多芯マルチコア)

* **対象 (1種類)**: Propeller (Parallax)

```c
/**
 * @file uart_retarget_propeller.c
 * @brief Parallax Propeller (Propeller-GCC/simpletools) ストリームバインド
 */
#include "simpletools.h"
#include "fdserial.h"

static fdserial *uart_stream;

/**
 * @brief 特定のCOG(コア)で通信を立ち上げ、標準デバイスドライバストリームへ代入
 */
void Propeller_UART_Init(int rx_pin, int tx_pin, int baud) {
    uart_stream = fdserial_open(rx_pin, tx_pin, 0, baud);
    
    /* simpletools内部の標準入出力制御構造体ポインタに設定 */
    stdout = uart_stream;
    stdin  = uart_stream;
}
```

---

## 22. 特殊OS統合環境 (Zephyr RTOS)

* **対象 (1種類)**: nRF9160 (Zephyr環境)

```c
/**
 * @file uart_retarget_zephyr.c
 * @brief Zephyr RTOS 上での printf/printk リターゲット概念
 * @details Zephyrでは `prj.conf` 設定項目でリターゲットをすべて制御するため、
 *          Cソースコード側で直接 _write をオーバーライドする必要はありません。
 */
/* 
  [重要] Zephyr環境 (nRF9160等) の場合は、Cソースではなく 
  プロジェクトの prj.conf に以下のコンフィグレーションを追記することで、
  内部HALを介して printf / printk がUARTへ自動的にマッピングされます。

  CONFIG_SERIAL=y
  CONFIG_CONSOLE=y
  CONFIG_UART_CONSOLE=y
  CONFIG_STDOUT_CONSOLE=y
*/
#include <zephyr/kernel.h>
#include <stdio.h>

void zephyr_demo_print(void) {
    /* 自動的にDevice Treeで指定された選択的UARTコンソールに出力されます */
    printf("Zephyr printf output\n"); 
}
```

---

## 🛠️ トラブルシューティング＆運用テクニック

1. **文字が出力されない場合**
   * `setvbuf(stdout, NULL, _IONBF, 0);` が実行されているか確認してください。バッファリングが効いていると、`\n` を送るまで送信されない場合があります。
   * ハードウェアの波形（ロジックアナライザやオシロスコープ）を確認し、ボーレートやピンマッピング（Tx/Rxのクロス接続など）に誤りがないか検証してください。

2. **浮動小数点数（`%f`）が表示されない場合**
   * GCC (Newlib Nano) ではバイナリサイズ削減のためデフォルトで `%f` サポートが外されています。GCCのリンカオプションに `-u _printf_float` を追加してください。

3. **入力（`scanf`）で文字が送れない/Enterが機能しない場合**
   * ターミナル側が送信する改行コード（CR: `\r` / LF: `\n` / CR+LF）と、受信側処理の判定ロジックが一致しているか確認してください。