# Air Sensor Node - STM32F030C8T6

Sensor node sử dụng **STM32F030C8T6** để đọc nhiều cảm biến qua I2C và truyền dữ liệu về Gateway bằng **Modbus RTU qua RS-485**.

## 1. Hardware Configuration

### MCU

- MCU: STM32F030C8T6
- HSE: 8 MHz
- System Clock: 48 MHz

### I2C1

| Signal | STM32 Pin |
|---|---|
| SCL | PB6 |
| SDA | PB7 |

Các cảm biến dùng chung bus I2C:

| Sensor | I2C Address | Dữ liệu |
|---|---:|---|
| BH1750 | `0x23` | Light / Lux |
| BMP280 | `0x76` | Temperature + Pressure |

Do mỗi cảm biến có I2C Address khác nhau nên có thể kết nối chung PB6/PB7.

---

# 2. RS-485 / Modbus RTU

STM32 giao tiếp với RS-485 transceiver bằng USART2.

| Signal | STM32 Pin |
|---|---|
| USART2_TX | PA2 |
| USART2_RX | PA3 |
| RS485_DE/RE | PA4 |

PA4 điều khiển đồng thời chân `DE` và `/RE` của RS-485 transceiver.

```text
PA4 = 0 -> Receive
PA4 = 1 -> Transmit
```

## UART Configuration

```text
Baud Rate : 9600
Data Bits : 8
Parity    : None
Stop Bits : 1
Format    : 8N1
```

Modbus hiện hỗ trợ:

```text
Function 0x04 - Read Input Registers
```

---

# 3. Hardware Slave ID

Slave ID **không được cố định trong firmware**.

Khi STM32 khởi động, firmware đọc 3 chân cấu hình phần cứng để xác định Slave ID.

## Slave ID Pins

| Bit | STM32 Pin | CubeMX User Label |
|---|---|---|
| Bit 2 - MSB | PB2 | `SLAVE_ID_2` |
| Bit 1 | PA10 | `SLAVE_ID_1` |
| Bit 0 - LSB | PA11 | `SLAVE_ID_0` |

Trong STM32CubeMX, cấu hình cả 3 chân:

```text
GPIO Mode         : Input mode
GPIO Pull-up/down : No pull-up and no pull-down
```

PCB đã có điện trở kéo cứng nên không sử dụng Pull-up/Pull-down nội của STM32.

---

# 4. Cách hàn trở để cấu hình Slave ID

Quy ước mức logic:

```text
0 -> điện trở kéo chân xuống GND
1 -> điện trở kéo chân lên 3.3V
```

Thứ tự bit:

```text
        MSB             LSB
         |               |
        PB2     PA10    PA11
      ID_BIT2  ID_BIT1  ID_BIT0
```

Slave ID được tính:

```text
Slave ID = (PB2 << 2) | (PA10 << 1) | PA11
```

## Bảng cấu hình

| Slave ID | PB2 | PA10 | PA11 | Binary |
|---:|:---:|:---:|:---:|:---:|
| 1 | 0 | 0 | 1 | `001` |
| 2 | 0 | 1 | 0 | `010` |
| 3 | 0 | 1 | 1 | `011` |
| 4 | 1 | 0 | 0 | `100` |
| 5 | 1 | 0 | 1 | `101` |
| 6 | 1 | 1 | 0 | `110` |
| 7 | 1 | 1 | 1 | `111` |

## Ví dụ: cấu hình Slave ID = 2

Cần cấu hình:

```text
PB2  = 0
PA10 = 1
PA11 = 0
```

Tương ứng:

```text
PB2  ---- R ---- GND

PA10 ---- R ---- 3.3V

PA11 ---- R ---- GND
```

Kết quả:

```text
PB2 PA10 PA11
 0    1    0

010 binary = 2 decimal
```

Firmware sẽ đọc:

```text
Modbus_SlaveID = 2
```

> Sau khi thay đổi cấu hình điện trở Slave ID, cần reset hoặc cấp nguồn lại board để firmware đọc ID mới.

### Lưu ý về ID 0

```text
000 = 0
```

Địa chỉ `0` không được sử dụng làm địa chỉ slave thông thường trong Modbus.

Firmware hiện tại xử lý trường hợp đọc được `000` bằng cách sử dụng:

```text
Slave ID = 1
```

---

# 5. Modbus Register Map

Mỗi giá trị cảm biến hiện được lưu trong **một Input Register 16-bit**.

| Register Address | Sensor | Value | Scale | Unit |
|---:|---|---|---:|---|
| `0x0000` | BH1750 | Light | `/ 10` | lux |
| `0x0001` | BMP280 | Temperature | `/ 100` | °C |
| `0x0002` | BMP280 | Pressure | `/ 10` | hPa |

Tất cả được đọc bằng:

```text
Modbus Function 0x04
Read Input Registers
```

---

# 6. BH1750 - Light

BH1750 có địa chỉ:

```text
I2C Address = 0x23
```

Firmware đọc giá trị lux dạng float, ví dụ:

```text
162.5 lux
```

Trước khi đưa vào Modbus:

```text
162.5 × 10 = 1625
```

Register:

```text
0x0000 = 1625
```

Gateway nhận `1625` và tính:

```text
Lux = Register[0] / 10.0
```

Ví dụ:

```text
Register = 1625

1625 / 10 = 162.5 lux
```

---

# 7. BMP280 - Temperature

BMP280 có địa chỉ:

```text
I2C Address = 0x76
Chip ID     = 0x58
```

Temperature được lưu tại:

```text
Register 0x0001
```

Firmware scale:

```text
Temperature × 100
```

Ví dụ:

```text
Temperature = 24.44 °C

24.44 × 100 = 2444
```

Register:

```text
0x0001 = 2444
```

Gateway tính:

```text
Temperature = Register[1] / 100.0
```

Kết quả:

```text
2444 / 100 = 24.44 °C
```

---

# 8. BMP280 - Pressure

BMP280 driver trả pressure theo đơn vị:

```text
Pa
```

Ví dụ:

```text
100710 Pa
```

Đổi sang hPa:

```text
100710 Pa / 100 = 1007.10 hPa
```

Để lưu vừa một register 16-bit, firmware lưu:

```text
Pressure Register = Pressure(Pa) / 10
```

Ví dụ:

```text
100710 / 10 = 10071
```

Register:

```text
0x0002 = 10071
```

Gateway tính:

```text
Pressure(hPa) = Register[2] / 10.0
```

Kết quả:

```text
10071 / 10 = 1007.1 hPa
```

---

# 9. Đọc từng Register

Ví dụ board có:

```text
Slave ID = 2
```

Gateway có thể đọc riêng từng giá trị.

### BH1750

```text
Start Address = 0x0000
Quantity      = 1
Function      = 0x04
```

### BMP280 Temperature

```text
Start Address = 0x0001
Quantity      = 1
Function      = 0x04
```

### BMP280 Pressure

```text
Start Address = 0x0002
Quantity      = 1
Function      = 0x04
```

---

# 10. Đọc tất cả cảm biến trong một request

Do 3 register nằm liên tiếp:

```text
0x0000 -> BH1750 Lux
0x0001 -> BMP280 Temperature
0x0002 -> BMP280 Pressure
```

Gateway không cần gửi 3 request riêng.

Có thể đọc cả 3 bằng:

```text
Slave ID      = 2
Function      = 0x04
Start Address = 0x0000
Quantity      = 3
```

Request Modbus có cấu trúc:

```text
02 04 00 00 00 03 CRC_L CRC_H
```

Trong đó:

```text
02       Slave ID = 2
04       Read Input Registers

00 00    Start Address = 0x0000

00 03    Quantity = 3

CRC_L
CRC_H    Modbus CRC16
```

Slave sẽ trả:

```text
02 04 06 XX XX XX XX XX XX CRC_L CRC_H
      |
      +-- 6 data bytes = 3 registers × 2 bytes
```

Data được sắp xếp:

```text
02 04 06 | REG0_H REG0_L | REG1_H REG1_L | REG2_H REG2_L | CRC
             |                |                |
             |                |                +-- Pressure
             |                |
             |                +------------------- Temperature
             |
             +------------------------------------ Lux
```

---

# 11. Ví dụ frame thực tế

Ví dụ Slave trả:

```text
01 04 06 06 40 09 8C 27 57 F9 43
```

Phân tích:

```text
01       Slave ID

04       Function 04

06       6 byte data

06 40    Register 0
09 8C    Register 1
27 57    Register 2

F9 43    CRC
```

## Register 0 - BH1750

```text
0x0640 = 1600 decimal

1600 / 10 = 160.0 lux
```

## Register 1 - BMP280 Temperature

```text
0x098C = 2444 decimal

2444 / 100 = 24.44 °C
```

## Register 2 - BMP280 Pressure

```text
0x2757 = 10071 decimal

10071 / 10 = 1007.1 hPa
```

Kết quả:

```text
Light       = 160.0 lux
Temperature = 24.44 °C
Pressure    = 1007.1 hPa
```

---

# 12. Cách hoạt động của Sensor Node

Gateway **không trực tiếp đọc cảm biến I2C**.

STM32 tự đọc cảm biến định kỳ và cập nhật Register Map:

```text
BH1750 0x23 -----+
                 |
                 +---- I2C1 ---- STM32
                 |
BMP280 0x76 -----+
                         |
                         v
               Modbus Input Registers

               0x0000 = Light
               0x0001 = Temperature
               0x0002 = Pressure
                         |
                         v
                     Modbus RTU
                         |
                         v
                       RS-485
                         |
                         v
                      Gateway
```

Khi Gateway gửi request Modbus, STM32 chỉ lấy giá trị mới nhất đang lưu trong các register và trả về.

Điều này giúp response Modbus nhanh và không phải chờ cảm biến I2C thực hiện phép đo tại thời điểm Gateway request.

---

# 13. Quy ước khi thêm cảm biến mới

Mỗi giá trị cảm biến nên được thêm vào Register Map.

Ví dụ:

```text
0x0000 -> Light
0x0001 -> Temperature
0x0002 -> Pressure
0x0003 -> Humidity
0x0004 -> CO2
0x0005 -> TVOC
...
```

Nếu giá trị sau khi scale nằm trong khoảng của 16-bit thì dùng một register:

```text
1 Register = 16 bit = 2 byte
```

Ví dụ:

```text
25.36 °C
   |
   ×100
   |
 2536
   |
uint16_t
   |
1 Modbus Register
```

Nếu một giá trị cần 32 bit thì phải sử dụng **2 register liên tiếp**.

Ví dụ:

```text
uint32_t / float 32-bit

Register N
Register N+1
```

Khi thêm dữ liệu mới cần cập nhật đồng thời:

1. `MODBUS_INPUT_REG_COUNT`
2. Địa chỉ register trong `modbus_slave.h`
3. Code cập nhật register
4. Bảng Register Map trong README này
5. Code Gateway để decode đúng scale

---

# 14. Register Definitions trong Firmware

Register map hiện tại:

```c
#define MODBUS_REG_BH1750_LUX       0x0000
#define MODBUS_REG_BMP280_TEMP      0x0001
#define MODBUS_REG_BMP280_PRESSURE  0x0002

#define MODBUS_INPUT_REG_COUNT      3
```

Cập nhật dữ liệu:

```c
Modbus_SetInputRegister(
    MODBUS_REG_BH1750_LUX,
    (uint16_t)(bh1750_lux * 10.0f)
);
```

```c
Modbus_SetInputRegister(
    MODBUS_REG_BMP280_TEMP,
    (uint16_t)(bmp280_temperature * 100.0f)
);
```

```c
Modbus_SetInputRegister(
    MODBUS_REG_BMP280_PRESSURE,
    (uint16_t)(bmp280_pressure / 10.0f)
);
```

---

# 15. Tóm tắt cho Gateway

Gateway cần biết 4 thông tin:

```text
Protocol : Modbus RTU
Baud     : 9600
Format   : 8N1
Function : 0x04
```

Register map:

```text
0x0000 -> Light       -> value / 10   -> lux
0x0001 -> Temperature -> value / 100  -> °C
0x0002 -> Pressure    -> value / 10   -> hPa
```

Để đọc toàn bộ dữ liệu hiện tại:

```text
Start Address = 0x0000
Quantity      = 3
```

Slave ID được xác định bằng cấu hình điện trở trên PCB:

```text
PB2  = ID bit 2
PA10 = ID bit 1
PA11 = ID bit 0
```

Ví dụ:

```text
PB2 PA10 PA11 = 010
              = Slave ID 2
```
