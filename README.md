## Transmit Beamforming in Monitor Mode 

This is a very early demo, still WIP. Do not use it now.  

What we know now:  
1. The RTL88X2E has proper VHT beamforming implementation in hardware/firmware
2. The driver maintains an FSM for the sounding process (while there are maybe >= 3 versions of FSM code in the driver XD)
3. The sounding is triggered by the beamformer's FSM sending a VHT NDPA frame, then the hardware will automatically send the NDP packet after a SIFS time
4. The beamformee will reply with a VHT Compressed Beamforming Report (CBR) frame, which can be captured (e.g. Wireshark), and then transformed into the V matrix
5. The beamformer captures the CBR frame, then, enable the TXBF
6. In my test, sometimes the RSSI increases suddenly by +2dB after sounding, but sometimes no increase at all. The ```psi``` and ```phi``` values in CBR frame do change when moving adapters. -- so, it seems working?


Weird behaviors I don't understand: 
1. The beamformer and beamformee seem to only check the NDPA/CBR frame's receiver address with the stock (efuse) MAC address.
   Setting any MAC address is not working. The G_ID / P_AID seems no use at all.
2. The TX power override not work when TXBF is enabled.
3. ... (I'm sure there will be more confusing issues)

Some picture: 

Measuring the VDET pin of the adapter's PA (can indicate TX status). The waveform stands for NDPA, SIFS slot, NDP, and re-transmissions (as no bfee when taking the picture), respectively
![image](https://github.com/user-attachments/assets/50bc6bb0-0dae-4940-b354-8eaf719d8218)

One CBR frame, captured on the beamformer side, with 2 8812eu (2T2R) as bfer & bfee
![image](https://github.com/user-attachments/assets/c7dbeb3f-4633-4ce1-b5fd-09692c63c784)  
