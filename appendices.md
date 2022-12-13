---
title: ""
layout: post
---

## Appendix A - Permissions

The group approves this report for inclusion on the course website.

The group approves the video for inclusion on the course youtube channel.

## Appendix B - Commented Code
```c
// Include standard libraries
#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include <string.h>
// Include Pico libraries
#include "pico/stdlib.h"
#include "pico/divider.h"
#include "pico/multicore.h"
#include "pico/binary_info.h"
// Include hardware libraries
#include "hardware/spi.h"
#include "hardware/gpio.h"
// Include protothreads
#include "pt_cornell_rp2040_v1.h"
// Include LCD libraries
#include "DEV_Config.h"
#include "GUI_Paint.h"
#include "Debug.h"
#include "Infrared.h"
#include "LCD_1in14.h"
#include "fonts.h"
// Include SD card third party library
#include "sd_card.h"
#include "ff.h"

// GPIO pins for the LCD screen
uint8_t keyA = 15; 
uint8_t keyB = 17; 
uint8_t left = 2; // originally up
uint8_t right = 18; // originally dowm
uint8_t dowm = 16; // originally left
uint8_t up = 20; // originally right
uint8_t ctrl = 3;

// Variables for the UI
#define MAX_LABEL_CHAR_SIZE 15
#define MAX_BUTTON_PAIRING 8
#define MAX_REMOTE_SIZE 10

int numDevices = 5;
int numButtonsDevice0 = 7;
int numButtonsDevice1 = 5;
int numButtonsDevice2 = 4;
int numButtonsDevice3 = 4;
int numButtonsDevice4 = 7;
int numButtonsDevice5 = 0;
int numButtonsDevice6 = 0;
int numButtonsDevice7 = 0;
int numButtonsDevice8 = 0;
int numButtonsDevice9 = 0;

char devices[MAX_REMOTE_SIZE][MAX_LABEL_CHAR_SIZE];
char nullDevices[MAX_LABEL_CHAR_SIZE];
char dev0Buttons[MAX_BUTTON_PAIRING][MAX_LABEL_CHAR_SIZE];
char dev1Buttons[MAX_BUTTON_PAIRING][MAX_LABEL_CHAR_SIZE];
char dev2Buttons[MAX_BUTTON_PAIRING][MAX_LABEL_CHAR_SIZE];
char dev3Buttons[MAX_BUTTON_PAIRING][MAX_LABEL_CHAR_SIZE];
char dev4Buttons[MAX_BUTTON_PAIRING][MAX_LABEL_CHAR_SIZE];
char dev5Buttons[MAX_BUTTON_PAIRING][MAX_LABEL_CHAR_SIZE];
char dev6Buttons[MAX_BUTTON_PAIRING][MAX_LABEL_CHAR_SIZE];
char dev7Buttons[MAX_BUTTON_PAIRING][MAX_LABEL_CHAR_SIZE];
char dev8Buttons[MAX_BUTTON_PAIRING][MAX_LABEL_CHAR_SIZE];
char dev9Buttons[MAX_BUTTON_PAIRING][MAX_LABEL_CHAR_SIZE];
char nullButtons[MAX_LABEL_CHAR_SIZE];
char buf[50];


int windowTop = 0;
int selectedItem = 0;
int displayState = 0;
int selectedSlot = 0;
int selectedDevice = 0;
UWORD *BlackImage;
int saveData = 0;


#define MAX_SIGNAL_SIZE 99
#define INPUT_IR_TIMEOUT_WINDOW_MS 500

#define IR_LED_PIN 21
#define IR_RECEIVER_PIN 19

bool volatile recieved = false; // Flag communicating fully recieved and encoded signal from reciever processor to Serial interface FSM

alarm_id_t timeout; // Timeout to detect end of reading signal

#define MAX_DATA_SIZE 400
uint32_t edges[MAX_SIGNAL_SIZE];
// edges stores every rising and falling edge time index
// Due to binary nature of communication, will always guarentee toggle at each index
// Index 0: Not a timestamp, rather an entry dictating the valid tail of the data structure
uint32_t dataArr[MAX_SIGNAL_SIZE];

// Structs that Liam uses to manipulate the buttons and remotes
typedef struct button{
  bool valid;
  char name[MAX_LABEL_CHAR_SIZE];
  uint32_t signal[MAX_SIGNAL_SIZE];
} button;

typedef struct remote{
  bool valid;
  char name[MAX_LABEL_CHAR_SIZE];
  button linkedButtons[MAX_BUTTON_PAIRING];
} remote;

remote remoteData[MAX_REMOTE_SIZE];

volatile static int runSerialInterface;


absolute_time_t timeoutWindow; //Manually induce secondary timer window

//////////////////////////////////
//      Reproduce Signal        //
//////////////////////////////////
//Note: Must be function call to prevent ISR-ISR locks
void broadcast_IR(uint32_t* edgeFlip)
{
    gpio_put(IR_LED_PIN, 0); // Make sure the system starts in a non-broadcasting state (LED Off)
    sleep_ms((uint64_t) 5); //Delay to make sure no accidental edge of the system
    if(edgeFlip[0] > MAX_SIGNAL_SIZE) {
      printf("ERROR: OUT OF BOUNDS\n");
      return;
    }
    for(int i = 1; i < edgeFlip[0]; i++) { //May need to implement instruction-execution-time bit-hacking
        sleep_us((uint64_t) edgeFlip[i]); //Wait specified interval
        gpio_put(IR_LED_PIN, !gpio_get(IR_LED_PIN)) ; //Toggle to create edge
    }
}



// ==================================================
// === String array assignment method for 2d arrays
// ==================================================
void string2DArrayEquals(int firstIndex, char firstArray[][MAX_LABEL_CHAR_SIZE], char secondArray[MAX_LABEL_CHAR_SIZE]){
  for(int index = 0; index < MAX_LABEL_CHAR_SIZE; index ++){
    firstArray[firstIndex][index] = secondArray[index];
  }
}

// ==================================================
// === String array assignment method for 2d arrays
// ==================================================
void string1DArrayEquals(char firstArray[MAX_LABEL_CHAR_SIZE], char secondArray[MAX_SIGNAL_SIZE]){
  for(int index = 0; index < MAX_LABEL_CHAR_SIZE; index ++){
    firstArray[index] = secondArray[index];
  }
}


// ==================================================
// === String array assignment method for 2d arrays
// ==================================================
void string1DArrayEqualsBuf(char firstArray[50], char secondArray[50]){
  for(int index = 0; index < 50; index ++){
    firstArray[index] = secondArray[index];
  }
}


// ==================================================
// === Convert Liams data to Owens data types
// ==================================================
void convertData(){
  static int ra = 0;
  static int ba = 0;
  static int b = 0;
  static int r = 0;
  // Make all entries null in case of a removed remote
  for(r=0; r<MAX_REMOTE_SIZE; r++){
    string2DArrayEquals(r, devices, nullDevices);
    string2DArrayEquals(r, dev0Buttons, nullButtons);
    string2DArrayEquals(r, dev1Buttons, nullButtons);
    string2DArrayEquals(r, dev2Buttons, nullButtons);
    string2DArrayEquals(r, dev3Buttons, nullButtons);
    string2DArrayEquals(r, dev4Buttons, nullButtons);
    string2DArrayEquals(r, dev5Buttons, nullButtons);
    string2DArrayEquals(r, dev6Buttons, nullButtons);
    string2DArrayEquals(r, dev7Buttons, nullButtons);
    string2DArrayEquals(r, dev8Buttons, nullButtons);
    string2DArrayEquals(r, dev9Buttons, nullButtons);
  }
  
  r=0;
  // Parse remotes and their buttons into the arrays
  for(r = 0; r<MAX_REMOTE_SIZE; r++){
    if(remoteData[r].valid){
      string2DArrayEquals(ra, devices, remoteData[r].name);
      // Extract buttons from remote's buttons
      for(b = 0; b < MAX_BUTTON_PAIRING; b++){
        if(remoteData[r].linkedButtons[b].valid){
          if(ra==0){
            string2DArrayEquals(ba, dev0Buttons, remoteData[r].linkedButtons[b].name);
          } else if(ra==1){
            string2DArrayEquals(ba, dev1Buttons, remoteData[r].linkedButtons[b].name);
          } else if(ra==2){
            string2DArrayEquals(ba, dev2Buttons, remoteData[r].linkedButtons[b].name);
          } else if(ra==3){
            string2DArrayEquals(ba, dev3Buttons, remoteData[r].linkedButtons[b].name);
          } else if(ra==4){
            string2DArrayEquals(ba, dev4Buttons, remoteData[r].linkedButtons[b].name);
          } else if(ra==5){
            string2DArrayEquals(ba, dev5Buttons, remoteData[r].linkedButtons[b].name);
          } else if(ra==6){
            string2DArrayEquals(ba, dev6Buttons, remoteData[r].linkedButtons[b].name);
          } else if(ra==7){
            string2DArrayEquals(ba, dev7Buttons, remoteData[r].linkedButtons[b].name);
          } else if(ra==8){
            string2DArrayEquals(ba, dev8Buttons, remoteData[r].linkedButtons[b].name);
          } else if(ra==9){
            string2DArrayEquals(ba, dev9Buttons, remoteData[r].linkedButtons[b].name);
          }
          // Increment actual button number
          ba++;
        }
        if(ra==0){
          numButtonsDevice0 = ba;
        } else if(ra==1){
          numButtonsDevice1 = ba;
        } else if(ra==2){
          numButtonsDevice2 = ba;
        } else if(ra==3){
          numButtonsDevice3 = ba;
        } else if(ra==4){
          numButtonsDevice4 = ba;
        } else if(ra==5){
          numButtonsDevice5 = ba;
        } else if(ra==6){
          numButtonsDevice6 = ba;
        } else if(ra==7){
          numButtonsDevice7 = ba;
        } else if(ra==8){
          numButtonsDevice8 = ba;
        } else if(ra==9){
          numButtonsDevice9 = ba;
        }
      }
      ba=0;
      // Increment actual device number
      ra++;
    }
  }
  numDevices = ra;
  ra=0;
}



// ==================================================
// === Highlight the correct item
// ==================================================
void highlightSelection(){
  if(selectedSlot == 0){
    // Display the correct line
    Paint_DrawLine(3,  48+0*48+4, 3, 48+1*48-4, RED, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
    // Erase the other lines
    Paint_DrawLine(3,  48+1*48+4, 3, 48+2*48-4, WHITE, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
    Paint_DrawLine(3,  48+2*48+4, 3, 48+3*48-4, WHITE, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
    Paint_DrawLine(3,  48+3*48+4, 3, 48+4*48-4, WHITE, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
  } else if(selectedSlot == 1){
    // Display the correct line
    Paint_DrawLine(3,  48+1*48+4, 3, 48+2*48-4, RED, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
    // Erase the other lines
    Paint_DrawLine(3,  48+0*48+4, 3, 48+1*48-4, WHITE, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
    Paint_DrawLine(3,  48+2*48+4, 3, 48+3*48-4, WHITE, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
    Paint_DrawLine(3,  48+3*48+4, 3, 48+4*48-4, WHITE, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
  } else if(selectedSlot == 2){
    // Display the correct line
    Paint_DrawLine(3,  48+2*48+4, 3, 48+3*48-4, RED, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
    // Erase the other lines
    Paint_DrawLine(3,  48+0*48+4, 3, 48+1*48-4, WHITE, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
    Paint_DrawLine(3,  48+1*48+4, 3, 48+2*48-4, WHITE, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
    Paint_DrawLine(3,  48+3*48+4, 3, 48+4*48-4, WHITE, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
  } else if(selectedSlot == 3){
    // Display the correct line
    Paint_DrawLine(3,  48+3*48+4, 3, 48+4*48-4, RED, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
    // Erase the other lines
    Paint_DrawLine(3,  48+0*48+4, 3, 48+1*48-4, WHITE, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
    Paint_DrawLine(3,  48+1*48+4, 3, 48+2*48-4, WHITE, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
    Paint_DrawLine(3,  48+2*48+4, 3, 48+3*48-4, WHITE, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
  }
}


// ==================================================
// === Scroll to the correct item
// ==================================================
void scroll(int buttonPressed, int numItems){
    // Determine the button pressed and update it
    if(buttonPressed == 2){ //If up, subtract unless you would go below zero
      if(selectedItem > 0){
        selectedItem --;
      }
      if(selectedSlot > 0){
        selectedSlot --;
      }
    } else if(buttonPressed == 3){ //If down, add unless you go above number of things
      if(selectedItem < numItems-1){
        selectedItem ++;
      }
      if(selectedSlot < 3){
        selectedSlot ++;
      }
    }
    // Handle scrolling too far down or up
    if(selectedItem > windowTop + 3){
      windowTop ++;
      selectedSlot = 3;
    } else if(selectedItem < windowTop){
      windowTop --;
      selectedSlot = 0;
    } 
}

// ==================================================
// === Run any menu
// ==================================================
void displayMenu(int numItems, char items[][MAX_LABEL_CHAR_SIZE], int buttonPressed){
  // ================= ERASE OLD LABELS
  Paint_DrawString_EN(10, 65, items[windowTop], &Font16, WHITE, WHITE);
  Paint_DrawString_EN(10, 65+1*48, items[windowTop+1], &Font16, WHITE, WHITE);
  Paint_DrawString_EN(10, 65+2*48, items[windowTop+2], &Font16, WHITE, WHITE);
  Paint_DrawString_EN(10, 65+3*48, items[windowTop+3], &Font16, WHITE, WHITE);

  // ================= DISPLAY THE SCROLLING
  if(buttonPressed == 2){
    scroll(2, numItems);
  } else if(buttonPressed == 3){
    scroll(3, numItems);
  }

  // ================= DISPLAY THE SELECTION LINE
  highlightSelection();

  // ================= DISPLAY THE SELECTIONS
  Paint_DrawString_EN(10, 65, items[windowTop], &Font16, WHITE, BLACK);
  Paint_DrawString_EN(10, 65+1*48, items[windowTop+1], &Font16, WHITE, BLACK);
  Paint_DrawString_EN(10, 65+2*48, items[windowTop+2], &Font16, WHITE, BLACK);
  Paint_DrawString_EN(10, 65+3*48, items[windowTop+3], &Font16, WHITE, BLACK);

  // ================= ERASE AGAIN
  if(buttonPressed == 7){
    Paint_DrawString_EN(10, 65, items[windowTop], &Font16, WHITE, WHITE);
    Paint_DrawString_EN(10, 65+1*48, items[windowTop+1], &Font16, WHITE, WHITE);
    Paint_DrawString_EN(10, 65+2*48, items[windowTop+2], &Font16, WHITE, WHITE);
    Paint_DrawString_EN(10, 65+3*48, items[windowTop+3], &Font16, WHITE, WHITE);
  }


}

// ==================================================
// === Dispaly the correct item
// ==================================================
void refreshDisplay(int buttonPressed){
  // Button 0 = A
  // Button 1 = B
  // Button 2 = up
  // Button 3 = down
  // Button 4 = left
  // Button 5 = right
  // Button 6 = center


  // Draw the menu lines
  Paint_DrawLine(10,  48+0*48, 125, 48+0*48, BLACK, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
  Paint_DrawLine(10,  48+1*48, 125, 48+1*48, BLACK, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
  Paint_DrawLine(10,  48+2*48, 125, 48+2*48, BLACK, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
  Paint_DrawLine(10,  48+3*48, 125, 48+3*48, BLACK, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
  Paint_DrawLine(10,  48+4*48, 125, 48+4*48, BLACK, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
  // Draw header
  Paint_DrawRectangle(0, 0, 135, 48, BLUE, DOT_PIXEL_2X2,DRAW_FILL_FULL);
  Paint_DrawRectangle(0, 0, 135, 48, BLACK, DOT_PIXEL_2X2,DRAW_FILL_EMPTY);
  Paint_DrawString_EN(11, 5, "Choose a", &Font20, WHITE, BLACK);


  // ================= START SERIAL INTERFACE 'B'
  if(buttonPressed == 1){
    Paint_Clear(WHITE);
    LCD_1IN14_Display(BlackImage);
    // width 14 of char
    // width of display
    Paint_DrawString_EN(18, 40, "Running", &Font20, WHITE, BLACK);
    Paint_DrawString_EN(25, 65, "Serial", &Font20, WHITE, BLACK);
    Paint_DrawString_EN(4, 90, "Interface", &Font20, WHITE, BLACK);

    LCD_1IN14_Display(BlackImage);
    runSerialInterface = 1;
    displayState = -1;
  } else if (buttonPressed == 9) {
    //Wipe the screen
    displayState = 0;
    convertData();
    Paint_Clear(WHITE);
    LCD_1IN14_Display(BlackImage);
  } else {
    convertData() ;
  }


  // STATE 0 = CHOOSE THE DEVICE
  if(displayState == 0){
    // ================= SETUP HEADER
    Paint_DrawString_EN(25, 25, "Remote", &Font20, WHITE, BLACK);

    // ================= DISPLAY MENU
    displayMenu(numDevices, devices, buttonPressed);



    // ================= GO TO NEXT MENU IF CENTER PRESSED
    if(buttonPressed == 6 || buttonPressed == 5){
      // Increment display state
      displayState = 1;
      // Set selected device
      selectedDevice = selectedItem;
      // Reset menu variables
      selectedSlot = 0;
      selectedItem = 0;
      // Erase old menu items
      Paint_DrawString_EN(10, 65, devices[windowTop], &Font16, WHITE, WHITE);
      Paint_DrawString_EN(10, 65+1*48, devices[windowTop+1], &Font16, WHITE, WHITE);
      Paint_DrawString_EN(10, 65+2*48, devices[windowTop+2], &Font16, WHITE, WHITE);
      Paint_DrawString_EN(10, 65+3*48, devices[windowTop+3], &Font16, WHITE, WHITE);
      // Erase old header and make new one
      Paint_DrawString_EN(25, 25, "Remote", &Font20, BLUE, BLUE);
      Paint_DrawString_EN(25, 25, "Button", &Font20, WHITE, BLACK);
      // Pre display the menu
      windowTop = 0;
      if(selectedDevice == 0){
        displayMenu(numButtonsDevice0, dev0Buttons, 8);
      } else if(selectedDevice == 1){
        displayMenu(numButtonsDevice1, dev1Buttons, 8);
      } else if(selectedDevice == 2){
        displayMenu(numButtonsDevice2, dev2Buttons, 8);
      } else if(selectedDevice == 3){
        displayMenu(numButtonsDevice3, dev3Buttons, 8);
      } else if(selectedDevice == 4){
        displayMenu(numButtonsDevice4, dev4Buttons, 8);
      } else if(selectedDevice == 5){
        displayMenu(numButtonsDevice5, dev5Buttons, 8);
      } else if(selectedDevice == 6){
        displayMenu(numButtonsDevice6, dev6Buttons, 8);
      } else if(selectedDevice == 7){
        displayMenu(numButtonsDevice7, dev7Buttons, 8);
      } else if(selectedDevice == 8){
        displayMenu(numButtonsDevice8, dev8Buttons, 8);
      } else if(selectedDevice == 9){
        displayMenu(numButtonsDevice9, dev9Buttons, 8);
      } 
    }


  // STATE 1 = CHOOSE THE BUTTON
  } else if(displayState == 1){
    Paint_DrawString_EN(25, 25, "Button", &Font20, WHITE, BLACK);
    if(selectedDevice == 0){
      displayMenu(numButtonsDevice0, dev0Buttons, buttonPressed);
    } else if(selectedDevice == 1){
      displayMenu(numButtonsDevice1, dev1Buttons, buttonPressed);
    } else if(selectedDevice == 2){
      displayMenu(numButtonsDevice2, dev2Buttons, buttonPressed);
    } else if(selectedDevice == 3){
      displayMenu(numButtonsDevice3, dev3Buttons, buttonPressed);
    } else if(selectedDevice == 4){
      displayMenu(numButtonsDevice4, dev4Buttons, buttonPressed);
    } else if(selectedDevice == 5){
      displayMenu(numButtonsDevice5, dev5Buttons, buttonPressed);
    } else if(selectedDevice == 6){
      displayMenu(numButtonsDevice6, dev6Buttons, buttonPressed);
    } else if(selectedDevice == 7){
      displayMenu(numButtonsDevice7, dev7Buttons, buttonPressed);
    } else if(selectedDevice == 8){
      displayMenu(numButtonsDevice8, dev8Buttons, buttonPressed);
    } else if(selectedDevice == 9){
      displayMenu(numButtonsDevice9, dev9Buttons, buttonPressed);
    } 
    // ================= BROADCAST BUTTON 'A'
    if(buttonPressed == 0){
      // TODO: add broadcast implementation here
      // First: Find the correct remote from Liams data type
      // Second: Find the correct button from Liams data type
      // Third: Send the broadcast method the correct array
      static remote deviceToBroadcast;
      static button buttonToBroadcast;
      static int i = 0;
      // First:
      for(i=0; i < MAX_REMOTE_SIZE; i++){
        if(remoteData[i].valid && strcmp(remoteData[i].name, devices[selectedDevice]) == 0){
          deviceToBroadcast = remoteData[i];
        }
      }
      // Second:
      for(i=0; i < MAX_BUTTON_PAIRING; i++){
        if(selectedDevice == 0){
          if(deviceToBroadcast.linkedButtons[i].valid && strcmp(deviceToBroadcast.linkedButtons[i].name, dev0Buttons[selectedItem]) == 0){
            buttonToBroadcast = deviceToBroadcast.linkedButtons[i];
          }        
        } else if(selectedDevice == 1){
          if(deviceToBroadcast.linkedButtons[i].valid && strcmp(deviceToBroadcast.linkedButtons[i].name, dev1Buttons[selectedItem]) == 0){
            buttonToBroadcast = deviceToBroadcast.linkedButtons[i];
          }  
        } else if(selectedDevice == 2){
          if(deviceToBroadcast.linkedButtons[i].valid && strcmp(deviceToBroadcast.linkedButtons[i].name, dev2Buttons[selectedItem]) == 0){
            buttonToBroadcast = deviceToBroadcast.linkedButtons[i];
          }  
        } else if(selectedDevice == 3){
          if(deviceToBroadcast.linkedButtons[i].valid && strcmp(deviceToBroadcast.linkedButtons[i].name, dev3Buttons[selectedItem]) == 0){
            buttonToBroadcast = deviceToBroadcast.linkedButtons[i];
          }  
        } else if(selectedDevice == 4){
          if(deviceToBroadcast.linkedButtons[i].valid && strcmp(deviceToBroadcast.linkedButtons[i].name, dev4Buttons[selectedItem]) == 0){
            buttonToBroadcast = deviceToBroadcast.linkedButtons[i];
          }  
        } else if(selectedDevice == 5){
          if(deviceToBroadcast.linkedButtons[i].valid && strcmp(deviceToBroadcast.linkedButtons[i].name, dev5Buttons[selectedItem]) == 0){
            buttonToBroadcast = deviceToBroadcast.linkedButtons[i];
          }  
        } else if(selectedDevice == 6){
          if(deviceToBroadcast.linkedButtons[i].valid && strcmp(deviceToBroadcast.linkedButtons[i].name, dev6Buttons[selectedItem]) == 0){
            buttonToBroadcast = deviceToBroadcast.linkedButtons[i];
          }  
        } else if(selectedDevice == 7){
          if(deviceToBroadcast.linkedButtons[i].valid && strcmp(deviceToBroadcast.linkedButtons[i].name, dev7Buttons[selectedItem]) == 0){
            buttonToBroadcast = deviceToBroadcast.linkedButtons[i];
          }  
        } else if(selectedDevice == 8){
          if(deviceToBroadcast.linkedButtons[i].valid && strcmp(deviceToBroadcast.linkedButtons[i].name, dev8Buttons[selectedItem]) == 0){
            buttonToBroadcast = deviceToBroadcast.linkedButtons[i];
          }  
        } else if(selectedDevice == 9){
          if(deviceToBroadcast.linkedButtons[i].valid && strcmp(deviceToBroadcast.linkedButtons[i].name, dev9Buttons[selectedItem]) == 0){
            buttonToBroadcast = deviceToBroadcast.linkedButtons[i];
          }  
        } 
      }
      // Third:
      broadcast_IR(buttonToBroadcast.signal);
    }

    // ================= BACK BUTTON
    if(buttonPressed == 4){
      // ================= ERASE OLD MENU
      if(selectedDevice == 0){
        displayMenu(numButtonsDevice0, dev0Buttons, 7);
      } else if(selectedDevice == 1){
        displayMenu(numButtonsDevice1, dev1Buttons, 7);
      } else if(selectedDevice == 2){
        displayMenu(numButtonsDevice2, dev2Buttons, 7);
      } else if(selectedDevice == 3){
        displayMenu(numButtonsDevice3, dev3Buttons, 7);
      } else if(selectedDevice == 4){
        displayMenu(numButtonsDevice4, dev4Buttons, 7);
      } else if(selectedDevice == 5){
        displayMenu(numButtonsDevice5, dev5Buttons, 7);
      } else if(selectedDevice == 6){
        displayMenu(numButtonsDevice6, dev6Buttons, 7);
      } else if(selectedDevice == 7){
        displayMenu(numButtonsDevice7, dev7Buttons, 7);
      } else if(selectedDevice == 8){
        displayMenu(numButtonsDevice8, dev8Buttons, 7);
      } else if(selectedDevice == 9){
        displayMenu(numButtonsDevice9, dev9Buttons, 7);
      } 
      // Erase old header and make new one
      Paint_DrawString_EN(25, 25, "Button", &Font20, BLUE, BLUE);
      Paint_DrawString_EN(25, 25, "Remote", &Font20, WHITE, BLACK);
      // Increment display state
      displayState = 0;
      // Set selected device
      selectedDevice = selectedItem;
      // Reset menu variables
      selectedSlot = 0;
      selectedItem = 0;
      windowTop = 0;
      // ================= DISPLAY MENU
      displayMenu(numDevices, devices, buttonPressed);
    }
  }

  LCD_1IN14_Display(BlackImage);


}


// ==================================================
// === Eliminates invisible characters from a string
// ==================================================
char * eliminateInvisibleChars(char buf[50]){
  static int i;
  for(i=0; i<50; i++){
    if(buf[i] == '\n' || buf[i] == '\r'){
      buf[i] = '\0';
    }
  }
  return buf;
}


// ==================================================
// === Reads the structs from the SD card
// ==================================================
void readFromSD(){
  static FRESULT fr;
  static FATFS fs;
  static FIL fil;
  static int ret;
  static char filename[] = "remote_data.txt";

  static int remoteNumber;
  static int buttonNumber;
  static int signalNumber;
  static int previousItem; // 0=remote, 1=button, 2=signal
  remoteNumber = -1;
  buttonNumber = -1;
  previousItem = 1;
  signalNumber = 0;
  
  // Initialize SD card
  if (!sd_init_driver()) {
      printf("ERROR: Could not initialize SD card\r\n");
      while (true);
  }

  // Mount drive
  fr = f_mount(&fs, "0:", 1);
  if (fr != FR_OK) {
      printf("ERROR: Could not mount filesystem (%d)\r\n", fr);
      while (true);
  }

  // Open file for reading
  fr = f_open(&fil, filename, FA_READ);
  if (fr != FR_OK) {
      printf("ERROR: Could not open file (%d)\r\n", fr);
      while (true);
  }

  // Print every line in file over serial
  printf("Reading from file '%s':\r\n", filename);

  while (f_gets(buf, sizeof(buf), &fil)) {
       
      string1DArrayEqualsBuf( buf, eliminateInvisibleChars(buf));

      // If the line is "remote" then the next thing is a remote name
      if(strcmp(buf, "remote") == 0 && (previousItem == 1 || previousItem == 2)){
        printf("Buf : %s", buf);
        f_gets(buf, sizeof(buf), &fil);
        printf("Buf 2: %s", buf);
        string1DArrayEqualsBuf( buf, eliminateInvisibleChars(buf));
        remoteNumber ++;
        string1DArrayEquals(remoteData[remoteNumber].name, buf);
        remoteData[remoteNumber].valid = 1;
        printf("\n\rRemote: %s, RemoteNumber %d, Buffer, %s", remoteData[remoteNumber].name, remoteNumber, buf);
        previousItem = 0;
        buttonNumber = -1;
      } else // If the line is "button" then the next thing is a button name
      if(strcmp(buf, "button") == 0 && (previousItem == 0 || previousItem == 2)){
        f_gets(buf, sizeof(buf), &fil);
        string1DArrayEqualsBuf( buf, eliminateInvisibleChars(buf));
        buttonNumber ++;
        string1DArrayEquals(remoteData[remoteNumber].linkedButtons[buttonNumber].name, buf);
        remoteData[remoteNumber].linkedButtons[buttonNumber].valid = 1;
        printf("\n\rButton: %s, ButtonNumber %d, RemoteNumber %d, Buffer, %s", remoteData[remoteNumber].linkedButtons[buttonNumber].name, buttonNumber, remoteNumber, buf);
        previousItem = 1;
      } else// If the line is "signal" then the next thing is a signal
      if(strcmp(buf, "signal") == 0 && previousItem == 1){
        for(signalNumber = 0; signalNumber < MAX_SIGNAL_SIZE; signalNumber++){
          f_gets(buf, sizeof(buf), &fil);
          string1DArrayEqualsBuf( buf, eliminateInvisibleChars(buf));
          remoteData[remoteNumber].linkedButtons[buttonNumber].signal[signalNumber] = atoi(buf);
          printf("\n\rInt version of the string: %d", atoi(buf));
        }
        previousItem = 2;
      }
  }
  printf("\r\n---\r\n");

  // Close file
  fr = f_close(&fil);
  if (fr != FR_OK) {
      printf("ERROR: Could not close file (%d)\r\n", fr);
      while (true);
  }

  // Unmount drive
  f_unmount("0:");

}

// ==================================================
// === Writes the structs to the SD card
// ==================================================
void writeToSD(){
  static FRESULT fr;
  static FATFS fs;
  static FIL fil;
  static int ret;
  static char filename[] = "remote_data.txt";
  static int i;
  static int b;
  static int s;

  // Initialize SD card
  if (!sd_init_driver()) {
      printf("ERROR: Could not initialize SD card\r\n");
      while (true);
  }

  // Mount drive
  fr = f_mount(&fs, "0:", 1);

  if (fr != FR_OK) {
      printf("ERROR: Could not mount filesystem (%d)\r\n", fr);
      while (true);
  }

  // Open file for writing ()
  fr = f_open(&fil, filename, FA_WRITE | FA_CREATE_ALWAYS);
  if (fr != FR_OK) {
      printf("ERROR: Could not open file (%d)\r\n", fr);
      while (true);
  }

  // =========== BEGIN WIRITNG STRUCTS
  // Run through every remote
  for(i=0; i<MAX_REMOTE_SIZE; i++){
    // If the remote is valid: print "remote" and then the name
    if(remoteData[i].valid){
      ret = f_printf(&fil, "remote\r\n");
      ret = f_printf(&fil, "%s\r\n",  remoteData[i].name);
      // Run through all buttons
      for(b=0; b<MAX_BUTTON_PAIRING; b++){
        // If the button is valid: print "button" and then the name
        if(remoteData[i].linkedButtons[b].valid){
          ret = f_printf(&fil, "button\r\n");
          ret = f_printf(&fil, "%s\r\n", remoteData[i].linkedButtons[b].name);
          // After name print "signal" and then the whole signal
          ret = f_printf(&fil, "signal\r\n");
          for(s=0; s<MAX_SIGNAL_SIZE; s++){
            ret = f_printf(&fil, "%d\r\n", remoteData[i].linkedButtons[b].signal[s]);
          }
        }
      }
    }

  }
  // Close file
  fr = f_close(&fil);
  if (fr != FR_OK) {
      printf("ERROR: Could not close file (%d)\r\n", fr);
      while (true);
  }

  // Unmount drive
  f_unmount("0:");
  printf("FINISHED WRITING TO SD\r\n");
}

//////////////////////////////////////////
//  Testing Method, Not used in runtime //
//////////////////////////////////////////
void initRemoteData() {
    for(int i = 0; i < MAX_REMOTE_SIZE; i++) {
        for(int j = 0; j < MAX_BUTTON_PAIRING; j++) {
            remoteData[i].linkedButtons[j].valid = false;
        }
        remoteData[i].valid = false;
    }
}


//////////////////////////////////////
//  Differentiate relative timings  //
//////////////////////////////////////
void encode() {
    int tail = edges[0] - 1;
    uint32_t baseTime = edges[1];

    //TODO: Save entry by subtracting up, so no empty edges[1]?
    for(int j = tail; j > 1; j--) {
        edges[j] -= edges[j-1];
    }
    edges[1] = 0;

}

//////////////////////////////////////////////
//  Partner method for encapsle_button_ISR  //
//  (See info below)                        //
//////////////////////////////////////////////
void encapsle_button(uint32_t *edgeFlip, uint32_t *dest) {
    encode();
    int tail = (int) edges[0];
    for(int i = 0; i < tail; i++) {
        dest[i] = edgeFlip[i];
    }
    edgeFlip[0] = 1;
}


//////////////////////////////////////////////////
//  Runs encoding and transfer of data from     //
//  temp edges[] to more permenant dataArr[]    //
//  for storage once end of signal is detected  //
//////////////////////////////////////////////////
//Note: Can merge with encapsle button method 
//if/once able to pass args into ISR
alarm_callback_t encapsle_button_ISR()  {
    
    // Guard against dummy timeout triggers
    if(!recieved && time_reached(timeoutWindow)) {
        printf("\n\rTIMEOUT TRIG");
        encapsle_button(edges, dataArr);

        printf("\n\rEXIT TIMEOUT\n");
        timeout = add_alarm_in_ms(9999999, encapsle_button_ISR, NULL, false); //Reset dummy timer
        recieved = true;
    }
}



//////////////////////////////////////////////////
//  Detects and logs rising/falling edges of    //
//  signal.  Handles as an ISR.  Sampling ended //
//  via encapsle_button_ISR() timeout           //
//////////////////////////////////////////////////
void gpio_log() {
    int ind = edges[0];
    if (ind < MAX_DATA_SIZE) { //PLACE BACK TO MAX DATA SIZE ONCE FINISHED
        edges[ind] = time_us_32();
        
        
        if(!cancel_alarm(timeout)) {
            printf("\n\rWARN: Timeout fault\n\n\n\n");
        }
        timeout = add_alarm_in_ms(INPUT_IR_TIMEOUT_WINDOW_MS, encapsle_button_ISR, NULL, false);
        if(timeout <= 0) {
          printf("\n\rWARN: Timeout failed to create\n\n");
        }
        
        timeoutWindow = make_timeout_time_us((uint32_t) INPUT_IR_TIMEOUT_WINDOW_MS);
        
        edges[0] += 1; //Increment array size
    } else {
        printf("\n\rERROR: OUT OF BOUNDS\n");
        for(int j = 0; j < MAX_DATA_SIZE; j++) {
            printf("%d\n", edges[j]);
        }
    }
}



static PT_THREAD (protothread_core_1(struct pt *pt))
{
  // Indicate thread beginning
  PT_BEGIN(pt) ;
  static int user_input_int, user_input_temp_int;
  static int remoteIndex, buttonIndex;
  static int i, j;
  static char user_input_string[
    MAX_LABEL_CHAR_SIZE*4]; //Expand further to mitigate accidental overflows, truncate input
  static char trunc_string[MAX_LABEL_CHAR_SIZE];
  

  while(1) {
      
    top:

    PT_YIELD_usec(300000);
    PT_YIELD_UNTIL(pt, runSerialInterface != 0);
      
    if(runSerialInterface != 0){
      user_input_int = -1;
      while(user_input_int != 1 && user_input_int != 2 && user_input_int != 3) {
          sprintf(pt_serial_out_buffer, "\n\rWould you not like to (1) Add a button? (2) Remove a button? (3) Remove Remote\n\r"); //TODO: Add option to remove remotes? (Implicit add remote via add button)
          serial_write;
          serial_read;
          sscanf(pt_serial_in_buffer,"%d", &user_input_int);
      }


      //////////////////
      //  Add Button  //
      //////////////////
      if(user_input_int == 1) {


          remoteIndex = -1;



          sprintf(pt_serial_out_buffer, "\rPlease enter a name for your remote\n\r");
          serial_write;
          sprintf(pt_serial_out_buffer, "\rOr insert a new name to create one.\n\r");
          serial_write;


          sprintf(pt_serial_out_buffer, "\rValid entries include:\n\r");
          serial_write;
          for(i = 0; i < MAX_REMOTE_SIZE; i++) {
              if(remoteData[i].valid) {
                  sprintf(pt_serial_out_buffer, " - %s\n\r", remoteData[i].name);
                  serial_write;
              }
          
          }
          serial_read;
          sscanf(pt_serial_in_buffer,"%s", &user_input_string);
          for(i = 0; i < MAX_LABEL_CHAR_SIZE; i++) {
              trunc_string[i] = user_input_string[i];
          }
          sprintf(pt_serial_out_buffer, "\rREC: %s\n\r", trunc_string);
          serial_write;
          
          for(i = 0; i < MAX_REMOTE_SIZE; i++) { //Ensures no name conflict
              if(remoteData[i].valid &&
                  strcmp(remoteData[i].name, trunc_string) == 0) {
                  sprintf(pt_serial_out_buffer, "\rSlot found: %d\n\r", i);
                  serial_write;
                  remoteIndex = i;
                  break;
              }
          }

          // Found remote with same namespace
          if(remoteIndex != -1) {
              sprintf(pt_serial_out_buffer, "\rFound matching remote name!\n\r");
              serial_write;
              
          } else {
              for(i = 0; i < MAX_REMOTE_SIZE; i++) {

                  if(!remoteData[i].valid) {
                      remoteIndex = i;
                      remoteData[i].valid = true;
                      for(i = 0; i < MAX_LABEL_CHAR_SIZE; i++) { //Destructively uses i due to break afterwards
                          //TODO: Verify Trunc size to save single index
                          remoteData[remoteIndex].name[i] = trunc_string[i];
                      }

                      break;
                  }
              }

              if(remoteIndex == -1) {
                  sprintf(pt_serial_out_buffer, "\rERROR: No spare slot found.  EXITING...\n\r");
                  serial_write;

                  runSerialInterface = 0;

                  goto top;
              }
          }

          //////////////////////////
          //  Remote designated   //
          //  Cont. adding button //
          //////////////////////////

          struct remote remoteInst = remoteData[remoteIndex];
          buttonIndex = -1;

          sprintf(pt_serial_out_buffer, "\rInsert button name:\n\r");
          serial_write;

          serial_read;
          sscanf(pt_serial_in_buffer,"%s", &user_input_string);
          for(i = 0; i < MAX_LABEL_CHAR_SIZE; i++) {
              trunc_string[i] = user_input_string[i];
          }

          //Search for matching button
          for(i = 0; i < MAX_BUTTON_PAIRING; i++) {
              if(remoteInst.linkedButtons[i].valid &&
                  !(strcmp(remoteInst.linkedButtons[i].name, trunc_string))) {
                  buttonIndex = i;
                  break;
              }
          }

          if(buttonIndex != -1) {
              while(user_input_int != 1 && user_input_int != 2) {
                  sprintf(pt_serial_out_buffer, "\rFound matching button name.  Override? (1) Yes (2) No\n\r");
                  serial_write;
                  serial_read;
                  sscanf(pt_serial_in_buffer,"%d", &user_input_int);
              }

              // Nothing needs to be done if overriding, skip to abort protocol
              if(user_input_int == 2) {
                  sprintf(pt_serial_out_buffer, "\rCancelling...\n\r");
                  serial_write;
                  runSerialInterface = 0;
                  goto top; // Bad input, return to top of processing
              }
          } else {
              //Find available namespace
              for(i = 0; i < MAX_BUTTON_PAIRING; i++) {
                  if(!remoteInst.linkedButtons[i].valid) {
                      buttonIndex = i;
                      break;
                  }
              }

              //No namespace available
              if(buttonIndex == -1) {
                  sprintf(pt_serial_out_buffer, "\rERROR: No available button space on remote\n\r");
                  serial_write;
                  sprintf(pt_serial_out_buffer, "\rAborting...\n\r");
                  serial_write;
                  runSerialInterface = 0;
                  goto top;
              }
          }

          //////////////////////////
          //  Button Designated   //
          //  Cont. Setting up    //
          //////////////////////////

          sprintf(pt_serial_out_buffer, "\rReady to detect button signal...\n\r");
          serial_write;
          recieved = false;
          // Enables detection of reading IR signal edges to encode
          gpio_set_irq_enabled(IR_RECEIVER_PIN, GPIO_IRQ_EDGE_RISE | GPIO_IRQ_EDGE_FALL, true);

          PT_YIELD_UNTIL(pt, recieved); //Handshake variable that signal got received

          //... And disable detection now that the process is finished
          gpio_set_irq_enabled(IR_RECEIVER_PIN, GPIO_IRQ_EDGE_RISE | GPIO_IRQ_EDGE_FALL, false);

          remoteData[remoteIndex].linkedButtons[buttonIndex].valid = true;
          sprintf(pt_serial_out_buffer, "\rSignal recieved!\n\r");
          serial_write;


          
          for(i = 0; i < MAX_LABEL_CHAR_SIZE; i++) {
              remoteData[remoteIndex].linkedButtons[buttonIndex].name[i] = trunc_string[i];
          }

          for(i = 0; i < MAX_SIGNAL_SIZE; i++) {
              remoteData[remoteIndex].linkedButtons[buttonIndex].signal[i] = dataArr[i];
          }

          sprintf(pt_serial_out_buffer, "\rButton set!\n\r");
          serial_write;
          // Wipe screen and go up to refresh display properly
          refreshDisplay(9); 
          refreshDisplay(2); 
          saveData = 1;
          runSerialInterface = 0;
          goto top;



      } else if (user_input_int == 2) {

          //////////////////
          //  Rem Button  //
          //////////////////
          remoteIndex = -1;
          while(remoteIndex == -1) {
              sprintf(pt_serial_out_buffer, "\rPlease enter a name for your remote (q to quit)\n\r");
              serial_write;
              sprintf(pt_serial_out_buffer, "\rValid entries include:\n\r");
              serial_write;
              for(i = 0; i < MAX_REMOTE_SIZE; i++) {
                  if(remoteData[i].valid) {
                      sprintf(pt_serial_out_buffer, " - %s\n\r", remoteData[i].name);
                      serial_write;
                  }
              }


              serial_read;
              sscanf(pt_serial_in_buffer,"%s", &user_input_string);
              for(i = 0; i < MAX_LABEL_CHAR_SIZE; i++) {
                  trunc_string[i] = user_input_string[i];
              }

              if(!strcmp(trunc_string, "q")) {
                  sprintf(pt_serial_out_buffer, "\rExiting...\n\r");
                  serial_write;
                  runSerialInterface = 0;
                  goto top;
              }

              
              for(i = 0; i < MAX_REMOTE_SIZE; i++) {
                  if(remoteData[i].valid &&
                      !(strcmp(remoteData[i].name, trunc_string))) {
                      remoteIndex = i;
                      break;
                  }
              }

              if(remoteIndex == -1) {
                  sprintf(pt_serial_out_buffer, "\rERROR: Remote name not found\n");
                  serial_write;
              }
          }

          struct remote remoteInst = remoteData[remoteIndex];
          buttonIndex = -1;

          while(buttonIndex == -1) {
              sprintf(pt_serial_out_buffer, "\rInsert button name (q to quit):\n\r");
              serial_write;
              sprintf(pt_serial_out_buffer, "\rValid entries include:\n\r");
              serial_write;
              for(i = 0; i < MAX_BUTTON_PAIRING; i++) {
                  if(remoteInst.linkedButtons[i].valid) {
                      sprintf(pt_serial_out_buffer, " - %s\n\r", remoteInst.linkedButtons[i].name);
                      serial_write;
                  }
              }


              serial_read;
              sscanf(pt_serial_in_buffer,"%s", &user_input_string);
              for(i = 0; i < MAX_LABEL_CHAR_SIZE; i++) {
                  trunc_string[i] = user_input_string[i];
              }

              if(!strcmp(trunc_string, "q")) {
                  sprintf(pt_serial_out_buffer, "\rExiting...\n\r");
                  serial_write;
                  runSerialInterface = 0;
                  goto top;
              }

              for(i = 0; i < MAX_BUTTON_PAIRING; i++) {
                  if(remoteInst.linkedButtons[i].valid &&
                      !(strcmp(remoteInst.linkedButtons[i].name, trunc_string))) {
                      buttonIndex = i;
                      break;
                  }
              }

              if(buttonIndex == -1) {
                  sprintf(pt_serial_out_buffer, "\rERROR: Button name not found\n\r");
                  serial_write;
              }
          }


          sprintf(pt_serial_out_buffer, "\rButton found, deleting...\n\r");
          serial_write;

          remoteData[remoteIndex].linkedButtons[buttonIndex].valid = false;

          sprintf(pt_serial_out_buffer, "\rButton deleted!\n\r");
          serial_write;
          // Wipe screen and go up to refresh display properly
          refreshDisplay(9); 
          refreshDisplay(2); 
          saveData = 1;
          runSerialInterface = 0;

      } else if (user_input_int == 3) {

          //////////////////
          //  Rem Remote  //
          //////////////////
          remoteIndex = -1;
          while(remoteIndex == -1) {
              sprintf(pt_serial_out_buffer, "\rPlease enter a name for your remote (q to quit)\n");
              serial_write;
              sprintf(pt_serial_out_buffer, "\rValid entries include:\n\r");
              serial_write;
              for(i = 0; i < MAX_REMOTE_SIZE; i++) {
                  if(remoteData[i].valid) {
                      sprintf(pt_serial_out_buffer, " - %s\n\r", remoteData[i].name);
                      serial_write;
                  }
              }


              serial_read;
              sscanf(pt_serial_in_buffer,"%s", &user_input_string);
              for(i = 0; i < MAX_LABEL_CHAR_SIZE; i++) {
                  trunc_string[i] = user_input_string[i];
              }

              if(!strcmp(trunc_string, "q")) {
                  sprintf(pt_serial_out_buffer, "\rExiting...\n\r");
                  serial_write;
                  runSerialInterface = 0;
                  goto top;
              }

              // Locate match
              for(i = 0; i < MAX_REMOTE_SIZE; i++) {
                  if(remoteData[i].valid &&
                      !(strcmp(remoteData[i].name, trunc_string))) {
                      remoteIndex = i;
                      break;
                  }
              }

              if(remoteIndex == -1) {
                  sprintf(pt_serial_out_buffer, "\rERROR: Remote name not found\n\r");
                  serial_write;
              }
          }
          
          sprintf(pt_serial_out_buffer, "\rRemote found, deleting data...\n\r");
          serial_write;

          for(i = 0; i < MAX_BUTTON_PAIRING; i++) {
              remoteData[remoteIndex].linkedButtons[i].valid = false;
          }

          remoteData[remoteIndex].valid = false;

          // Wipe screen and go up to refresh display properly
          refreshDisplay(9); 
          refreshDisplay(2); 
          sprintf(pt_serial_out_buffer, "\rRemote data deleted!\n\r");
          serial_write;

          saveData = 1;
          runSerialInterface = 0;

          goto top;


      }
    }
  }
  printf("GOT OUT OF THE WHILE LOOP SOMEHOW");
  PT_END(pt);
}



// ==================================================
// === Run the Boot sequence for the LCD
// ==================================================
void bootDisplay(){
  DEV_Delay_ms(100);
  if(DEV_Module_Init()!=0){
    while(true); // Crash
  }

  DEV_SET_PWM(50);
  // LCD Init
  LCD_1IN14_Init(1); //0=horizontal, 1=vertical
  LCD_1IN14_Clear(WHITE); 


  UDOUBLE Imagesize = LCD_1IN14_HEIGHT*LCD_1IN14_WIDTH*2;
  if((BlackImage = (UWORD *)malloc(Imagesize)) == NULL) {
    printf("Failed to apply for black memory...\r\n");
    exit(0);
  }

  // Create a new image cache named IMAGE_RGB and fill it with white
  Paint_NewImage((UBYTE *)BlackImage,LCD_1IN14.WIDTH,LCD_1IN14.HEIGHT, 0, WHITE);
  Paint_SetScale(65);
  Paint_Clear(WHITE);
  Paint_SetRotate(ROTATE_180);
  Paint_Clear(WHITE);

  // Display boot-up screen
  Paint_DrawString_EN(4, 40, "Universal", &Font20, WHITE, BLACK);
  Paint_DrawString_EN(25, 65, "Remote", &Font20, WHITE, BLACK);
  Paint_DrawString_EN(39, 100, "ECE 4760", &Font12, WHITE, BLACK); 
  Paint_DrawString_EN(22, 116, "ocl6 & lms445", &Font12, WHITE, BLACK); 

  // Refresh the picture in RAM to LCD
  LCD_1IN14_Display(BlackImage);
  DEV_Delay_ms(2000);



  // Set backlight to 100% brightness
  DEV_SET_PWM(100);
  // Assign display pins
  SET_Infrared_PIN(keyA);    
  SET_Infrared_PIN(keyB);
  SET_Infrared_PIN(up);
  SET_Infrared_PIN(dowm);
  SET_Infrared_PIN(left);
  SET_Infrared_PIN(right);
  SET_Infrared_PIN(ctrl);
  // Wipe the screen
  Paint_Clear(WHITE);
  LCD_1IN14_Display(BlackImage);


  // Draw the menu lines
  Paint_DrawLine(10,  48+0*48, 125, 48+0*48, BLACK, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
  Paint_DrawLine(10,  48+1*48, 125, 48+1*48, BLACK, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
  Paint_DrawLine(10,  48+2*48, 125, 48+2*48, BLACK, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
  Paint_DrawLine(10,  48+3*48, 125, 48+3*48, BLACK, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
  Paint_DrawLine(10,  48+4*48, 125, 48+4*48, BLACK, DOT_PIXEL_2X2, LINE_STYLE_SOLID);
  LCD_1IN14_Display(BlackImage);
  convertData();
  refreshDisplay(2);  


}

// ==================================================
// === Poll buttons for presses
// ==================================================
void pollButtons(){
  if(DEV_Digital_Read(keyA ) == 0){
    refreshDisplay(0);
  } else 
  if(DEV_Digital_Read(keyB ) == 0){
    refreshDisplay(1);
  } else
  if(DEV_Digital_Read(up ) == 0){
    refreshDisplay(2);
  } else
  if(DEV_Digital_Read(dowm ) == 0){
    refreshDisplay(3);
  } else
  if(DEV_Digital_Read(left ) == 0){
    refreshDisplay(4);
  } else
  if(DEV_Digital_Read(right ) == 0){
    refreshDisplay(5);
  } else
  if(DEV_Digital_Read(ctrl ) == 0){
    refreshDisplay(6);
  }

}






// ========================================
// === Core 1 main -- started in main below
// ========================================
void core1_main(){
  
  while(1){

    sleep_us(30000);

    if(saveData == 1){
      writeToSD();
      saveData=0;
    }
    while(runSerialInterface == 0){
      //Main process
      sleep_us(30000);
      pollButtons();
    }
    
  }


}

int main(){
  // Initialize stdio
  stdio_init_all();
  // Initialize data from SD Card
  readFromSD();
  // Initialize LCD display
  bootDisplay();
  // Initialize Structs
  initRemoteData();
  //Finish SD init
  convertData(); 
  refreshDisplay(8);
  writeToSD(); 


  //Init IR LED/REC PINs
  gpio_init(IR_RECEIVER_PIN);
  gpio_disable_pulls(IR_RECEIVER_PIN); //Disable Pullup on interupt pin

  gpio_init(IR_LED_PIN);
  gpio_set_dir(IR_LED_PIN, GPIO_OUT);

  gpio_put(IR_LED_PIN, 0);


  timeout = add_alarm_in_ms(9999999, encapsle_button_ISR, NULL, false); //Set indef. future for makeshift valid timer

  // Configure GPIO interrupt (Can be packaged for toggle on/off method for open-receiving)
  // -> Must be called while IR LED is currently off 
  edges[0] = 1; // Ensure indexing tail is init first
  //Init to false, will be enabled during serial interface FSM
  gpio_set_irq_enabled_with_callback(IR_RECEIVER_PIN, GPIO_IRQ_EDGE_RISE | GPIO_IRQ_EDGE_FALL, true, &gpio_log);
  gpio_set_irq_enabled(IR_RECEIVER_PIN, GPIO_IRQ_EDGE_RISE | GPIO_IRQ_EDGE_FALL, false);

  // Add thread to core 0
  pt_add_thread(protothread_core_1) ;

  // Start core 1 
  multicore_reset_core1();
  multicore_launch_core1(&core1_main);

  // Start threads
  pt_schedule_start ;

  return 0;
}


```

## Appendix C - Schematics

## Appendix D - Work Breakdown

## Appendix E - References and Parts

### Libraries

* SD Card C Library: http://elm-chan.org/fsw/ff/00index_e.html
* LCD Screen C Library: https://www.waveshare.com/wiki/Pico-LCD-1.14

### Parts

* Raspberry Pi Pico: https://www.raspberrypi.com/products/raspberry-pi-pico/
* LCD Screen: https://www.waveshare.com/wiki/Pico-LCD-1.14
* SD Card Breakout Board: https://www.digikey.com/en/products/detail/adafruit-industries-llc/4682/12822319
* AA Battery Holder: https://www.amazon.com/dp/B09V7Z4MT7?psc=1&ref=ppx_yo2ov_dt_b_product_details
* Infrared LED: https://www.sparkfun.com/products/9349?_ga=2.201535761.1075059845.1667497389-1261656821.1667497389
* Infrared Reciever: https://www.sparkfun.com/products/10266?_ga=2.197351087.1075059845.1667497389-1261656821.1667497389



