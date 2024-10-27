## Transmit Beamforming in Monitor Mode 
A minimum re-implementation of using Transmit Single-User Beamforming in monitor mode (without establishing AP/STA connection)  

**This is a very early demo, still WIP. Do not use it now. No guarantee it's working.**  
**Beamforming on a pair of 2T2R adapters gives almost no gain (+3dB, theoretically maximum) at all.**

What we know now:  
1. The RTL88X2E has proper VHT beamforming implementation in hardware/firmware, and the firmware/hardware does not check if it's working in AP/STA mode
2. The driver maintains an FSM for the sounding process (while there are maybe >= 3 versions of FSM code in the driver XD)
3. The sounding is triggered by the beamformer's FSM sending a VHT NDPA frame, then the hardware will automatically send the NDP packet after a SIFS time
4. The beamformee will reply with a VHT Compressed Beamforming Report (CBR) frame (which can be captured (e.g. Wireshark) and then transformed into the V matrix)
5. The beamformer captures the CBR frame, converts it into the V matrix (in hardware/firmware), then enables the TXBF
6. In my test, sometimes the RSSI increases suddenly by +2dB after sounding, but sometimes no increase at all. The ```psi``` and ```phi``` values in CBR frame do change when moving adapters. -- so, it seems working?

The RTL8812EU adapter can act as both beamformer & beamformee, simultaneously.
 - Beamformer: the role sending NDPA & NDP, receiving CBR frame, applying V matrix to TX 
 - Beamformee: the role receiving NDPA & NDP, calculate and send the CBR frame

Weird behaviors I don't understand: 
1. The beamformer and beamformee seem to only check the NDPA/CBR frame's receiver address with the stock (efuse) MAC address.
   Setting any MAC address is not working. The G_ID (Group ID, 0 or 63 in SU BF) / P_AID (Partial AID, 12bit) seems no use at all.
   Maybe the SU BF just applies the V matrix to every connection (RF part, basically) and the hardware/firmware does not need to know or care about the MAC address on the higher level?
2. The TX power override not work when TXBF is enabled. Why? It doesn't touch anything about the power table at all.
3. ... (I'm sure there will be more confusing issues)

### Usage
See ```/proc```.
The APIs are unstable and can be changed. No guarantee it's working.  
Some of the args do not affect anything in my test, but the Realtek driver filled them, ... and nobody has the datasheet, so I'm just following their code.  
(In my test -- it works by only giving the correct MAC address. And I've just filled all other things zero.)

Scenario: 
```
BFer: 00:66:77:88:99:aa
BFee: 00:11:22:33:44:55
```
1. On the beamformee side: 
```
# enable bf
# <en:0/1> <remote mac addr> <remote g_id> <remote p_aid>
echo "1 00:66:77:88:99:aa 0 0" > /proc/net/rtl88x2eu/<wlan0>/bf_monitor_conf
```
2. On the beamformer side:
```
# enable bf
# <en:0/1> <remote mac addr> <remote g_id> <remote p_aid>
echo "1 00:11:22:33:44:55 0 0" > /proc/net/rtl88x2eu/<wlan0>/bf_monitor_conf

# trigger sounding (send NDPA+NDP)
# <local mac addr> <remote mac addr> <p_aid> <g_id> <sounding_token, 0~63> <bandwidth, 20/40/80>
echo "00:66:77:88:99:aa 00:11:22:33:44:55 0 0 0 20" > /proc/net/rtl88x2eu/<wlan0>/bf_monitor_conf
```
The triggering can be done periodically.  

3. On the beamformer side, check the result:
```
cat /proc/net/rtl88x2eu/<wlan0>/bf_monitor_trig
```

To convert the CBR frame to the V matrix, you may want to check [80211BeamformingReport](https://github.com/Vito-Swift/dpkt-80211BeamformingReport) or [WiPiCap](https://github.com/watalabo/WiPiCap). 

### Some picture 

Measuring the VDET pin of the adapter's PA (can indicate TX status). The waveform stands for NDPA (high), SIFS slot (the ~0.2v part middle), NDP (high), and re-transmissions (as no bfee when taking the picture), respectively
![image](https://github.com/user-attachments/assets/50bc6bb0-0dae-4940-b354-8eaf719d8218)

One CBR frame, captured on the beamformer side, with 2 8812eu (2T2R) as bfer & bfee  
The capture in this screenshot is [CBR_cap_20_40_80M.pcapng](https://github.com/libc0607/rtl88x2eu-20230815/blob/beamforming_research/CBR_cap_20_40_80M.pcapng).
![image](https://github.com/user-attachments/assets/c7dbeb3f-4633-4ce1-b5fd-09692c63c784)  
