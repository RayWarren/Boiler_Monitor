/* vers .1
Program to monitor the status of water temperatures out of a a hydronic heating boiler and the
temerature of the water returning to the boiler from 2 loops and the outside temperature using DS18B20 sensors ,the status of the relays for the thermostat and burner gas valve using MID400-D
opto isolator chips and the boiler pressure using a Walfront6byhrcuzok 30psi pressure transducer.
The measured values are displayed on a 20x4 lcd and sent to a local mariadb database.The MySQL connector library was modified in Cursor.cpp,the close and execute_cmd functions, to send an explicit quit ,command 1, to the database to avoid database logged errors from a dropped connection.Most of the code is from various examples just fitted togethor.The sampling interval is controlled by the 1square wave output from a DS3231 clock module,the time is displayed  but not sent the database  ,the timestamp from thedatabase is used instead for data storage.the display is optional and a timing signal could be provided by any externa square wave ckt 555 etc since the time is not used for anything but display.Internal timer was not used because of possible conflick with WiFi over long term .
  This program is free software; you can redistribute it and/or modify
  it under the terms of the GNU General Public License as published by
  the Free Software Foundation; version 2 of the License.

  This program is distributed in the hope that it will be useful,
  but WITHOUT ANY WARRANTY; without even the implied warranty of
  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  GNU General Public License for more details.

  You should have received a copy of the GNU General Public License
  along with this program; if not, write to the Free Software
  Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA 02110-1301  USA
*/
#include <Arduino.h>
#include <DallasTemperature.h>
#include <WiFiNINA.h>
#include <Wire.h>
#include <MySQL_Connection.h>
#include <MySQL_Cursor.h>
#include <RTClib.h>
#include <LiquidCrystal_I2C.h>

// lcd stuff
LiquidCrystal_I2C lcd(PCF8574_ADDR_A21_A11_A01, 4, 5, 6, 16, 11, 12, 13, 14, POSITIVE);
#define columns 20
#define rows 4
// sensor processing
int displaycount;
int timecount;
// pressure
int boilerPin = A0;
int BoilerReading;
float sensorvoltage;
float boilerpressure;
float conversionfactor = .00488; // for 5V vcc
float scaleamount = 7.5;         // psi/volt of sensor output
float offset = .44;              // minimum pressure sensor V

// temperature
#define thermostat_mon 6 // pin for thermostat monitor
#define burner_mon 7

float outputtemp;
float bigreturntemp;
float smallreturntemp;
float outsidetemp;

// charbuffers for query
char outputdata[7];
char bigreturndata[7];
char smallreturndata[7];
char outsidedata[7];
char burnerstate[3];
char Thermostaton[3];
char boilerdata[6];

uint8_t burner_status; // not boolean because of processing
uint8_t call_for_heat;

// clock and interrupts
const byte SQWinput = 8;
volatile int EDGE;
RTC_DS3231 rtc;
int sampletime = 0; // counter to control how often to read values
char daysOfTheWeek[7][12] = {"Sunday", "Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday"};

// Creating a oneWire instance(object) for each temp bus
OneWire ds18x20[] = {2, 3, 4, 5};
const int oneWireCount = sizeof(ds18x20) / sizeof(OneWire);
DallasTemperature sensor[oneWireCount];

// mysql database
IPAddress server_addr(192, 168, 1, 41);                                                                                                                // IP of the MySQL *server* here
char user[] = "name";                                                                                                                                  // MySQL user login username
char password[] = "MYOB";                                                                                                                              // MySQL user login password
char INSERT_DATA[] = "INSERT INTO test1.tankpressure (TankP,tempout,longloop,shortloop,outside,burner_status,Thermostat)VALUES(%s,%s,%s,%s,%s,%s,%s)"; // actual values match to database.table and columns or could add database to login and just specify table here
char query[128];                                                                                                                                       // large enough for complete query text
char MYSQL_QUIT[] = "quit";                                                                                                                            // my mariahdb grumbles if you don't say goodbye

// WiFi card
char ssid[] = "xxxxxxxxxx"; // your SSID
char pass[] = "scrambled";  // your SSID Password}
int status = WL_IDLE_STATUS;
byte mac_addr[] = {0xff, 0xff, 0xff, 0xff, 0xff, 0xff}; // use the real one,not sure it matters
WiFiClient client;                                      // Use this for WiFi instead of EthernetClient
MySQL_Connection conn((Client *)&client);
IPAddress ip;
//_______________________________________________________________________________________________________________________________
// Interrupt Service Routine - This routine is performed when a falling edge on the 1Hz SQW clock from the RTC is detected
void Isr()
{
  EDGE = 0; // A falling edge was detected on the SQWinput pin.  Now set EDGE equal to 0.
}

//_______________________________________________________
void displayTime()
{
  DateTime now = rtc.now();

  /*Serial.print(now.year(), DEC);
  Serial.print('/');
  Serial.print(now.month(), DEC);
  Serial.print('/');
  Serial.print(now.day(), DEC);
  Serial.print("  ");
  Serial.print(now.hour(), DEC);
  Serial.print(':');
  Serial.print(now.minute(), DEC);
  Serial.print(':');
  Serial.print(now.second(), DEC);
  Serial.println();
*/
  lcd.setCursor(0, 3);
  lcd.print(now.year(), DEC);
  lcd.print('/');
  lcd.print(now.month(), DEC);
  lcd.print('/');
  lcd.print(now.day(), DEC);
  lcd.print("  ");
  lcd.print(now.hour(), DEC);
  lcd.print(':');
  lcd.print(now.minute(), DEC);
  lcd.print(':');
  lcd.print(now.second(), DEC);
}
//__________________________________________________________________
void chkheat()
{
  // temperatures of boiler ,return temp of both circulation loops ,and outside air temp
  for (int i = 0; i < oneWireCount; i++)
  {
    sensor[i].requestTemperatures();
  }
  delay(1000);
  outputtemp = sensor[0].getTempFByIndex(0);
  bigreturntemp = sensor[1].getTempFByIndex(0);
  smallreturntemp = sensor[2].getTempFByIndex(0);
  outsidetemp = sensor[3].getTempFByIndex(0);

  // convert floats to string for mariadb query, left justified nnn.n format
  dtostrf(outputtemp, -6, 1, outputdata);
  dtostrf(bigreturntemp, -6, 1, bigreturndata);
  dtostrf(smallreturntemp, -6, 1, smallreturndata);
  dtostrf(outsidetemp, -6, 1, outsidedata);

  // local temperature display
  lcd.setCursor(0, 0);
  lcd.print("Boiltemp = ");
  lcd.print(outputdata);

  lcd.setCursor(0, 1);
  lcd.print("Maintmp = ");
  lcd.print(bigreturndata);

  lcd.setCursor(0, 2);
  lcd.print("bdrtmp = ");
  lcd.print(smallreturndata);

  lcd.setCursor(0, 3);
  lcd.print("ext: ");
  lcd.print(outsidedata);
  delay(1000);

  //  get the status of the thermostat and burner control on/off
  burner_status = (digitalRead(burner_mon));
  call_for_heat = (digitalRead(thermostat_mon));

// on/off status format is n
  dtostrf(burner_status, 2, 0, burnerstate);
  dtostrf(call_for_heat, 2, 0, Thermostaton);

  /*
  Serial.print("burner status = ");
  Serial.println(burnerstate);

  Serial.print("thermostat = ");
  Serial.println(Thermostaton);
  */

  lcd.setCursor(17, 0);
  lcd.print("B:");
  lcd.print(burnerstate);
  lcd.setCursor(17, 1);
  lcd.print("T:");
  lcd.print(Thermostaton);
}
//_______________________________________________________________________
void chkpressure()
{
  BoilerReading = analogRead(boilerPin);
  sensorvoltage = (BoilerReading * conversionfactor);
  boilerpressure = (sensorvoltage - offset) * scaleamount;
 
 // pressure format is nn.nn
  dtostrf(boilerpressure, 6, 2, boilerdata);
  lcd.setCursor(0, 3);
  lcd.print("bp = ");
  lcd.print(boilerpressure);
  delay(1000);
}
//____________________________________________________________________________________
// if the wifi dropped out reconnect it
void TestWiFiConnection()
{
  int status = WiFi.status();
  //Serial.print("wifi status  = ");
  //Serial.println(status);

  while (status != WL_CONNECTED)
  {
    //Serial.print("Attempting to reconnect to Network named: ");
    //Serial.println(ssid);
    //Serial.println(pass);
    status = WiFi.begin(ssid, pass);
    delay(3000);
  }
  /* print out info about the connection:for troubleshooting

  Serial.println("Connected to network");
  ip = WiFi.localIP();
  Serial.print("My IP address is: ");
  Serial.println(ip);
  */
}

/*
//useful for testing 
int get_free_memory()
{
  extern char __bss_end;
  extern char *__brkval;
  int free_memory;
  if ((int)__brkval == 0)
    free_memory = ((int)&free_memory) - ((int)&__bss_end);
  else
    free_memory = ((int)&free_memory) - ((int)__brkval);
  return free_memory;
}
*/

void setup()
{
  Serial.begin(57600);  
  // set up the interrupt
  rtc.begin();
  rtc.writeSqwPinMode(DS3231_SquareWave1Hz);

  pinMode(SQWinput, INPUT_PULLUP); // Configure the SQWinput pin as an INPUT to monitor the SQW pin of the DS3231.

  attachInterrupt(digitalPinToInterrupt(SQWinput), Isr, FALLING); // Configure SQWinput (pin 2 of the Arduino) for use by the Interrupt Service Routine (Isr)
  EDGE = 1;

  lcd.begin(columns, rows);
  lcd.setCursor(0, 0);
  lcd.clear();

  // Start up the onewire on all defined bus-wires
  DeviceAddress deviceAddress;
  Serial.print("============Ready with ");
  Serial.print(oneWireCount);
  Serial.println(" Sensors================");
  for (int i = 0; i < oneWireCount; i++)
  {
    // begin one wire
    //  set sensors to 10 bit resolution
    sensor[i].setOneWire(&ds18x20[i]);
    sensor[i].begin();
    if (sensor[i].getAddress(deviceAddress, 0))
      sensor[i].setResolution(deviceAddress, 10);
  }
  pinMode(thermostat_mon, INPUT_PULLUP);
  pinMode(burner_mon, INPUT_PULLUP);

  // Begin WiFi section
  // int status = WiFi.begin(ssid, pass);
  while (status != WL_CONNECTED)
  {
    status = WiFi.begin(ssid, pass);
    delay(3000);
  }
  // print out info about the connection:

  //Serial.println("Connected to network");
  IPAddress ip = WiFi.localIP();
  //Serial.print("My IP address is: ");
  //Serial.println(ip);
  lcd.print(ip);      // maybe useful after power out

  //Serial.println("Connecting to mysql");

  if (conn.connect(server_addr, 3306, user, password))
  {
    Serial.print("got connection");
    delay(100);
  }
  else
    Serial.println("Connection failed.");

  delay(1000);
  displaycount = -721; // make sure there are initial readings to display
 
}

void loop()
{

  if (EDGE == 0)

  {
    // increment the interval counts
    displaycount++;
    sampletime++;

    if (displaycount > 120)
    {
      // local temperature display
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Boiltemp = ");
      lcd.print(outputdata);

      lcd.setCursor(16, 0);
      lcd.print("B:");
      lcd.print(burnerstate);

      lcd.setCursor(0, 1);
      lcd.print("maintmp = ");
      lcd.print(bigreturndata);

      lcd.setCursor(16, 1);
      lcd.print("T:");
      lcd.print(Thermostaton);

      lcd.setCursor(0, 2);
      lcd.print("bdrtmp = ");
      lcd.print(smallreturndata);

      lcd.setCursor(0, 3);
      lcd.print("ext: ");
      lcd.print(outsidedata);
      displaycount = 0;
      if (displaycount > 240)
        displaycount = 0;
    }
    else if (displaycount < 120)
    {
      lcd.clear();
      lcd.setCursor(0, 1);
      lcd.print("boiler press = ");
      lcd.print(boilerpressure);
      displayTime();
    }
    if (sampletime > 720) //~ every 12 min or 5 samples/hour
    {
      detachInterrupt(digitalPinToInterrupt(SQWinput)); //don't interrupt while sampling
      ip = WiFi.localIP();
      Serial.print("ip = ");
      Serial.println(ip);
      chkheat();
      chkpressure();
      char query[180]; // strings add up to 169 ,cushion or 1 more sensor
      sprintf(query, INSERT_DATA, boilerdata, outputdata, bigreturndata, smallreturndata, outsidedata, burnerstate, Thermostaton);
      
      TestWiFiConnection();
      delay(500);
      if (conn.connect(server_addr, 3306, user, password))
      {
        Serial.println("got connection");
        delay(200);
      }
      MySQL_Cursor *cur_mem = new MySQL_Cursor(&conn);
      delay(500);
      cur_mem->execute(query);
      delay(100);
      sampletime = 0; // reset the interval counter

      // Deleting the cursor also frees up memory used
      sprintf(query, MYSQL_QUIT);
      cur_mem->disconnect(MYSQL_QUIT);
      delete cur_mem;
      conn.close();
      // reattach the interrupt
      attachInterrupt(digitalPinToInterrupt(SQWinput), Isr, FALLING);
      sampletime = 0; // reset the sample interval

    }
    EDGE = 1; // reset the interrupt
  }
}