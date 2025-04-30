
# STM32 USB Host CDC for ACM Devices

This repository provides modified code to support USB CDC ACM (Abstract Control Model) devices as USB hosts using STM32's USB Host Library.

## Steps to Implement

1. **Generate USB Host code using STM32CubeIDE.**  
2. **Copy the CDC class from the USB Host library:**  
   Copy the following directory to your project:  
   `Middlewares/ST/STM32_USB_Host_Library/Class/CDC`  
3. **Add the following callbacks to `main.c`:**
   ```c
   void USBH_CDC_LineCodingChanged(USBH_HandleTypeDef *phost) {
     M_USB_STATE = USER_USB_SET_LINE_STATE;
   }

   void USBH_CDC_LineStateChanged(USBH_HandleTypeDef *phost) {
     M_USB_STATE = USER_USB_WRITE_DATA;
   }
   ```
4. **Modify `usb_host.c`:**  
   In the `USBH_UserProcess()` function, update the case `HOST_USER_CLASS_ACTIVE` to initiate line coding:
   ```c
   static void USBH_UserProcess(USBH_HandleTypeDef *phost, uint8_t id) {
     switch (id) {
       case HOST_USER_SELECT_CONFIGURATION:
         break;

       case HOST_USER_DISCONNECTION:
         Appli_state = APPLICATION_DISCONNECT;
         break;

       case HOST_USER_CLASS_ACTIVE:
         Appli_state = APPLICATION_READY;
         M_USB_STATE = USER_USB_SET_LINE_CODING;
         break;

       case HOST_USER_CONNECTION:
         Appli_state = APPLICATION_START;
         break;

       default:
         break;
     }
   }
   ```

## Modifications

1. **Change the class protocol in `USBH_CDC_InterfaceInit()`:**  
   Replace the protocol with `NO_CLASS_SPECIFIC_PROTOCOL_CODE`:
   ```c
   interface = USBH_FindInterface(phost, COMMUNICATION_INTERFACE_CLASS_CODE,
                                   ABSTRACT_CONTROL_MODEL,
                                   NO_CLASS_SPECIFIC_PROTOCOL_CODE);
   ```

2. **Add `USBH_CDC_SetLineState()` function:**
   ```c
   USBH_StatusTypeDef USBH_CDC_SetLineState(USBH_HandleTypeDef *phost, uint16_t dtr_rts) {
     CDC_HandleTypeDef *CDC_Handle = (CDC_HandleTypeDef*) phost->pActiveClass->pData;

     if (phost->gState == HOST_CLASS) {
       CDC_Handle->state = CDC_SET_LINE_STATE;
       CDC_Handle->dtr_rts = dtr_rts;
     }
     return USBH_OK;
   }
   ```

3. **Add a weak implementation of the `USBH_CDC_LineStateChanged()` callback:**
   ```c
   __weak void USBH_CDC_LineStateChanged(USBH_HandleTypeDef *phost) {
     UNUSED(phost);
   }
   ```

4. **Handle the `CDC_SET_LINE_STATE` state in `USBH_CDC_Process()`:**
   ```c
   case CDC_SET_LINE_STATE:
     req_status = SetControlLineState(phost, CDC_Handle->dtr_rts);
     if (req_status == USBH_OK) {
       CDC_Handle->state = CDC_IDLE_STATE;
       USBH_CDC_LineStateChanged(phost);
     } else {
       if (req_status != USBH_BUSY) {
         CDC_Handle->state = CDC_ERROR_STATE;
       }
     }
     break;
   ```

5. **Add `CDC_SET_LINE_STATE` to `CDC_StateTypeDef` in `usbh_cdc.h`:**
   ```c
   typedef enum {
     CDC_IDLE_STATE = 0U,
     CDC_SET_LINE_CODING_STATE,
     CDC_SET_LINE_STATE, // Added
     CDC_GET_LAST_LINE_CODING_STATE,
     CDC_GET_LAST_LINE_STATE,
     CDC_TRANSFER_DATA,
     CDC_ERROR_STATE,
   } CDC_StateTypeDef;
   ```

6. **Set DTR and RTS signals in `main.c`:**
   ```c
   USBH_CDC_SetLineState(&hUsbHostFS, 0x03); // Set DTR = 1 and RTS = 1
   ```

