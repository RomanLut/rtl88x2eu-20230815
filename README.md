## Branch: 5MHz BW 


See [this commit](https://github.com/libc0607/rtl88x2eu-20230815/commit/67dbbff1f01b8edd5b532c2a2c6e719452740ff5).  
```
odm_set_bb_reg(dm, R_0x9b4, 0x00000700, 0x4);  // DAC clock for 5MHz BW
```

**Note**: 8/20/2024 It's been reported that the current setting has some leakage (mirror?) out of the band. Need more investigation.   

Disclaimer: The patch is based on my guess. There's no guarantee of its performance. There are no "Chinese-only documents" of all those hidden register bits.  
If it's not working, ask [The Crab](https://www.realtek.com/Article/Index?menu_id=847) for this feature.  
The patch may damage your hardware and I'm not gonna pay for that, so use it at your own risk.  


![47c0f34d645e6f2c0f1a897120b9a59](https://github.com/user-attachments/assets/a908cfd2-ce8b-4436-8805-181dde37b202)  
![3d7ddf8c4ae25e07abfea4d2fa91f71](https://github.com/user-attachments/assets/be5ba7f4-208f-45cd-b6c2-48a40c3c2f9e)  
