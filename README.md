## Transmit Beamforming in Monitor Mode 
A minimum re-implementation of using Transmit Single-User Beamforming in monitor mode (without establishing AP/STA connection)  

Why it can be useful:
 - Beamforming gives several dB gains on RSSI by focusing the beam in the right direction
 - The sounding report (Compressed Beamforming Report, CBR) contains 1. the SNR of the two streams seen on the RX side, which somehow can be used for dynamic bitrate controlling ...?, and 2. a feedback matrix, which can be further converted to the Channel State Information (CSI)

**This is a very early demo, still WIP. Use it at your own risk. No guarantee it's working.**  
**Beamforming on a pair of 2T2R adapters gives almost no gain (+3dB, theoretically maximum) at all, and only sees some benefit in the middle-range multipath environment.**

### Progress
The RTL8812EU adapter can act as both beamformer & beamformee, simultaneously.    
And, the beamforming only affects TX behavior -- so both sides need to trigger sounding (as beamformer) to maximize the benefit.  
Note: the chipset does not support RX beamforming. Only the Maximum Ratio Combining (MRC) is improving the minimum SNR requirement.  
 - Beamformer: the role sending NDPA & NDP, receiving CBR frame, applying V matrix to TX 
 - Beamformee: the role receiving NDPA & NDP, calculate and send the CBR frame

What we know now:  
1. The RTL88X2E has proper VHT beamforming implementation in hardware/firmware, and the firmware/hardware does not check if it's working in AP/STA mode
2. The driver maintains an FSM for the sounding process (while there are maybe >= 3 versions of FSM code in the driver XD)
3. The sounding is triggered by the beamformer's FSM sending a VHT NDPA frame, then the hardware will automatically send the NDP packet after a SIFS time
4. The beamformee will reply with a VHT CBR frame (which can be captured (e.g. Wireshark) and then transformed into the V matrix)
5. The beamformer captures the CBR frame, converts it into the V matrix (in hardware/firmware), then enables the TXBF
6. In my test, setting ```bf_monitor_en``` gives several dB differences in the multipath environment, but sometimes no obvious gain at all. The ```psi``` and ```phi``` values in the CBR frame do change when moving adapters. -- so, it seems working...

Weird behaviors I don't understand: 
1. The beamformer and beamformee seem to only check the NDPA/CBR frame's receiver address with the stock (efuse) MAC address.
   Setting any MAC address is not working. The G_ID (Group ID, 0 or 63 in SU BF) / P_AID (Partial AID, 12bit) seems no use at all.
   Maybe the SU BF just applies the V matrix to every connection (RF part, basically), and the hardware/firmware does not need to know or care about the MAC address on the higher level?
2. ~~The TX power override not work when TXBF is enabled. Why? It doesn't touch anything about the power table at all.~~ Seems working now, but I didn't touch anything. (
3. How does the chipset determine which beamforming matrix to use when transmitting? By checking the G_ID/P_AID/MAC? Even if the MAC address in the injected packet's IEEE 802.11 data header changes, the RSSI benefit still can be seen... So it's just simply applying to all TX traffic?
4. ... (I'm sure there will be more confusing issues)

### Usage
Scenario: 
Two RTL8812EU adapters with this branch driver, or RTL88x2CU adapter with [this driver](https://github.com/libc0607/rtl88x2cu-20230728), **STBC disabled, injecting packets only in HT MCS 0\~7 or VHT MCS 0\~9**  
```
Local MAC Address: 00:66:77:88:99:aa
Remote MAC Address: 00:11:22:33:44:55
# Use the stock address (in efuse)! e.g. the LB-LINK starts with 98:03:cf ...
```

The NDPA + NDP will retransmit if CBR has not been received in ~50us (measured by oscilloscope).  
The ```ACK_TIMEOUT``` affects the waiting time. By default, it's ```33```us, and can be tuned up to ```255```us.  
So the sounding range is limited to about 35\~40km.  

To convert the CBR frame to the V matrix (Channel State Information), you need to capture the frame on the BFer side (by Wireshark or sth) first, then check [80211BeamformingReport](https://github.com/Vito-Swift/dpkt-80211BeamformingReport) or [WiPiCap](https://github.com/watalabo/WiPiCap). 

#### Script
See ```bf_mon.sh```. Should be run on both sides.
```
# bf_mon.sh start <WLAN_DRV> <NIC> <LOCAL_MAC> <REMOTE_MAC> <Bandwidth:20/40/80> <ACK_TIMEOUT:33~255> <INTERVAL:second>
# bf_mon.sh stop  <WLAN_DRV> <NIC>"

# Example: 
bf_mon.sh start rtl88x2eu wlan0 00:66:77:88:99:aa 00:11:22:33:44:55 20 255 0.1
bf_mon.sh stop rtl88x2eu wlan0
```

#### Manually, if you will...

See ```/proc```.
The APIs are unstable and can be changed. No guarantee it's working.  
Some of the args do not affect anything in my test, but the Realtek driver filled them, ... and nobody has the datasheet, so I'm just following their code.  
(In my test -- it works by only giving the correct MAC address. And I've just filled all other things zero.)
1. On the beamformee side: 
```
# init bf (init registers, enable interrupt, etc.)
# <en:0/1> <remote mac addr> <remote g_id> <remote p_aid>
# In this case just set g_id and p_aid to zero
echo "1 00:66:77:88:99:aa 0 0" > /proc/net/rtl88x2eu/<wlan0>/bf_monitor_conf
```
2. On the beamformer side:
```
# 1. Enable bf
# <en:0/1> <remote mac addr> <remote g_id> <remote p_aid>
# In this case just set g_id and p_aid to zero
echo "1 00:11:22:33:44:55 0 0" > /proc/net/rtl88x2eu/<wlan0>/bf_monitor_conf

# 2. Trigger sounding (send NDPA+NDP)
# <local mac addr> <remote mac addr> <p_aid> <g_id> <sounding_token, 0~63> <bandwidth, 20/40/80>
# Token (seq num) is set in NDPA frame, then the bfee will send the token back in CBR frame
# In this case just set g_id and p_aid to zero
# The triggering can be done periodically, to keep the matrix up-to-date.  
echo "00:66:77:88:99:aa 00:11:22:33:44:55 0 0 0 20" > /proc/net/rtl88x2eu/<wlan0>/bf_monitor_trig

# 3. Apply the matrix to transmitting
# <en: 0/1>
echo "1" > /proc/net/rtl88x2eu/<wlan0>/bf_monitor_en
# and then, the RSSI on the receiver side should increase by several dB ...
```

3. On the beamformer side, check the status:
```
# CBR feedback information 
cat /proc/net/rtl88x2eu/<wlan0>/bf_monitor_trig
# config
cat /proc/net/rtl88x2eu/<wlan0>/bf_monitor_conf
```

4. Disable beamforming mode
```
# only need to set <en:0>
echo "0 00:00:00:00:00:00 0 0" > /proc/net/rtl88x2eu/<wlan0>/bf_monitor_conf

# Then you can use STBC or MCS8~15 again.
``` 

Check ```dmesg``` for more details.

### Some picture 

Measuring the VDET pin of the adapter's PA (can indicate TX status). The waveform stands for NDPA (high), SIFS slot (the ~0.2v part middle), NDP (high), and re-transmissions (as no bfee when taking the picture), respectively
![image](https://github.com/user-attachments/assets/50bc6bb0-0dae-4940-b354-8eaf719d8218)

One CBR frame, captured on the beamformer side, with 2 8812eu (2T2R) as bfer & bfee  
The capture in this screenshot is [CBR_cap_20_40_80M.pcapng](https://github.com/libc0607/rtl88x2eu-20230815/blob/beamforming_research/CBR_cap_20_40_80M.pcapng).
![image](https://github.com/user-attachments/assets/c7dbeb3f-4633-4ce1-b5fd-09692c63c784)  

