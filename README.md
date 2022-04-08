# LEDBlinkTrustZone
 A *working* LED Blink project set up with MPLab Harmony v3 Software Framework and SAM L11 Development Board (SAM L11 MCU). 

## Why this exists? 
I had trouble getting the example LEDBlink program provided in Atmel START to work. I could additionaly not get the sample MPLab Harmony LED project to work either. So, I sought out to figure out how to initialize devices and peripherals in a TrustZone environment and eventually toggle an LED on the SAM L11 xPlained Pro development board. The MPLab Harmony v3 framework allows you to initialize peripherals easily, making the coding process straightforward however example code provided by the framework has proven to be non functional, at least in my experience. 

## What this does
All this project does is simply toggle pin 8 otherwise named 'PA07' which is connected to an amber LED on the L11 development board. The board originally did this until I flashed it with example code (go figure). So, after messing around, I came up with this which is two implementations that both toggle the same led on pin 8, one using a TrustZone initialized timer, and one using a non-secure initialized timer. This demonstrates the use of TrustZone to hide functions from a non secure implementation.  

## Project structure
- The folder named LEDBlink contains an implementation using the non-secure region of memory.
- The folder named LEDBlinkSecure contains an implementation using the secure region of memory.

## Implementation
### Secure Implementation:
```C
// Global variable.
static bool volatile LEDstate = false;  // Used to save state of LED. 

// Interrupt service routine.
void TC0_CH0_TimerInterruptHandler(TC_TIMER_STATUS status, uintptr_t context) {
    LEDstate = true; 
}
    
/* typedef for non-secure callback functions */
typedef void (*funcptr_void) (void) __attribute__((cmse_nonsecure_call));

// Main entry point.
int main (void)
{
    uint32_t msp_ns = *((uint32_t *)(TZ_START_NS));
    volatile funcptr_void NonSecure_ResetHandler;

    /* Initialize all modules */
    SYS_Initialize (NULL);

    if (msp_ns != 0xFFFFFFFF)
    {
        /* Set non-secure main stack (MSP_NS) */
        __TZ_set_MSP_NS(msp_ns);

        /* Get non-secure reset handler */
        NonSecure_ResetHandler = (funcptr_void)(*((uint32_t *)((TZ_START_NS) + 4U)));

        /* Start non-secure state software application */
        NonSecure_ResetHandler();
    }
    
    // Set reset register for timer interrupt.
    TC0_TimerCallbackRegister(TC0_CH0_TimerInterruptHandler, (uintptr_t)NULL);
   
    // Start timer. 
    TC0_TimerStart();

    while (true) {
        if (LEDstate == true) {
            LED0_Toggle();
            LEDstate = false; 
        }
    }

    /* Execution should not come here during normal operation */
    return ( EXIT_FAILURE );
}

```
- The above code is from "main.c" located in LEDBlinkSecure\secure\firmware\src
- The implementation works by simply generating an interrupt every half second, changing an internal varible named LEDstate, and calling LED0_Toggle() (when LEDstate is true) which toggles the logic level of PA07 on each call. 
### Non secure implementation:
```C
static bool volatile led_status = false;

/**
 *  Interrupt handler for timer.
 */
void TC0_CH0_TimerInterruptHandler(TC_TIMER_STATUS status, uintptr_t context) {
    led_status = true;
}

int main ( void )
{
    /* Initialize all modules */
    SYS_Initialize ( NULL );
    
    // Register callback for interrupt.
    TC0_TimerCallbackRegister(TC0_CH0_TimerInterruptHandler, (uintptr_t)NULL);
    
     // Start T0 Timer. 
    TC0_TimerStart();

    while ( true )
    {
        /* Maintain state machines of all polled MPLAB Harmony modules. */
        SYS_Tasks ( );
        
        // LED toggle code section. 
        if (led_status == true) {
            LED0_Toggle();
            led_status = false;
        }
    }

    /* Execution should not come here during normal operation */

    return ( EXIT_FAILURE );
}
```
- The above code is from "main.c" located in LEDBlink\NonSecure\firmware\src
- Note the difference between this implemenation and the secure implementation.
## Things to note
- It is possible to define functions within the secure region and interface them with the nonsecure program by means of a region of memory named the NSC which stands for 'Non Secure Callable' Memory. I could elaborate on this, but this would lengthen the readme substantially. 
    - See "nonsecure_entry.c" in the "trustZone" folder inside of the "firmware" folder of the secure portion of any one of the two examples. Also note the typedef within each implementation.
## Useful resources:
- https://www.microchip.com/en-us/tools-resources/configure/mplab-harmony
- https://www.microchip.com/en-us/tools-resources/develop/mplab-x-ide

