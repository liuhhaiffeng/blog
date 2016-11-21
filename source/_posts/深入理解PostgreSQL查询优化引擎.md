---
title: 深入理解PostgreSQL查询优化引擎
comments: true
date: 2016-11-21 05:57:19
tags: [数据库,database,PostgreSQL]
categories: [slides]
---
# 深入理解PostgreSQL查询优化引擎

标签（空格分隔）： PostgreSQL

---


![幻灯片1.png-34.1kB][1]
![幻灯片2.png-52.9kB][3]
<!--more-->
![幻灯片3.png-58.1kB][4]
![幻灯片4.png-346.7kB][5]
![幻灯片5.png-216.4kB][6]
![幻灯片6.png-247.6kB][7]
![幻灯片7.png-182.8kB][8]
![幻灯片8.png-143.6kB][9]
![幻灯片9.png-189.5kB][10]
![幻灯片10.png-186.6kB][11]
![幻灯片11.png-189.5kB][12]
![幻灯片12.png-169.6kB][13]
![幻灯片13.png-54.3kB][14]
![幻灯片14.png-252kB][15]
![幻灯片15.png-249.6kB][16]
![幻灯片16.png-259.8kB][17]
![幻灯片17.png-243kB][18]
![幻灯片18.png-238.3kB][19]
![幻灯片19.png-178.8kB][20]
![幻灯片20.png-218.8kB][21]
![幻灯片21.png-198.1kB][22]
![幻灯片22.png-163.4kB][23]
![幻灯片23.png-157.3kB][24]
![幻灯片24.png-192.3kB][25]
![幻灯片25.png-215.7kB][26]
![幻灯片26.png-216.6kB][27]
![幻灯片27.png-85.8kB][28]
![幻灯片28.png-68kB][29]
![幻灯片29.png-49.3kB][30]
![幻灯片30.png-221.9kB][31]
![幻灯片31.png-47.7kB][32]
![幻灯片32.png-25.4kB][33]
![幻灯片33.png-44.2kB][34]
![幻灯片34.png-50.7kB][35]
![幻灯片35.png-91.5kB][36]
![幻灯片36.png-96kB][37]
![幻灯片37.png-95.9kB][38]
![幻灯片38.png-104.6kB][39]
![幻灯片39.png-223.8kB][40]
![幻灯片40.png-55.7kB][41]
![幻灯片41.png-34.2kB][42]
![幻灯片42.png-97.4kB][43]
![幻灯片43.png-67kB][44]
![幻灯片44.png-76.7kB][45]
![幻灯片45.png-14.2kB][46]
![幻灯片46.png-77.2kB][47]
![幻灯片47.png-79.8kB][48]
![幻灯片48.png-76.8kB][49]
![幻灯片49.png-69kB][50]
![幻灯片50.png-21.6kB][51]
![幻灯片51.png-58.9kB][52]
![幻灯片52.png-69.4kB][53]
![幻灯片53.png-78.3kB][54]
![幻灯片54.png-72.1kB][55]
![幻灯片55.png-67.6kB][56]
![幻灯片56.png-64.2kB][57]
![幻灯片57.png-68.4kB][58]
![幻灯片58.png-61.8kB][59]
![幻灯片59.png-224.3kB][60]
![幻灯片60.png-52kB][61]
![幻灯片61.png-40.8kB][62]
![幻灯片62.png-34.7kB][63]
![幻灯片63.png-79.1kB][64]
![幻灯片64.png-44.6kB][65]
![幻灯片65.png-242kB][66]
![幻灯片66.png-163.9kB][67]
![幻灯片67.png-98.2kB][68]
![幻灯片68.png-99.8kB][69]
![幻灯片69.png-220.4kB][70]
![幻灯片70.png-56.4kB][71]
![幻灯片71.png-56.1kB][72]
![幻灯片72.png-43.9kB][73]
![幻灯片73.png-51.4kB][74]
![幻灯片74.png-68.6kB][75]
![幻灯片75.png-52.5kB][76]
![幻灯片76.png-21kB][77]
![幻灯片77.png-73.2kB][78]
![幻灯片78.png-161.6kB][79]
![幻灯片79.png-158.5kB][80]
![幻灯片80.png-38.9kB][81]
![幻灯片81.png-31.9kB][82]
![幻灯片82.png-49.3kB][83]
![幻灯片83.png-91.1kB][84]
![幻灯片84.png-83.3kB][85]
![幻灯片85.png-223.3kB][86]
![幻灯片86.png-185.2kB][87]
![幻灯片87.png-163.4kB][88]
![幻灯片88.png-209.2kB][89]
![幻灯片89.png-259kB][90]
![幻灯片90.png-205.1kB][91]
![幻灯片91.png-244.9kB][92]
![幻灯片92.png-211.1kB][93]
![幻灯片93.png-195kB][94]
![幻灯片94.png-199.7kB][95]
![幻灯片95.png-100.8kB][96]
![幻灯片96.png-112.2kB][97]
![幻灯片97.png-115.5kB][98]
![幻灯片98.png-184.1kB][99]
![幻灯片99.png-71.7kB][100]
![幻灯片100.png-272.2kB][101]
![幻灯片101.png-45.9kB][102]
![幻灯片102.png-81.1kB][103]
![幻灯片103.png-86.6kB][104]
![幻灯片104.png-58.6kB][105]
![幻灯片105.png-64.6kB][106]
![幻灯片106.png-54.3kB][107]
![幻灯片107.png-70.8kB][108]
![幻灯片108.png-63.4kB][109]
![幻灯片109.png-54.3kB][110]
![幻灯片110.png-67.8kB][111]
![幻灯片111.png-54.4kB][112]
![幻灯片112.png-112.9kB][113]
![幻灯片113.png-129.9kB][114]
![幻灯片114.png-41.9kB][115]
![幻灯片115.png-91.1kB][116]
![幻灯片116.png-81.5kB][117]
![幻灯片117.png-32.8kB][118]
![幻灯片118.png-35.9kB][119]
![幻灯片119.png-93.4kB][120]
![幻灯片120.png-48.5kB][121]
![幻灯片121.png-44.8kB][122]
![幻灯片122.png-45kB][123]
![幻灯片123.png-28.6kB][124]
![幻灯片124.png-64.6kB][125]
![幻灯片125.png-69.1kB][126]
![幻灯片126.png-91.8kB][127]
![幻灯片127.png-71.2kB][128]
![幻灯片128.png-90.4kB][129]
![幻灯片129.png-24kB][130]
![幻灯片130.png-68.2kB][131]
![幻灯片131.png-69.1kB][132]
![幻灯片132.png-159.6kB][133]


  [1]: http://static.zybuluo.com/shenyuflying/jvqhy0z9ktshjd06tcvfifcm/%E5%B9%BB%E7%81%AF%E7%89%871.png
  [2]: http://static.zybuluo.com/shenyuflying/paz273e50wa24a7wkufstcp2/%E5%B9%BB%E7%81%AF%E7%89%872.png
  [3]: http://static.zybuluo.com/shenyuflying/q0yatpul7e0m3iiunk9j389t/%E5%B9%BB%E7%81%AF%E7%89%872.png
  [4]: http://static.zybuluo.com/shenyuflying/877ir8ozck08wdhd1h4t1rvi/%E5%B9%BB%E7%81%AF%E7%89%873.png
  [5]: http://static.zybuluo.com/shenyuflying/qe612v7qpadzo6ggqia7yahq/%E5%B9%BB%E7%81%AF%E7%89%874.png
  [6]: http://static.zybuluo.com/shenyuflying/s7omh9y4geeso8kk6sxzcype/%E5%B9%BB%E7%81%AF%E7%89%875.png
  [7]: http://static.zybuluo.com/shenyuflying/27ru82cqcgy7tvakzka78tmq/%E5%B9%BB%E7%81%AF%E7%89%876.png
  [8]: http://static.zybuluo.com/shenyuflying/3k5pmccouj6by0mjk5k00fu4/%E5%B9%BB%E7%81%AF%E7%89%877.png
  [9]: http://static.zybuluo.com/shenyuflying/pd11sv4ai1g6j080gde6w01l/%E5%B9%BB%E7%81%AF%E7%89%878.png
  [10]: http://static.zybuluo.com/shenyuflying/fh2x3tmnoweqlfh4a5bw4p9r/%E5%B9%BB%E7%81%AF%E7%89%879.png
  [11]: http://static.zybuluo.com/shenyuflying/gneunqz5i06l7wgyq38sy6ot/%E5%B9%BB%E7%81%AF%E7%89%8710.png
  [12]: http://static.zybuluo.com/shenyuflying/dvcz7gnogz1vn4ws2f0zdblw/%E5%B9%BB%E7%81%AF%E7%89%8711.png
  [13]: http://static.zybuluo.com/shenyuflying/c1axvfd3p7ykss8cpem96wdw/%E5%B9%BB%E7%81%AF%E7%89%8712.png
  [14]: http://static.zybuluo.com/shenyuflying/uteqki84bgyx1b4ij0ecm10f/%E5%B9%BB%E7%81%AF%E7%89%8713.png
  [15]: http://static.zybuluo.com/shenyuflying/nc50r7o51kp7i0560p83rjqy/%E5%B9%BB%E7%81%AF%E7%89%8714.png
  [16]: http://static.zybuluo.com/shenyuflying/ujt8bd94pm11gy103un86174/%E5%B9%BB%E7%81%AF%E7%89%8715.png
  [17]: http://static.zybuluo.com/shenyuflying/rfwsqz3td7rcohd62dxwnlbx/%E5%B9%BB%E7%81%AF%E7%89%8716.png
  [18]: http://static.zybuluo.com/shenyuflying/hlzx8jw3mg4vru8me9f20p25/%E5%B9%BB%E7%81%AF%E7%89%8717.png
  [19]: http://static.zybuluo.com/shenyuflying/igbaih7nd7n9j63gj75sfip6/%E5%B9%BB%E7%81%AF%E7%89%8718.png
  [20]: http://static.zybuluo.com/shenyuflying/dwrcbxwssvz9754fz780af1q/%E5%B9%BB%E7%81%AF%E7%89%8719.png
  [21]: http://static.zybuluo.com/shenyuflying/wvae67x2489j8zyr7svz5vmv/%E5%B9%BB%E7%81%AF%E7%89%8720.png
  [22]: http://static.zybuluo.com/shenyuflying/5p6jb3dfmdulbhpjpb6yd4iq/%E5%B9%BB%E7%81%AF%E7%89%8721.png
  [23]: http://static.zybuluo.com/shenyuflying/jzti5tc3oh4c15lt1xw2rvwn/%E5%B9%BB%E7%81%AF%E7%89%8722.png
  [24]: http://static.zybuluo.com/shenyuflying/b1dsgnl6btuqbxi87mfh6hxp/%E5%B9%BB%E7%81%AF%E7%89%8723.png
  [25]: http://static.zybuluo.com/shenyuflying/7qjwrridvj2377ni5kt7xjzq/%E5%B9%BB%E7%81%AF%E7%89%8724.png
  [26]: http://static.zybuluo.com/shenyuflying/4jlacvwjuhpq9jmay8ynp792/%E5%B9%BB%E7%81%AF%E7%89%8725.png
  [27]: http://static.zybuluo.com/shenyuflying/bvxwj9y7473g4ecjs2o1cqwy/%E5%B9%BB%E7%81%AF%E7%89%8726.png
  [28]: http://static.zybuluo.com/shenyuflying/ys2z3oqmnnbtj8y5n8zeihbo/%E5%B9%BB%E7%81%AF%E7%89%8727.png
  [29]: http://static.zybuluo.com/shenyuflying/nxdh607v7ilqm4s0h1g06g1j/%E5%B9%BB%E7%81%AF%E7%89%8728.png
  [30]: http://static.zybuluo.com/shenyuflying/gnkzpxs0qwqxaeyof6rdqrsn/%E5%B9%BB%E7%81%AF%E7%89%8729.png
  [31]: http://static.zybuluo.com/shenyuflying/e4mwg6j78nspoldnyxxhsesa/%E5%B9%BB%E7%81%AF%E7%89%8730.png
  [32]: http://static.zybuluo.com/shenyuflying/pxyvfup0wqa62zd0qe870trb/%E5%B9%BB%E7%81%AF%E7%89%8731.png
  [33]: http://static.zybuluo.com/shenyuflying/te79hytppmsdmjwvtt30bzm8/%E5%B9%BB%E7%81%AF%E7%89%8732.png
  [34]: http://static.zybuluo.com/shenyuflying/6nfsluzqw842bz7mzsepiak8/%E5%B9%BB%E7%81%AF%E7%89%8733.png
  [35]: http://static.zybuluo.com/shenyuflying/mofyhkdecnu2hqmqghij9euu/%E5%B9%BB%E7%81%AF%E7%89%8734.png
  [36]: http://static.zybuluo.com/shenyuflying/vx130whs4kxw41wlp7fam4v9/%E5%B9%BB%E7%81%AF%E7%89%8735.png
  [37]: http://static.zybuluo.com/shenyuflying/tg2wmoqqlab9z4ewf91o1f4a/%E5%B9%BB%E7%81%AF%E7%89%8736.png
  [38]: http://static.zybuluo.com/shenyuflying/57ncsasqxb490r5pob5zxppb/%E5%B9%BB%E7%81%AF%E7%89%8737.png
  [39]: http://static.zybuluo.com/shenyuflying/81c4038o58eqleqxn788verr/%E5%B9%BB%E7%81%AF%E7%89%8738.png
  [40]: http://static.zybuluo.com/shenyuflying/rs1e3zr2kqa6van4eh0mrpyr/%E5%B9%BB%E7%81%AF%E7%89%8739.png
  [41]: http://static.zybuluo.com/shenyuflying/8ih0p2q9o4oldl6densuoj5f/%E5%B9%BB%E7%81%AF%E7%89%8740.png
  [42]: http://static.zybuluo.com/shenyuflying/7nkuifedh2k5qedxu5vfax9z/%E5%B9%BB%E7%81%AF%E7%89%8741.png
  [43]: http://static.zybuluo.com/shenyuflying/pse5xe03kr9exunswbf3l3h2/%E5%B9%BB%E7%81%AF%E7%89%8742.png
  [44]: http://static.zybuluo.com/shenyuflying/flzy8k79q578humvjx24yyiv/%E5%B9%BB%E7%81%AF%E7%89%8743.png
  [45]: http://static.zybuluo.com/shenyuflying/vtu7pkhkyoaqb65y7wnjmsbt/%E5%B9%BB%E7%81%AF%E7%89%8744.png
  [46]: http://static.zybuluo.com/shenyuflying/gbbm3k631ubdckxb80zkn8fz/%E5%B9%BB%E7%81%AF%E7%89%8745.png
  [47]: http://static.zybuluo.com/shenyuflying/su06h9vnli6y0zk05p363gcw/%E5%B9%BB%E7%81%AF%E7%89%8746.png
  [48]: http://static.zybuluo.com/shenyuflying/znrhw1ffxnf9e8cb6ro8xuwr/%E5%B9%BB%E7%81%AF%E7%89%8747.png
  [49]: http://static.zybuluo.com/shenyuflying/1y2r4io219t5fenuqrydr10e/%E5%B9%BB%E7%81%AF%E7%89%8748.png
  [50]: http://static.zybuluo.com/shenyuflying/ht513ei7a83t37d75h18443x/%E5%B9%BB%E7%81%AF%E7%89%8749.png
  [51]: http://static.zybuluo.com/shenyuflying/z1mabaka4uoln6c66582cg09/%E5%B9%BB%E7%81%AF%E7%89%8750.png
  [52]: http://static.zybuluo.com/shenyuflying/nhyk7drut2ow12as11589zis/%E5%B9%BB%E7%81%AF%E7%89%8751.png
  [53]: http://static.zybuluo.com/shenyuflying/kxn8p2rt1df2vhnfetgp5egd/%E5%B9%BB%E7%81%AF%E7%89%8752.png
  [54]: http://static.zybuluo.com/shenyuflying/mdnn58l0he99pds8z4zgs5ag/%E5%B9%BB%E7%81%AF%E7%89%8753.png
  [55]: http://static.zybuluo.com/shenyuflying/hnzvkgnfxzfonq9t7kin83io/%E5%B9%BB%E7%81%AF%E7%89%8754.png
  [56]: http://static.zybuluo.com/shenyuflying/360eexwape2q8ou1nwmv69jv/%E5%B9%BB%E7%81%AF%E7%89%8755.png
  [57]: http://static.zybuluo.com/shenyuflying/4o1bqsibwpfuzdewjsfguzsr/%E5%B9%BB%E7%81%AF%E7%89%8756.png
  [58]: http://static.zybuluo.com/shenyuflying/9murlz88jqywuw1hpwwzqnci/%E5%B9%BB%E7%81%AF%E7%89%8757.png
  [59]: http://static.zybuluo.com/shenyuflying/35ts93z58d5qgbxtal7v270b/%E5%B9%BB%E7%81%AF%E7%89%8758.png
  [60]: http://static.zybuluo.com/shenyuflying/u8snipur77lrvng9vqi0ol2w/%E5%B9%BB%E7%81%AF%E7%89%8759.png
  [61]: http://static.zybuluo.com/shenyuflying/l7x3b6cfa85z8renc7pfxbkw/%E5%B9%BB%E7%81%AF%E7%89%8760.png
  [62]: http://static.zybuluo.com/shenyuflying/y1x8b9drzne1btvpimgkjei9/%E5%B9%BB%E7%81%AF%E7%89%8761.png
  [63]: http://static.zybuluo.com/shenyuflying/gwacllvn1lcbh4uhaueiwjcf/%E5%B9%BB%E7%81%AF%E7%89%8762.png
  [64]: http://static.zybuluo.com/shenyuflying/qgv53tnut3enag48myneimn2/%E5%B9%BB%E7%81%AF%E7%89%8763.png
  [65]: http://static.zybuluo.com/shenyuflying/nf060s84rc3uont4ox6oxu8c/%E5%B9%BB%E7%81%AF%E7%89%8764.png
  [66]: http://static.zybuluo.com/shenyuflying/xwduo2m4xjggs7asddjdltmd/%E5%B9%BB%E7%81%AF%E7%89%8765.png
  [67]: http://static.zybuluo.com/shenyuflying/3n61zrj48mfybb7o8w6luqsr/%E5%B9%BB%E7%81%AF%E7%89%8766.png
  [68]: http://static.zybuluo.com/shenyuflying/4b4wpceng3if8w8jr4d13qpm/%E5%B9%BB%E7%81%AF%E7%89%8767.png
  [69]: http://static.zybuluo.com/shenyuflying/qh5mit4aluzj1vbu497r60rs/%E5%B9%BB%E7%81%AF%E7%89%8768.png
  [70]: http://static.zybuluo.com/shenyuflying/wd7q0nglu2m1gmqgylsputns/%E5%B9%BB%E7%81%AF%E7%89%8769.png
  [71]: http://static.zybuluo.com/shenyuflying/409tzkelzihcviagdhf3fkgx/%E5%B9%BB%E7%81%AF%E7%89%8770.png
  [72]: http://static.zybuluo.com/shenyuflying/c16k53hrvhrm7yh3s344a2dv/%E5%B9%BB%E7%81%AF%E7%89%8771.png
  [73]: http://static.zybuluo.com/shenyuflying/hp526n29ruvxasd15eey3a1l/%E5%B9%BB%E7%81%AF%E7%89%8772.png
  [74]: http://static.zybuluo.com/shenyuflying/mprs2s2ikgwu1m36xohw6wno/%E5%B9%BB%E7%81%AF%E7%89%8773.png
  [75]: http://static.zybuluo.com/shenyuflying/2dpfj20alz4j2uzlleu4bv81/%E5%B9%BB%E7%81%AF%E7%89%8774.png
  [76]: http://static.zybuluo.com/shenyuflying/11mxiix7u4splqpu6klczboa/%E5%B9%BB%E7%81%AF%E7%89%8775.png
  [77]: http://static.zybuluo.com/shenyuflying/ytj6q5vyrx143677wxq5t8o3/%E5%B9%BB%E7%81%AF%E7%89%8776.png
  [78]: http://static.zybuluo.com/shenyuflying/0x9wqoen87y9c2n2p9184iv8/%E5%B9%BB%E7%81%AF%E7%89%8777.png
  [79]: http://static.zybuluo.com/shenyuflying/7cnca1ykcslxwryh1uq6j34x/%E5%B9%BB%E7%81%AF%E7%89%8778.png
  [80]: http://static.zybuluo.com/shenyuflying/df3arrsao690srl48se8fkvx/%E5%B9%BB%E7%81%AF%E7%89%8779.png
  [81]: http://static.zybuluo.com/shenyuflying/gus683r55tvedbxlfd78so6m/%E5%B9%BB%E7%81%AF%E7%89%8780.png
  [82]: http://static.zybuluo.com/shenyuflying/5qgel78m9ca40bz7xy54hdak/%E5%B9%BB%E7%81%AF%E7%89%8781.png
  [83]: http://static.zybuluo.com/shenyuflying/2h35ocsc3u18q6qp60058x2m/%E5%B9%BB%E7%81%AF%E7%89%8782.png
  [84]: http://static.zybuluo.com/shenyuflying/i9q1lw0n0k90yyuwnek415hl/%E5%B9%BB%E7%81%AF%E7%89%8783.png
  [85]: http://static.zybuluo.com/shenyuflying/fp1yx59w46i1lp4fqt84dwdp/%E5%B9%BB%E7%81%AF%E7%89%8784.png
  [86]: http://static.zybuluo.com/shenyuflying/v8dmkit14lwm2b5nmyrmjmv1/%E5%B9%BB%E7%81%AF%E7%89%8785.png
  [87]: http://static.zybuluo.com/shenyuflying/aemdqnnhkeaul335c3whk16q/%E5%B9%BB%E7%81%AF%E7%89%8786.png
  [88]: http://static.zybuluo.com/shenyuflying/xr7054hjbfjc053fkryvmqxb/%E5%B9%BB%E7%81%AF%E7%89%8787.png
  [89]: http://static.zybuluo.com/shenyuflying/oz3mn1dyhtlqc9bikzsv2k70/%E5%B9%BB%E7%81%AF%E7%89%8788.png
  [90]: http://static.zybuluo.com/shenyuflying/e0xmpsqxtiik74yaxbuj2wsk/%E5%B9%BB%E7%81%AF%E7%89%8789.png
  [91]: http://static.zybuluo.com/shenyuflying/ej8ebpp4jwrw053lbzq061id/%E5%B9%BB%E7%81%AF%E7%89%8790.png
  [92]: http://static.zybuluo.com/shenyuflying/xk4ph7a9skq4oxk2a59tz9fb/%E5%B9%BB%E7%81%AF%E7%89%8791.png
  [93]: http://static.zybuluo.com/shenyuflying/g7muhvmx4z6n1wjx9yi5gy6d/%E5%B9%BB%E7%81%AF%E7%89%8792.png
  [94]: http://static.zybuluo.com/shenyuflying/m9xl9ve6qtry3szujmxybqb8/%E5%B9%BB%E7%81%AF%E7%89%8793.png
  [95]: http://static.zybuluo.com/shenyuflying/3kcsab5omgknjkokkgwj4rc3/%E5%B9%BB%E7%81%AF%E7%89%8794.png
  [96]: http://static.zybuluo.com/shenyuflying/pfqbtwgj0sq7jvr759ar23bo/%E5%B9%BB%E7%81%AF%E7%89%8795.png
  [97]: http://static.zybuluo.com/shenyuflying/p9psbbti0h8o8btjrn4hpnkz/%E5%B9%BB%E7%81%AF%E7%89%8796.png
  [98]: http://static.zybuluo.com/shenyuflying/khvv6ghh6cn36gic0xc63efh/%E5%B9%BB%E7%81%AF%E7%89%8797.png
  [99]: http://static.zybuluo.com/shenyuflying/qkgwtzbduin4tht0jdhpc2kz/%E5%B9%BB%E7%81%AF%E7%89%8798.png
  [100]: http://static.zybuluo.com/shenyuflying/a5eshwuv7t5vga617x3dtbrk/%E5%B9%BB%E7%81%AF%E7%89%8799.png
  [101]: http://static.zybuluo.com/shenyuflying/n26zwzezh8awl6m3eb3vvx3w/%E5%B9%BB%E7%81%AF%E7%89%87100.png
  [102]: http://static.zybuluo.com/shenyuflying/3upj4w83x5c5rg317g94bfy2/%E5%B9%BB%E7%81%AF%E7%89%87101.png
  [103]: http://static.zybuluo.com/shenyuflying/y1zcms1bcwt6r9xyknb4nq3h/%E5%B9%BB%E7%81%AF%E7%89%87102.png
  [104]: http://static.zybuluo.com/shenyuflying/167vpcyqp07hv2oacuddql7v/%E5%B9%BB%E7%81%AF%E7%89%87103.png
  [105]: http://static.zybuluo.com/shenyuflying/iluowtgt8asx3ctsvxtcah71/%E5%B9%BB%E7%81%AF%E7%89%87104.png
  [106]: http://static.zybuluo.com/shenyuflying/nr8rtqjxfslnqfgexjnype2p/%E5%B9%BB%E7%81%AF%E7%89%87105.png
  [107]: http://static.zybuluo.com/shenyuflying/vyzrzl9ko7ca0o7ytquw98n0/%E5%B9%BB%E7%81%AF%E7%89%87106.png
  [108]: http://static.zybuluo.com/shenyuflying/wiz9otduyj5hsysewexfnhxs/%E5%B9%BB%E7%81%AF%E7%89%87107.png
  [109]: http://static.zybuluo.com/shenyuflying/jbz0wy39z57vgmn5dfvlil1i/%E5%B9%BB%E7%81%AF%E7%89%87108.png
  [110]: http://static.zybuluo.com/shenyuflying/jaqx0o42vnufwcl1c84n7vo6/%E5%B9%BB%E7%81%AF%E7%89%87109.png
  [111]: http://static.zybuluo.com/shenyuflying/6u1qr6u3j1kbv62tkhxfn6yi/%E5%B9%BB%E7%81%AF%E7%89%87110.png
  [112]: http://static.zybuluo.com/shenyuflying/yzffpxdo6whue2b164werzsf/%E5%B9%BB%E7%81%AF%E7%89%87111.png
  [113]: http://static.zybuluo.com/shenyuflying/gjdtbswznp4tafizuwednof6/%E5%B9%BB%E7%81%AF%E7%89%87112.png
  [114]: http://static.zybuluo.com/shenyuflying/6o5w5xmlw2ed184li1iykudj/%E5%B9%BB%E7%81%AF%E7%89%87113.png
  [115]: http://static.zybuluo.com/shenyuflying/3apj0aygsidvimwpaj9cq7gc/%E5%B9%BB%E7%81%AF%E7%89%87114.png
  [116]: http://static.zybuluo.com/shenyuflying/21gyp0qj0vgrzbsq7436f422/%E5%B9%BB%E7%81%AF%E7%89%87115.png
  [117]: http://static.zybuluo.com/shenyuflying/abc10eorugjv7l4m5gq7kikh/%E5%B9%BB%E7%81%AF%E7%89%87116.png
  [118]: http://static.zybuluo.com/shenyuflying/iojsb394gtwieijx1dsekec3/%E5%B9%BB%E7%81%AF%E7%89%87117.png
  [119]: http://static.zybuluo.com/shenyuflying/x0togse9i38ayyd8c2yqrzim/%E5%B9%BB%E7%81%AF%E7%89%87118.png
  [120]: http://static.zybuluo.com/shenyuflying/ucdfglbmme8lg1j5ps3smzrk/%E5%B9%BB%E7%81%AF%E7%89%87119.png
  [121]: http://static.zybuluo.com/shenyuflying/odgupuxghwsf7qduvflamlc9/%E5%B9%BB%E7%81%AF%E7%89%87120.png
  [122]: http://static.zybuluo.com/shenyuflying/b77bq3stjazqdzhsrlv7304r/%E5%B9%BB%E7%81%AF%E7%89%87121.png
  [123]: http://static.zybuluo.com/shenyuflying/s8llwdix3qb8ob3hk1e13zne/%E5%B9%BB%E7%81%AF%E7%89%87122.png
  [124]: http://static.zybuluo.com/shenyuflying/5dyvc1gqi5dxjjchqskaxm8j/%E5%B9%BB%E7%81%AF%E7%89%87123.png
  [125]: http://static.zybuluo.com/shenyuflying/kb18vlepn7ju2u1fuia0cvup/%E5%B9%BB%E7%81%AF%E7%89%87124.png
  [126]: http://static.zybuluo.com/shenyuflying/oooaepc7tcjtnfoa0g2829sy/%E5%B9%BB%E7%81%AF%E7%89%87125.png
  [127]: http://static.zybuluo.com/shenyuflying/qy2a9n2l8khxs6dkpgv9vi5s/%E5%B9%BB%E7%81%AF%E7%89%87126.png
  [128]: http://static.zybuluo.com/shenyuflying/dtcraj9vyni70brtmkjf5dq2/%E5%B9%BB%E7%81%AF%E7%89%87127.png
  [129]: http://static.zybuluo.com/shenyuflying/75xuv344u4fa9lsi5z2fn38c/%E5%B9%BB%E7%81%AF%E7%89%87128.png
  [130]: http://static.zybuluo.com/shenyuflying/661bdfwt36u8vqdn1da1398h/%E5%B9%BB%E7%81%AF%E7%89%87129.png
  [131]: http://static.zybuluo.com/shenyuflying/3w802htroi2mommj7rdbvivz/%E5%B9%BB%E7%81%AF%E7%89%87130.png
  [132]: http://static.zybuluo.com/shenyuflying/305s04uipunyrsao4dimthkp/%E5%B9%BB%E7%81%AF%E7%89%87131.png
  [133]: http://static.zybuluo.com/shenyuflying/8u4j1sutgx9ic1smxebfxu45/%E5%B9%BB%E7%81%AF%E7%89%87132.png



如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
