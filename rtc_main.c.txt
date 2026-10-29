/*
 * Bit-banged I2C + DS1307 RTC on Arduino Mega
 * PORTF: PF0 = SDA, PF1 = SCL
 * DDRF  = 0x30, PORTF = 0x31, PINF = 0x2F (data-space addresses)
 */

void I2C_start();
void I2C_write(char);
void I2C_delay();
void I2C_stop();
char I2C_read();
void RTC_write(char, char);
char RTC_read(char);

void setup() {
  Serial.begin(9600);

  // Set initial time (BCD): 23:59:28
  RTC_write(0x00, 0x28);
  RTC_write(0x01, 0x59);
  RTC_write(0x02, 0x23);

  while (1)
  {
    char second = RTC_read(0x00);
    char minute = RTC_read(0x01);
    char hour   = RTC_read(0x02);

    Serial.print("Time :");
    Serial.print(hour, HEX);
    Serial.print("/");
    Serial.print(minute, HEX);
    Serial.print("/");
    Serial.println(second, HEX);

    delay(1000);
  }
}

void RTC_write(char address, char data)
{
  I2C_start();
  I2C_write(0xD0);      // DS1307 address + write
  I2C_write(address);
  I2C_write(data);
  I2C_stop();
}

char RTC_read(char address)
{
  I2C_start();
  I2C_write(0xD0);      // DS1307 address + write
  I2C_write(address);
  I2C_start();          // repeated start
  I2C_write(0xD1);      // DS1307 address + read
  volatile char data = I2C_read();
  I2C_stop();
  return data;
}

void I2C_start()
{
  volatile char *portf_dir = (volatile char*)0x30;
  *portf_dir = 0x03;
  volatile char *portf_out = (volatile char*)0x31;
  *portf_out = 0x03;
  I2C_delay();
  *portf_out = 0x02;    // SDA low while SCL high
  I2C_delay();
  *portf_out = 0x00;    // SCL low
}

void I2C_write(char data)
{
  volatile char *portf_dir = (volatile char*)0x30;
  volatile char *portf_out = (volatile char*)0x31;
  volatile char k, a;

  for (k = 7; k >= 0; k--)
  {
    a = (data >> k) & 0x01;
    if (a == 1)
    {
      *portf_out |= 0x01;
    }
    else
    {
      *portf_out &= ~0x01;
    }
    I2C_delay();
    *portf_out |= 0x02;     // SCL high
    I2C_delay();
    *portf_out &= ~0x02;    // SCL low
  }

  // ACK bit: release SDA (input), clock once
  *portf_dir &= ~0x01;
  I2C_delay();
  *portf_out |= 0x02;
  I2C_delay();
  *portf_out &= ~0x02;
  *portf_dir |= 0x01;
}

char I2C_read()
{
  volatile char *portf_dir = (volatile char*)0x30;
  *portf_dir &= ~0x01;      // SDA as input
  volatile char *portf_out = (volatile char*)0x31;
  volatile char *portf_in  = (volatile char*)0x2F;
  volatile char k, data = 0x00;

  for (k = 0; k < 8; k++)
  {
    *portf_out &= ~0x02;    // SCL low
    I2C_delay();
    *portf_out |= 0x02;     // SCL high
    I2C_delay();
    data = (data << 1) | (*portf_in & 0x01);
  }

  // Send NACK
  *portf_dir |= 0x01;
  *portf_out |= 0x01;
  *portf_out &= ~0x02;
  I2C_delay();
  *portf_out |= 0x02;
  I2C_delay();
  *portf_out = 0x00;

  return data;
}

void I2C_stop()
{
  volatile char *portf_dir = (volatile char*)0x30;
  *portf_dir |= 0x01;
  volatile char *portf_out = (volatile char*)0x31;
  *portf_out |= 0x02;       // SCL high
  I2C_delay();
  *portf_out |= 0x01;       // SDA high while SCL high = STOP
  I2C_delay();
}

void I2C_delay()
{
  volatile char i;
  for (i = 0; i < 5; i++);
}

void loop() {
}
