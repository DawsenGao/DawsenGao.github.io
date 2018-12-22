---
layout: post
title: MTK SDK wifi参数生效过程
description: MTK SDK wifi参数生效过程
category: MTK
---

由于一直做QCA方案的路由，对于MTK wifi的一些配置流程不熟悉。这里简单追下，无线参数生效的过程。

一般页面修改无线参数后，会将对应参数配置到/etc/config/wireless，然后执行/etc/init.d/network restart/reload。

对于qca，使用的是qcawifi.sh脚本生效配置，而mtk的无线驱动使用mtxxx.dat配置文件来生效相关的无线配置,这个.dat文件是通过uci2dat将/etc/config/wireless配置进行转化的。

从执行/etc/init.d/network restart开始，关注无线相关的配置过程

1./etc/init.d/network文件中，执行restart函数

```shell
restart() {         
        rmmod hw_nat 
        rmmod nf_sc 
        ifdown -a   
        sleep 1     
        trap '' TERM                
        stop "$@"                            
        start "$@"                           
        #insmod /lib/modules/ralink/hw_nat.ko
        insmod /lib/modules/ralink/nf_sc.ko
} 
```
2. 调用service_running函数

```
service_running() {
        rmmod hw_nat
        rmmod nf_sc
        ubus -t 30 wait_for network.interface
        /sbin/wifi reload_legacy                      
        sleep 3                 
        init_switch
        insmod /lib/modules/ralink/hw_nat.ko
        insmod /lib/modules/ralink/nf_sc.ko 
        mii_mgr -s -p 0 -r 0 -v 3300       
        mii_mgr -s -p 1 -r 0 -v 3300
        mii_mgr -s -p 2 -r 0 -v 3300
        mii_mgr -s -p 3 -r 0 -v 3300
}        
```

3. 执行/sbin/wifi reload_legacy ,调用wifi_reload_legacy函数

```
reload_legacy) wifi_reload_legacy "$2";;
```

4. /sbin/wifi文件中

```
wifi_reload_legacy() {              
        _wifi_updown "disable" "$1"
        scan_wifi
        _wifi_updown "enable" "$1"             
}
```

5. 执行_wifi_updown "enable"

```
_wifi_updown() {    
		#这里介绍下shell的:-句法，${2:-$DEVICES}实际的值等于$DEVICES的值
		#$DEVICES有赋值，我这里是mt7628 mt7612e
        for device in ${2:-$DEVICES}; do (
                config_get disabled "$device" disabled
                [ 1 == "$disabled" ] && {             
                        echo "'$device' is disabled"
                        set disable
                }                                                         
                config_get iftype "$device" type  
                #typ=mt7628 mt7612e
                if eval "type ${1}_$iftype" 2>/dev/null >/dev/null; then
                        eval "scan_$iftype '$device'"  
                        #调用scan_mt7628 7628 或 scan_mt7612e mt7612e
                  
                        eval "${1}_$iftype '$device'" || echo "$device($iftype): ${1} failed"
                        #调用 enable_mt7628  mt7628 或enable_mt7612e  mt7612e
                        
                elif [ ! -f /lib/netifd/wireless/$iftype.sh ]; then
                        echo "$device($iftype): Interface type not supported"
                fi                        
        ); done       
}

```
6. 调用的/lib/wifi/mt7628.sh和mt7612e.sh

```
scan_mt7628() {
        scan_ralink_wifi mt7628 mt7628
}
enable_mt7628() {
        enable_ralink_wifi mt7628 mt7628
}
```

7. /lib/wifi/ralink_common.sh函数

```
scan_ralink_wifi() {                                                 
    local device="$1"                                              
    local module="$2"                                    
    echo "scan_ralink_wifi($1,$2,$3,$4)" >>/tmp/wifi.log  
    repair_wireless_uci                                            
    sync_uci_with_dat $device /etc/wireless/$device/$device.dat    
} 
```

8. sync_uci_with_dat函数中调用uci2dat生成配置文件

```
sync_uci_with_dat() {                                              
    echo "sync_uci_with_dat($1,$2,$3,$4)" >>/tmp/wifi.log          
    local device="$1"                                              
    local datpath="$2"                                             
    uci2dat -d $device -f $datpath > /tmp/uci2dat.log  
    #uci2dat -d mt7628 -f /etc/wireless/mt7628/mt7628.dat
    #uci2dat -d mt7612e -f /etc/wireless/mt7612e/mt7612e.dat
} 
```

## 

下面可以看到uci2dat的使用方法，及/etc/config/wireless配置的uci option对应于.dat文件的无线参数,通过对应的关系，指导我们在/etc/config/wirless文件配置中如何配置相应的参数！

```
root@OpenWrt:/# uci2dat 
uci2dat  -- a tool to translate uci config (/etc/config/wireless)
            into ralink driver dat.

Usage:
    uci2dat -d <dev-name> -f <dat-path>

Arguments:
    -d <dev-name>   device name, mt7620, eg.
    -f <dat-path>   dat file path.

Supported keywords:
     [uci-key]          [dat-key]               [default]
  0. region             CountryRegion           1
  1. aregion            CountryRegionABand      7
  2. country            CountryCode             (null)
  3. wifimode           WirelessMode            9
  4. txrate             TxRate                  0
  5. channel            Channel                 0
  6. basicrate          BasicRate               15
  7. beacon             BeaconPeriod            100
  8. dtim               DtimPeriod              1
  9. txpower            TxPower                 100
 10. disolbc            DisableOLBC             0
 11. bgprotect          BGProtection            0
 12. txantenna          TxAntenna               (null)
 13. rxantenna          RxAntenna               (null)
 14. txpreamble         TxPreamble              0
 15. rtsthres           RTSThreshold            2347
 16. fragthres          FragThreshold           2346
 17. txburst            TxBurst                 1
 18. pktaggre           PktAggregate            0
 19. turborate          TurboRate               0
 20. wmm                WmmCapable              1
 21. apsd               APSDCapable             1
 22. dls                DLSCapable              0
 23. apaifsn            APAifsn                 3;7;1;1
 24. apcwmin            APCwmin                 4;4;3;2
 25. apcwmax            APCwmax                 6;10;4;3
 26. aptxop             APTxop                  0;0;94;47
 27. apacm              APACM                   0;0;0;0
 28. bssaifsn           BSSAifsn                3;7;2;2
 29. bsscwmin           BSSCwmin                4;4;3;2
 30. bsscwmax           BSSCwmax                10;10;4;3
 31. bsstxop            BSSTxop                 0;0;94;47
 32. bssacm             BSSACM                  0;0;0;0
 33. ackpolicy          AckPolicy               0;0;0;0
 34. noforward          NoForwarding            0
 35. hidden             HideSSID                0
 36. shortslot          ShortSlot               1
 37. channel            AutoChannelSelect       2
 38. ieee8021x          IEEE8021X               0
 39. ieee80211h         IEEE80211H              0
 40. csperiod           CSPeriod                10
 41. wirelessevent      WirelessEvent           0
 42. idsenable          IdsEnable               0
 43. preauth            PreAuth                 0
 44. rekeyinteval       RekeyInterval           0
 45. pmkcacheperiod     PMKCachePeriod          10
 46. wds_enable         WdsEnable               0
 47. wds_enctype        WdsEncrypType           NONE
 48. auth_server        RADIUS_Server           (null)
 49. auth_port          RADIUS_Port             (null)
 50. radius_key1        RADIUS_Key1             (null)
 51. radius_key2        RADIUS_Key2             (null)
 52. radius_key2        RADIUS_Key3             (null)
 53. radius_key4        RADIUS_Key4             (null)
 54. own_ip_addr        own_ip_addr             192.168.5.234
 55. eapifname          EAPifname               (null)
 56. preauthifname      PreAuthifname           br-lan
 57. ht_htc             HT_HTC                  0
 58. ht_rdg             HT_RDG                  0
 59. ht_extcha          HT_EXTCHA               0
 60. ht_linkadapt       HT_LinkAdapt            0
 61. ht_opmode          HT_OpMode               0
 62. bw                 HT_BW                   0
 63. bw                 VHT_BW                  0
 64. vht2ndchannel      VHT_Sec80_Channel       (null)
 65. vht_sgi            VHT_SGI                 1
 66. vht_stbc           VHT_STBC                0
 67. vht_bw_sig         VHT_BW_SIGNAL           0
 68. vht_disnonvht      VHT_DisallowNonVHT      (null)
 69. vht_ldpc           VHT_LDPC                1
 70. ht_autoba          HT_AutoBA               1
 71. ht_amsdu           HT_AMSDU                (null)
 72. ht_bawinsize       HT_BAWinSize            64
 73. ht_gi              HT_GI                   1
 74. ht_mcs             HT_MCS                  33
 75. wscmanufacturer    WscManufacturer         (null)
 76. wscmodelname       WscModelName            (null)
 77. wscdevicename      WscDeviceName           (null)
 78. wscmodelnumber     WscModelNumber          (null)
 79. wscserialnumber    WscSerialNumber         (null)
 80. fixedtxmode        FixedTxMode             0
 81. autoprovision      AutoProvisionEn         0
 82. freqdelta          FreqDelta               0
 83. carrierdetect      CarrierDetect           0
 84. preantswitch       PreAntSwitch            1
 85. phyratelimit       PhyRateLimit            0
 86. debugflags         DebugFlags              0
 87. fineagc            FineAGC                 0
 88. streammode         StreamMode              0
 89. dfs_lowlimit       DfsLowerLimit           0
 90. dfs_uplimit        DfsUpperLimit           0
 91. dfs_outdoor        DfsOutdoor              0
 92. fixdfslimit        FixDfsLimit             0
 93. lpsradarth         LongPulseRadarTh        0
 94. avgrssireq         AvgRssiReq              0
 95. dfs_r66            DFS_R66                 0
 96. blockch            BlockCh                 (null)
 97. greenap            GreenAP                 0
 98. rekeymethod        RekeyMethod             TIME
 99. mesh_autolink      MeshAutoLink            0
100. mesh_authmode      MeshAuthMode            (null)
101. mesh_enctype       MeshEncrypType          (null)
102. mesh_defkey        MeshDefaultkey          0
103. mesh_wepkey        MeshWEPKEY              (null)
104. mesh_wpakey        MeshWPAKEY              (null)
105. mesh_id            MeshId                  (null)
106. hscount            HSCounter               0
107. ht_badec           HT_BADecline            0
108. ht_stbc            HT_STBC                 0
109. ht_ldpc            HT_LDPC                 1
110. ht_txstream        HT_TxStream             1
111. ht_rxstream        HT_RxStream             1
112. ht_protect         HT_PROTECT              1
113. ht_distkip         HT_DisallowTKIP         0
114. ht_bsscoexist      HT_BSSCoexistence       0
115. wsc_confmode       WscConfMode             0
116. wsc_confstatus     WscConfStatus           2
117. wcntest            WCNTest                 0
118. wds_phymode        WdsPhyMode              (null)
119. radius_acctserver  RADIUS_Acct_Server      (null)
120. radius_acctport    RADIUS_Acct_Port        1813
121. radius_acctkey     RADIUS_Acct_Key         (null)
122. ethifname          Ethifname               (null)
123. session_intv       session_timeout_interval        0
124. idle_intv          idle_timeout_interval   0
125. tgnwifitest        TGnWifiTest             0
126. efusebufmode       EfuseBufferMode         0
127. e2paccmode         E2pAccessMode           1
128. radio              RadioOn                 1
129. bw_enable          BW_Enable               0
130. bw_root            BW_Root                 0
131. bw_priority        BW_Priority             (null)
132. bw_grtrate         BW_Guarantee_Rate       (null)
133. bw_maxrate         BW_Maximum_Rate         (null)
134. autoch_skip        AutoChannelSkipList     (null)
135. wsc_confmethod     WscConfMethod           (null)
136. wsc_keyascii       WscKeyASCII             (null)
137. wsc_security       WscSecurityMode         (null)
138. wsc_4digitpin      Wsc4digitPinCode        (null)
139. wsc_vendorpin      WscVendorPinCode        (null)
140. wsc_v2             WscV2Support            (null)
141. mimops             HT_MIMOPS               3
142. g256qam            G_BAND_256QAM           0
143. dbdc               DBDC_MODE               0
144. txbf               txbf                    0
145. igmpsnoop          IgmpSnEnable            1
146. mutxrxenable       MUTxRxEnable            0
147. itxbfencond        ITxBfEnCond             0
148. ssid
149. encryption
150. hidden
151. cipher
152. key
```