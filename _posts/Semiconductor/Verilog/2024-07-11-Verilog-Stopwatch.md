---
layout: post
related_posts: 
title: "[FPGA] Stopwatch 설계 (VerilogHDL)"
date: 2024-07-11 11:00:00 +0900
categories:
  - semicon
  - verilog
---
* * *
* toc
{:toc}

vivado 2019.2와 basys3 FPGA board를 이용한 Stopwatch 설계를 진행하였다.

## 1. 설계 사양

설계 사양을 구체적으로 구상해 보았다.

- btn 총 4개를 이용하여 rst, clear, display_mode, run_mode를 설정한다.
- Reset 상태에서 00.00이 display되고, display_mode는 sec. under_sec로 설정된다.
- Run_mode에서 시간 증가, 정지 모드 두 가지 상태가 있으며 btnr에 의해 toggle 된다.
- Clr는 run_mode가 정지 상태일 때만 유효하며 btnu에 의해 display된 내용이 모두 clear 된다.
- Display_mode는 sec. under_sec / min. sec. 두 가지 상태가 있으며 btnd에 의해 toggle 된다.

usec, sec, min은 system clock 인 100MHz를 분주하여 사용하도록 한다.

![](https://blog.kakaocdn.net/dn/mRRRz/btsIpiSmBL3/V4oTOEkKwSDWKhaovmKCAk/img.jpg)
![](https://blog.kakaocdn.net/dn/bqLRix/btsIqP2dOCg/QmYf9K7wbq1hKK64UonEcK/img.jpg)

## 2. 설계 구조도

설계 사양을 반영하여 전체 구조도를 그려본 후 설계를 진행하였다.  
실제 설계 과정에서 지속적으로 수정하며 진행하였고, 최종 설계 구조도는 다음과 같다.

![](https://blog.kakaocdn.net/dn/88Qx7/btsIqdJgTtV/gzXenNKrW5UZyuRvoUGkKk/img.jpg)

  

## 3. 동작 설계

### 1. TOP
```verilog
module stop_watch_top(
    input btnr, // Run Mode Input
    input btnd, // Display Mode Input
    input btnl, // Reset
    input btnu, // clear
    input clk, // 100MHz Clock Input
//    input [1:0] sw;
    output reg [3:0] an,
    output reg dp, // dp
    output reg [6:0] seg, // g,~a
    output [15:0] led
    );

wire rst;
wire run_md; // btnr
wire disp_md; // btnd
wire clr_on; // btnu

wire convert;

wire pls_100hz, pls_sel;
wire [6:0] usec;
wire [5:0] sec;
wire [5:0] min;

wire [1:0] sel;
wire [3:0] bcd_a, bcd_b, bcd_c, bcd_d;
wire bcd_done;
reg [3:0] bcd_sel;

reg [5:0] h_val;
reg [6:0] l_val;

wire [6:0] seg_d;

reg pl0, pl1;

assign rst = ~btnl;

// State Control Module
state_ctl u_state_ctl
(
.rst     (rst    ),
.clk     (clk    ),
.btnr    (btnr   ),
.btnd    (btnd   ),
.btnu    (btnu   ),
// Output
.run_md  (run_md ),
.clr_on  (clr_on ),
.disp_md (disp_md),
.led     (led    )
);

// Control Signal Generation Module
ctl_sig_gen u_ctl_sig_gen
(
.rst       (rst        ),
.clk       (clk        ),
.run_md    (run_md     ),
.clr_on    (clr_on     ),
//.sw     (sw         ),
// Output
.pls_100hz (pls_100hz  )
//.sel    (sel        )
);

// Control Signal Generation Module
sel_sig_gen u_sel_sig_gen
(
.rst        (rst        ),
.clk        (clk        ),
//.sw         (sw         ),
.pls_100hz  (pls_100hz  ),
// Output
.convert    (convert    ),
.plso       (pls_sel    ),
.sel        (sel        )
);

// Stop Watch Counter
stop_watch_cnt u_stop_watch_cnt
(
.rst    (rst        ),
.clk    (clk        ),
.clr    (clr_on     ),
.plsi   (pls_100hz  ),
// Output
.usec   (usec       ),
.sec    (sec        ),
.min    (min        )
);

always@(negedge rst, posedge clk)
begin
    if (rst == 0)
        begin   h_val <= 0; l_val <= 0; end
    else
        if (disp_md == 0)
            begin   h_val <= sec; l_val <= usec; end
        else
            begin   h_val <= min; l_val <= sec;  end
end

// Hexa to BCD Conversion
hex2bcd_top u_hex2bcd_top
(
.rst    (rst       ),
.clk    (clk       ),
.start  (convert   ),
.h_val  (h_val     ),
.l_val  (l_val     ),
// Output
.done   (bcd_done  ),
.bcd_a  (bcd_a     ),
.bcd_b  (bcd_b     ),
.bcd_c  (bcd_c     ),
.bcd_d  (bcd_d     )
);

always@(negedge rst, posedge clk)
begin
    if (rst == 0)
        bcd_sel <= 0;
    else
        if      (sel == 0)  bcd_sel <= bcd_a;
        else if (sel == 1)  bcd_sel <= bcd_b;
        else if (sel == 2)  bcd_sel <= bcd_c;
        else                bcd_sel <= bcd_d;
end

hex2seg u_hex2seg
(
.din    (bcd_sel	),
.seg_d  (seg_d  	)
);

// dp display
always@(negedge rst, posedge clk)
begin
    if (rst == 0)
        begin
            pl0 <= 0;   pl1 <= 0;
            seg <= 7'h7f;    dp <= 1;    an <= 4'hf;
        end
    else
        begin
            pl0 <= pls_sel; pl1 <= pl0;
            if (pl0 & ~ pl1)
                begin
                    seg <= ~seg_d;
                    case(sel)
                    2'd0    : begin     an <= 4'b0111;  dp <= 1;        end
                    2'd1    : begin     an <= 4'b1011;  dp <= 0;        end
                    2'd2    : begin     an <= 4'b1101;  dp <= 1;        end
                    default : begin     an <= 4'b1110;  dp <= ~disp_md; end
                    endcase
                end
        end        
end

endmodule
```

### 2. State Control
```verilog
module state_ctl(
    input rst,
    input clk, // 100MHz Clock Input
    input btnr, // Run Mode Input
    input btnd, // Display Mode Input
    input btnu, // clear
    output reg run_md, disp_md, clr_on,
    output reg [15:0] led
    );
    
wire run_md_btn;  // btnr
wire disp_md_btn; // btnd
wire clr_btn;     // btnu

reg krn1; // run button shift
reg kdp1; // display button shift
reg kcl1; // clr button shift

//// state
//reg run_md_reg;
//reg disp_md_reg;
//reg clr_on;

debounce u_debounce_0
(
.rst    (rst        ),
.clk    (clk        ),
.btnr   (btnr       ),
.key    (run_md_btn )
);

debounce u_debounce_1
(
.rst    (rst        ),
.clk    (clk        ),
.btnr   (btnd       ),
.key    (disp_md_btn)
);

debounce u_debounce_2
(
.rst    (rst        ),
.clk    (clk        ),
.btnr   (btnu       ),
.key    (clr_btn    )
);

always@(negedge rst, posedge clk)
begin
    if (rst == 0)
        led <= 0;
    else
        begin
            led[0] <= run_md;  
            led[15] <= disp_md;  
            led[7] <= clr_on;  
            led[8] <= clr_on;  
        end
end    

always@(negedge rst, posedge clk)
begin
    if (rst == 0) 
        begin
            krn1 <=0;   kdp1 <=0;   kcl1 <=0;
            run_md <= 0;
            disp_md <= 0;
            clr_on <= 0;
        end
    else
        begin
            krn1 <= run_md_btn;
            kdp1 <= disp_md_btn;
            kcl1 <= clr_btn;
            if (run_md_btn & ~krn1)
                run_md <= ~run_md;
            if (disp_md_btn & ~kdp1)
                disp_md <= ~disp_md;
            if (clr_btn & ~kcl1 & ~run_md) // rising edge & stop mode
                clr_on <= ~clr_on;
            else if (~clr_btn & kcl1 & clr_on) // falling edge & clr button on
                clr_on <= ~clr_on;
        end
end


endmodule
```

- debounce  
      
    스위치 chattering 현상을 고려하여 debounce를 하는 코드를 작성하였다.    
    system clock인 100MHz를 1kHz로 분주한 후, counter를 이용해 30번 count해 안정구간에 진입했음을 확인한다.

```verilog
module debounce(
    input rst, clk,
    input btnr,
    output reg key
    );
    
reg [15:0] cnt; 
reg pls_1k0, pls_1k1;

reg btn0, btn1;
reg [4:0] btn_cnt; 

always@(negedge rst, posedge clk)
begin
    if (rst == 0) 
        begin
            btn_cnt <= 0;   btn0 <= 0;   btn1 <= 0; key <= 0;
        end
    else if (pls_1k0 & ~pls_1k1) // rising edge
        begin
            btn0 <= btnr;   btn1 <= btn0;
            if (btn0 ^ btn1)
                btn_cnt <= 0;
            else if (btn_cnt < 30)
                btn_cnt <= btn_cnt + 1;
            // 
            if (btn_cnt == 29)
                key <= btn1;
        end
end

always@(negedge rst, posedge clk)
begin
    if (rst == 0) 
        begin
            cnt <= 0;   pls_1k0 <= 0;   pls_1k1 <= 0;
        end
    else 
        begin
            pls_1k1 <= pls_1k0; 
            if (cnt < 49999) 
                cnt <= cnt + 1;
            else 
                begin
                    cnt <= 0;
                    pls_1k0 <= ~pls_1k0; // pls_1k0 toggle
                end
        end
end

endmodule
```

#### Simulation

- my_state_ctl.tcl

```
restart
add_force btnl {0 0ns} {1 1ps} {0 10ns}
add_force clk {0 0ns} {1 5ns} -repeat_every 10ns
add_force btnu 0
add_force btnd 0
add_force btnr 0

source D:/fpga/2024a/pjt_lab/pjt_lab.srcs/sources_1/new/disp_mode_ctl.tcl

add_force btnu 1
run 35us

add_force btnr 0
run 35us
add_force btnr 1
run 35us

add_force btnu 0
run 35us

add_force btnr 0
run 35us

add_force btnu 1
run 35us

add_force btnr 1
run 35us
add_force btnr 0
run 35us

add_force btnu 0
run 35us

source D:/fpga/2024a/pjt_lab/pjt_lab.srcs/sources_1/new/clr_ctl.tcl
```

- disp_mode_ctl.tcl

```
add_force btnd 0
run 35us
add_force btnd 1
run 35us
add_force btnd 0
run 35us

add_force btnd 1
run 35us
add_force btnd 0
run 35us
```

- clr_ctl.tcl

```
add_force btnu 0
run 35us
add_force btnu 1
run 35us
add_force btnu 0
run 35us
```

![state_ctl](https://blog.kakaocdn.net/dn/lpC5U/btsIpGejXfQ/jaQg1GQ1AbEu50eFZtbyJ0/img.png)

### 3. Mode & Select Signal
#### Mode Control Generator

```verilog
module ctl_sig_gen(
    input   rst,        // rst. Active Low Reset
    input   clk,        // 100MHz Clock Input
    input   run_md,     // run_md. Count Up at High
    input   clr_on,     // Clear counter at Rising Edge when RUN_MD is Low
    output pls_100hz    // 100Hz Pulse Output
    );
    
reg cl0,cl1;      // Clear counter at Rising Edge when RUN_MD is Low

reg [13:0] cnta;
reg [4:0]  cntb;

reg pl0,pl1;

assign pls_100hz = cntb[4];

always@(negedge rst, posedge clk)
begin
    if (rst == 0)
        begin
            cntb <= 0;  //pls_100hz <= 0;
        end
    else
        if ((run_md == 0) & (cl0 & ~cl1))
            begin
                cntb <= 0;  //pls_100hz <= 0;
            end
        else if (run_md == 1)
            begin
                if (pl1 & ~pl0)
                    cntb <= cntb + 1;
            end
end    

always@(negedge rst, posedge clk)
begin
    if (rst == 0)
        begin   cnta <= 0;  pl0 <= 0;   pl1 <= 0;   end       
    else
        begin
            pl1 <= pl0;
            if ((run_md == 0) & (cl0 & ~cl1))
                cnta <= 0;
            else if (run_md == 1)
                if (cnta < 15624)
                    cnta <= cnta + 1;
                else
                    begin
                        cnta <= 0;
                        pl0 <= ~pl0;
                    end
        end
end    

always@(negedge rst, posedge clk)
begin
    if (rst == 0)
        begin   cl1 <= 0;       cl0 <= 0;       end
    else
        begin   cl1 <= cl0;     cl0 <= clr_on;  end
end    
endmodule
```

#### Select Signal Generator

```verilog
module sel_sig_gen(
    input   rst,        // rst. Active Low Reset
    input   clk,        // 100MHz Clock Input
    input   [1:0] sw,   // SEL period selection
    input   pls_100hz,  // 100Hz Pulse Output
    output  reg convert,    // SEL Period Cycle Pulse Output
    output  reg pls_sel,    // SEL Period Cycle Pulse Output
    output  reg [1:0] sel   // Digit select for 7 Segment Display 
    );
    
reg cl0,cl1;      // Clear counter at Rising Edge when RUN_MD is Low

reg [13:0] cnta;
reg [4:0]  cntb;

reg pl0,pl1;

always@(negedge rst, posedge clk)
begin
    if (rst == 0)
        begin   sel <= 0;  pls_sel <= 0; end      
    else
        begin
            convert <= cntb[4];
            //
            case(sw)
            2'd0 :      begin   sel <= cntb[1:0];   pls_sel <= pl1;      end
            2'd1 :      begin   sel <= cntb[2:1];   pls_sel <= cntb[0];  end
            2'd2 :      begin   sel <= cntb[3:2];   pls_sel <= cntb[1];  end
            default :   begin   sel <= cntb[4:3];   pls_sel <= cntb[2];  end
            endcase
        end
end    

always@(negedge rst, posedge clk)
begin
    if (rst == 0)
        cntb <= 0;       
    else
        if (cl0 & ~cl1)
            cntb <= 0;
        else 
            if (pl1 & ~pl0)
                cntb <= cntb + 1;
end    

always@(negedge rst, posedge clk)
begin
    if (rst == 0)
        begin   cnta <= 0;  pl0 <= 0;   pl1 <= 0;   end       
    else
        begin
            pl1 <= pl0;
        	if (cl0 & ~cl1)
        		begin
             	   cnta <= 0;	pl0 <= 0;	pl1 <= 0;
            	end
            else 
                if (cnta < 15624)
                    cnta <= cnta + 1;
                else
                    begin
                        cnta <= 0;
                        pl0 <= ~pl0;
                    end
        end
end    

always@(negedge rst, posedge clk)
begin
    if (rst == 0)
        begin   cl1 <= 0;       cl0 <= 0;       end
    else
        begin   cl1 <= cl0;     cl0 <= ~pls_100hz;  end
end    
endmodule
```

#### Simulation
- my_ctl_sig_gen.tcl

```
restart

add_force rst {1 0ns} {0 1ps} {1 20ns}
add_force clk {0 0ns} {1 5ns} -repeat_every 10ns
add_force run_md 0
add_force clr_on 0
add_force sw -radix unsigned 3

run 5us

add_force run_md 1
run 5us

add_force clr_on 1
run 1us

add_force clr_on 0
run 5us

add_force run_md 0
run 5us

add_force clr_on 1
run 1us

add_force clr_on 0
run 5us

add_force run_md 1
run 5us

run 11ms
```

![[Pasted image 20240908143806.png]]

### 4. StopWatch Counter

```verilog
module stop_watch_cnt(
    input rst, clk, // 100MHz Clock Input
    input clr,
    input plsi, // 100Hz Pulse input. count up at falling edge of PLSI
    output [6:0] usec,
    output [5:0] sec,
    output [5:0] min
    );
    
wire pls_sec, pls_min, pls_hr;

// under sec counter (00~99)
pls_cnt_100 u_usec_cnt
(
.rst    (rst    ),
.clk    (clk    ),
.clr    (clr    ),
.plsi   (plsi   ),
.plso   (pls_sec),
.qout   (usec   )
);

// sec counter (00~59)
pls_cnt_60 u_sec_cnt
(
.rst    (rst    ),
.clk    (clk    ),
.clr    (clr    ),
.plsi   (pls_sec),
.plso   (pls_min),
.qout   (sec    )
);

// min counter (00~59)
pls_cnt_60 u_min_cnt
(
.rst    (rst    ),
.clk    (clk    ),
.clr    (clr    ),
.plsi   (pls_min),
.plso   (pls_hr ),
.qout   (min    )
);

endmodule
```

pulse counter 60, 100 모듈을 만든 후 이를 under sec counter, sec, min counter로 활용한다.
qout 으로 숫자 count 0~99(0~59),  plso 로 주기 100(60)의 1clock pulse 제공

- pls_cnt_60

```verilog
module pls_cnt_60(
    input rst,clk,
    input clr,plsi,
    output reg plso,
    output reg [5:0] qout
    );
    
reg cl0,cl1;
reg pl0,pl1;

always@(negedge rst, posedge clk)
begin
    if (rst == 0)
        begin
            cl0 <= 0;  cl1 <= 0;  pl0 <= 0;   pl1 <= 0;  
            plso <= 0;
            qout <= 0;       
        end
    else 
        begin
            cl0 <= clr;   cl1 <= cl0;
            pl0 <= plsi;  pl1 <= pl0;   // 1) reserve code
            if (cl0 & ~cl1)         // Rising Edge of CLR
                begin
                    pl0 <= 0;   pl1 <= 0;   // 2) reset pl0, pl1 0. ignore 1)
                    qout <= 0;  plso <= 0;
                end
            else if (pl1 & ~pl0)    // Falling Edge of PLSI
                begin
                    if (qout >= 59)
                        begin
                            qout <= 0;
                            plso <= 0;
                        end
                    else
                        begin
                            qout <= qout + 1;
                            if (qout < 29)
                                plso <= 0;
                            else
                                plso <= 1;
                        end
                end
        end
end
endmodule
```

- pls_cnt_100

```verilog
module pls_cnt_100(
    input rst,clk,
    input clr,plsi,
    output reg plso,
    output reg [6:0] qout
    );
    
reg cl0,cl1;
reg pl0,pl1;

always@(negedge rst, posedge clk)
begin
    if (rst == 0)
        begin
            cl0 <= 0;  cl1 <= 0;  pl0 <= 0;   pl1 <= 0;  
            plso <= 0;
            qout <= 0;       
        end
    else 
        begin
            cl0 <= clr;   cl1 <= cl0;
            pl0 <= plsi;  pl1 <= pl0;
            if (cl0 & ~cl1)         // Rising Edge of CLR
                begin
                    pl0 <= 0;   pl1 <= 0;
                    qout <= 0;  plso <= 0;
                end
            else if (pl1 & ~pl0)    // Falling Edge of PLSI
                begin
                    if (qout >= 99)
                        begin
                            qout <= 0;
                            plso <= 0;
                        end
                    else
                        begin
                            qout <= qout + 1;
                            if (qout < 49)
                                plso <= 0;
                            else
                                plso <= 1;
                        end
                end
        end
end
endmodule
```

#### Simulation

- my_stop_watch_cnt.tcl

```
restart
add_force rst {1 0ns} {0 1ps} {1 50ns}
add_force clk {0 0ns} {1 5ns} -repeat_every 10ns
add_force plsi {0 0ns} {1 500ns} -repeat_every 1000ns
add_force clr 0
run 10ms
```

![[Pasted image 20240908145126.png]]

### 5. HEX to BCD

