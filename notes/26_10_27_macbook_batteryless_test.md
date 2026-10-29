# No battery test

The battery on my 2014 MacBook pro is in bad shape. I'm curious if I can run it without the battery.

## 0. Dependencies

```
sudo apt install stress-ng linux-tools-common linux-tools-generic msr-tools lm-sensors
sudo modprobe msr
```

## 1. With battery tests

Use 2 terminals to test this before removing the battery

Terminal 1 (load the machine)
```bash
stress-ng --cpu 0 --timeout 120s
```


Terminal 2 (monitor)
```bash
sudo turbostat --interval 2 --quiet --show Core,Avg_MHz,Busy%,Bzy_MHz,PkgTmp,PkgWatt
```

What I noticed here was that temps quickly skyrocket, and the CPU throttles even with the battery.

At idle:
```
Core	Avg_MHz	Busy%	Bzy_MHz	PkgTmp	PkgWatt
-	    65	    3.41	1918	70	4.76
0	    57	    2.58	2230	70	4.77
0	    61	    3.14	1958
1	    65	    3.46	1883
1	    64	    3.53	1818
2	    71	    3.40	2081
2	    66	    3.60	1825
3	    49	    1.51	3266
3	    89	    6.08	1472
```
Stress test starts:
```
Core	Avg_MHz	Busy%	Bzy_MHz	PkgTmp	PkgWatt
-	    3442	99.83	3464	92	57.14
0	    3450	99.83	3464	92	57.46
0	    3450	99.83	3464
1	    3450	99.83	3464
1	    3450	99.83	3463
2	    3450	99.83	3464
2	    3450	99.83	3464
3	    3450	99.82	3464
3	    3450	99.83	3464
```

Then, CPU throttling starts. You can see that the temp is high, and the Busy% drops. Though the busy MHz is high, the Avg MHz drops for what I assume is temp-based throttling.
```
Core	Avg_MHz	Busy%	Bzy_MHz	PkgTmp	PkgWatt
-	    1982	63.60	3124	96	33.58
0	    1982	63.59	3124	96	33.58
0	    1981	63.57	3124
1	    1982	63.58	3124
1	    1981	63.57	3124
2	    1982	63.59	3124
2	    1982	63.58	3124
3	    1982	63.60	3124

```

## 2. With battery test after early troubleshooting

I removed the back and cleaned the fans + the rest of the computer. They were quite dusty. Still throttles.


## 3. Replace thermal paste

This was my first time replacing thermal paste, but it worked better than I expected!
I followed [this tutorial](https://www.youtube.com/watch?v=UHcYxtCppng) with TM30 from my neighborhood Walmart.

## 4. Rerun stress test
I reran the same stress test from before. 
Notably, CPU isn't throttled like it was before!

```
Core	Avg_MHz	Busy%	Bzy_MHz	PkgTmp	PkgWatt
-	    2401	99.77	2412	80	    36.97
0	    2401	99.78	2412	80	    36.98
0	    2401	99.78	2412
1	    2401	99.77	2412
1	    2401	99.78	2412
2	    2401	99.77	2412
2	    2401	99.77	2412
3	    2401	99.77	2412
3	    2401	99.77	2412
```

## 5. Disconnect the battery and boot

Follow steps [here](https://www.youtube.com/watch?v=0nVMJ3w-bm4). In my case, it kept spinning up the fans, starting the screen, then dying. I realized my charger is 45W, which may not be enough for this computer to run without battery. Ordering a new one...