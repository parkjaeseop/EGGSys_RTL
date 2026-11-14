# week1 수정점
* conv1_calc와 conv2_calc가 설계 의도인 100MHz에서는 정상적으로 동작할 수 없는 타이밍 에러를 가지고 있음을 발견함.
* 가장 timing을 많이 잡아먹는 calc_out_1,2,3 의 곱셈과 덧셈이 이어진 부분을 pipeline 으로 정리하여 100MHz에서도 잘 동작할 수 있도록 타이밍을 맞춤.
* conv1_buf에서 140칸의 버퍼는 너무 크다는 경고가 떠서 shift register 방식으로 변경함.
## conv_calc에서 변경한 부분 (464 ~ 592 번째 줄)
```verilog
// Add Pipeline to match clk
reg [5:0]         valid_d;
 
reg signed [16:0]  r_prod_1 [0:24];
reg signed [16:0]  r_prod_2 [0:24];
reg signed [16:0]  r_prod_3 [0:24];

reg signed [17:0]  r_st_01 [0:12];
reg signed [17:0]  r_st_02 [0:12];
reg signed [17:0]  r_st_03 [0:12];
 
reg signed [18:0]  r_st_11 [0:6] ;
reg signed [18:0]  r_st_12 [0:6] ;
reg signed [18:0]  r_st_13 [0:6] ;
 
reg signed [19:0]  r_st_21 [0:3] ;
reg signed [19:0]  r_st_22 [0:3] ;
reg signed [19:0]  r_st_23 [0:3] ;
 
reg signed [20:0]  r_st_31 [0:1] ;
reg signed [20:0]  r_st_32 [0:1] ;
reg signed [20:0]  r_st_33 [0:1] ;
 
reg signed [21:0]  r_st_41       ;
reg signed [21:0]  r_st_42       ;
reg signed [21:0]  r_st_43       ;
 
 always @(posedge clk or negedge rst_n) begin
    if(~rst_n) begin
        valid_d <= 6'b0;
    end
    else begin
        valid_d <= {valid_d[4:0],valid_out_buf};
    end    
 end
 // for weight 1
 always @(posedge clk) begin
    // product
    for(i=0; i<=24; i=i+1) begin
        r_prod_1[i] <= exp_data[i] * weight_1[i];
    end
    // first stage
    for(i=0; i<=11; i=i+1) begin
        r_st_01[i] <= r_prod_1[2*i] + r_prod_1[2*i+1];
    end
    r_st_01[12] <= r_prod_1[24];
    // second stage
    for(i=0; i<=5; i=i+1) begin
        r_st_11[i] <= r_st_01[2*i] + r_st_01[2*i+1]; 
    end
    r_st_11[6] <= r_st_01[12];
    // third stage
    for(i=0; i<=2; i=i+1) begin
        r_st_21[i] <= r_st_11[2*i] + r_st_11[2*i+1]; 
    end
    r_st_21[3] <= r_st_11[6];
    // forth stage
    for(i=0; i<=1; i=i+1) begin
        r_st_31[i] <= r_st_21[2*i] + r_st_21[2*i+1]; 
    end
    // fifth stage
    r_st_41 <= r_st_31[0] + r_st_31[1];
 end
 // for weight_2
  always @(posedge clk) begin
    // product
    for(i=0; i<=24; i=i+1) begin
        r_prod_2[i] <= exp_data[i] * weight_2[i];
    end
    // first stage
    for(i=0; i<=11; i=i+1) begin
        r_st_02[i] <= r_prod_2[2*i] + r_prod_2[2*i+1];
    end
    r_st_02[12] <= r_prod_2[24];
    // second stage
    for(i=0; i<=5; i=i+1) begin
        r_st_12[i] <= r_st_02[2*i] + r_st_02[2*i+1]; 
    end
    r_st_12[6] <= r_st_02[12];
    // third stage
    for(i=0; i<=2; i=i+1) begin
        r_st_22[i] <= r_st_12[2*i] + r_st_12[2*i+1]; 
    end
    r_st_22[3] <= r_st_12[6];
    // forth stage
    for(i=0; i<=1; i=i+1) begin
        r_st_32[i] <= r_st_22[2*i] + r_st_22[2*i+1]; 
    end
    // fifth stage
    r_st_42 <= r_st_32[0] + r_st_32[1];
 end
 // for weight_3
  always @(posedge clk) begin
    // product
    for(i=0; i<=24; i=i+1) begin
        r_prod_3[i] <= exp_data[i] * weight_3[i];
    end
    // first stage
    for(i=0; i<=11; i=i+1) begin
        r_st_03[i] <= r_prod_3[2*i] + r_prod_3[2*i+1];
    end
    r_st_03[12] <= r_prod_3[24];
    // second stage
    for(i=0; i<=5; i=i+1) begin
        r_st_13[i] <= r_st_03[2*i] + r_st_03[2*i+1]; 
    end
    r_st_13[6] <= r_st_03[12];
    // third stage
    for(i=0; i<=2; i=i+1) begin
        r_st_23[i] <= r_st_13[2*i] + r_st_13[2*i+1]; 
    end
    r_st_23[3] <= r_st_13[6];
    // forth stage
    for(i=0; i<=1; i=i+1) begin
        r_st_33[i] <= r_st_23[2*i] + r_st_23[2*i+1]; 
    end
    // fifth stage
    r_st_43 <= r_st_33[0] + r_st_33[1];
 end
 
 assign calc_out_1 = r_st_41;
 assign calc_out_2 = r_st_42;
 assign calc_out_3 = r_st_43;

 assign conv_out_1 = calc_out_1[19:8] + exp_bias[0];
 assign conv_out_2 = calc_out_2[19:8] + exp_bias[1];
 assign conv_out_3 = calc_out_3[19:8] + exp_bias[2];

 assign valid_out_calc = valid_d[5];
```
## conv_buf에서 변경한 부분
```verilog
 module conv1_buf #(parameter WIDTH = 28, HEIGHT = 28, DATA_BITS = 8)(
   input clk,
   input rst_n,
   input valid_in,
   input [DATA_BITS - 1:0] data_in,
   output reg [DATA_BITS - 1:0] data_out_0, data_out_1, data_out_2, data_out_3, data_out_4,
   data_out_5, data_out_6, data_out_7, data_out_8, data_out_9,
   data_out_10, data_out_11, data_out_12, data_out_13, data_out_14,
   data_out_15, data_out_16, data_out_17, data_out_18, data_out_19,
   data_out_20, data_out_21, data_out_22, data_out_23, data_out_24,
   output reg valid_out_buf
 );

 localparam FILTER_SIZE = 5;

 reg [DATA_BITS - 1:0] buffer_1 [0:WIDTH - 1];
 reg [DATA_BITS - 1:0] buffer_2 [0:WIDTH - 1];
 reg [DATA_BITS - 1:0] buffer_3 [0:WIDTH - 1];
 reg [DATA_BITS - 1:0] buffer_4 [0:WIDTH - 1];
 
 reg [4:0] w_idx, h_idx;

 always @(posedge clk or negedge rst_n) begin
    if(~rst_n) begin
        w_idx <= 0;
        h_idx <= 0;
        valid_out_buf <= 0;
     end
     else begin
        if(valid_in) begin
            w_idx <= w_idx + 1;
            if(w_idx == WIDTH - 1) begin
                w_idx <= 0;
                if(h_idx == HEIGHT - 1) begin
                    h_idx <= 0;
                end
                else begin
                    h_idx <= h_idx + 1;
                end
            end
            // does window valid?
            if(w_idx >= FILTER_SIZE - 1 && h_idx >= FILTER_SIZE - 1) begin
                valid_out_buf <= 1'b1;                
            end
            else begin
                valid_out_buf <= 1'b0;
            end           
        end
        
        else begin
            valid_out_buf <= 1'b0;
        end
    end
 end
 
 integer j;
 
 always @(posedge clk) begin
    if(valid_in) begin
        buffer_1 [0] <= data_in;
        buffer_2 [0] <= buffer_1 [WIDTH - 1];
        buffer_3 [0] <= buffer_2 [WIDTH - 1];
        buffer_4 [0] <= buffer_3 [WIDTH - 1];
        
        for (j = 1; j < WIDTH; j = j+1) begin
            buffer_1 [j] <= buffer_1[j-1];
            buffer_2 [j] <= buffer_2[j-1];
            buffer_3 [j] <= buffer_3[j-1];
            buffer_4 [j] <= buffer_4[j-1];            
        end
        
        // first window
        data_out_0  <= data_out_1;
        data_out_1  <= data_out_2;
        data_out_2  <= data_out_3;  
        data_out_3  <= data_out_4;
        data_out_4  <= buffer_4[WIDTH-1];
        // second window
        data_out_5  <= data_out_6;  
        data_out_6  <= data_out_7;
        data_out_7  <= data_out_8;  
        data_out_8  <= data_out_9;
        data_out_9  <= buffer_3[WIDTH-1];
        // third window
        data_out_10 <= data_out_11; 
        data_out_11 <= data_out_12;
        data_out_12 <= data_out_13; 
        data_out_13 <= data_out_14;
        data_out_14 <= buffer_2[WIDTH-1];
        // forth window
        data_out_15 <= data_out_16; 
        data_out_16 <= data_out_17;
        data_out_17 <= data_out_18; 
        data_out_18 <= data_out_19;
        data_out_19 <= buffer_1[WIDTH-1];
        // fifth window
        data_out_20 <= data_out_21; 
        data_out_21 <= data_out_22;
        data_out_22 <= data_out_23; 
        data_out_23 <= data_out_24;
        data_out_24 <= data_in;
     end
  end
endmodule
```


## 변경한 코드 전체
```verilog
/*------------------------------------------------------------------------
 *
 *  Copyright (c) 2021 by Bo Young Kang, All rights reserved.
 *
 *  File name  : conv1_layer.v
 *  Written by : Kang, Bo Young
 *  Written on : Sep 30, 2021
 *  Version    : 21.2
 *  Design     : 1st Convolution Layer for CNN MNIST dataset
 *
 *------------------------------------------------------------------------*/

/*-------------------------------------------------------------------
 *  Module: conv1_layer
 *------------------------------------------------------------------*/
 
 module conv1_layer (
   input clk,
   input rst_n,
   input [7:0] data_in,
   output [11:0] conv_out_1, conv_out_2, conv_out_3,
   output valid_out_conv,
   input [0:199] w_11,
   input [0:199] w_12,
   input [0:199] w_13,
   input [0:23] b_1
 );

 wire [7:0] data_out_0, data_out_1, data_out_2, data_out_3, data_out_4,
  data_out_5, data_out_6, data_out_7, data_out_8, data_out_9,
  data_out_10, data_out_11, data_out_12, data_out_13, data_out_14,
  data_out_15, data_out_16, data_out_17, data_out_18, data_out_19,
  data_out_20, data_out_21, data_out_22, data_out_23, data_out_24;
 wire valid_out_buf;

 conv1_buf #(.WIDTH(28), .HEIGHT(28), .DATA_BITS(8)) conv1_buf(
   .clk(clk),
   .rst_n(rst_n),
   .data_in(data_in),
   .data_out_0(data_out_0),
   .data_out_1(data_out_1),
   .data_out_2(data_out_2),
   .data_out_3(data_out_3),
   .data_out_4(data_out_4),
   .data_out_5(data_out_5),
   .data_out_6(data_out_6),
   .data_out_7(data_out_7),
   .data_out_8(data_out_8),
   .data_out_9(data_out_9),
   .data_out_10(data_out_10),
   .data_out_11(data_out_11),
   .data_out_12(data_out_12),
   .data_out_13(data_out_13),
   .data_out_14(data_out_14),
   .data_out_15(data_out_15),
   .data_out_16(data_out_16),
   .data_out_17(data_out_17),
   .data_out_18(data_out_18),
   .data_out_19(data_out_19),
   .data_out_20(data_out_20),
   .data_out_21(data_out_21),
   .data_out_22(data_out_22),
   .data_out_23(data_out_23),
   .data_out_24(data_out_24),
   .valid_out_buf(valid_out_buf)
 );

 conv1_calc conv1_calc(
   .clk(clk),
   .rst_n(rst_n),
   .valid_out_buf(valid_out_buf),
   .data_out_0(data_out_0),
   .data_out_1(data_out_1),
   .data_out_2(data_out_2),
   .data_out_3(data_out_3),
   .data_out_4(data_out_4),
   .data_out_5(data_out_5),
   .data_out_6(data_out_6),
   .data_out_7(data_out_7),
   .data_out_8(data_out_8),
   .data_out_9(data_out_9),
   .data_out_10(data_out_10),
   .data_out_11(data_out_11),
   .data_out_12(data_out_12),
   .data_out_13(data_out_13),
   .data_out_14(data_out_14),
   .data_out_15(data_out_15),
   .data_out_16(data_out_16),
   .data_out_17(data_out_17),
   .data_out_18(data_out_18),
   .data_out_19(data_out_19),
   .data_out_20(data_out_20),
   .data_out_21(data_out_21),
   .data_out_22(data_out_22),
   .data_out_23(data_out_23),
   .data_out_24(data_out_24),
   .conv_out_1(conv_out_1),
   .conv_out_2(conv_out_2),
   .conv_out_3(conv_out_3),
   .valid_out_calc(valid_out_conv),
     .w_11(w_11),
  .w_12(w_12),
  .w_13(w_13),
  .b_1(b_1)
 );
 endmodule

/*------------------------------------------------------------------------
 *
 *  Copyright (c) 2021 by Bo Young Kang, All rights reserved.
 *
 *  File name  : conv1_buf.v
 *  Written by : Kang, Bo Young
 *  Written on : Sep 30, 2021
 *  Version    : 21.2
 *  Design     : 1st Convolution Layer for CNN MNIST dataset
 *               Input Buffer
 *
 *------------------------------------------------------------------------*/

/*-------------------------------------------------------------------
 *  Module: conv1_buf
 *------------------------------------------------------------------*/
 
 module conv1_buf #(parameter WIDTH = 28, HEIGHT = 28, DATA_BITS = 8)(
   input clk,
   input rst_n,
   input [DATA_BITS - 1:0] data_in,
   output reg [DATA_BITS - 1:0] data_out_0, data_out_1, data_out_2, data_out_3, data_out_4,
   data_out_5, data_out_6, data_out_7, data_out_8, data_out_9,
   data_out_10, data_out_11, data_out_12, data_out_13, data_out_14,
   data_out_15, data_out_16, data_out_17, data_out_18, data_out_19,
   data_out_20, data_out_21, data_out_22, data_out_23, data_out_24,
   output reg valid_out_buf
 );

 localparam FILTER_SIZE = 5;

 reg [DATA_BITS - 1:0] buffer [0:WIDTH * FILTER_SIZE - 1];
 reg [DATA_BITS - 1:0] buf_idx;
 reg [4:0] w_idx, h_idx;
 reg [2:0] buf_flag;  // 0 ~ 4
 reg state;

 always @(posedge clk) begin
   if(~rst_n) begin
     buf_idx <= -1;
     w_idx <= 0;
     h_idx <= 0;
     buf_flag <= 0;
     state <= 0;
     valid_out_buf <= 0;
     data_out_0 <= 12'bx;
     data_out_1 <= 12'bx;
     data_out_2 <= 12'bx;
     data_out_3 <= 12'bx;
     data_out_4 <= 12'bx;
     data_out_5 <= 12'bx;
     data_out_6 <= 12'bx;
     data_out_7 <= 12'bx;
     data_out_8 <= 12'bx;
     data_out_9 <= 12'bx;
     data_out_10 <= 12'bx;
     data_out_11 <= 12'bx;
     data_out_12 <= 12'bx;
     data_out_13 <= 12'bx;
     data_out_14 <= 12'bx;
     data_out_15 <= 12'bx;
     data_out_16 <= 12'bx;
     data_out_17 <= 12'bx;
     data_out_18 <= 12'bx;
     data_out_19 <= 12'bx;
     data_out_20 <= 12'bx;
     data_out_21 <= 12'bx;
     data_out_22 <= 12'bx;
     data_out_23 <= 12'bx;
     data_out_24 <= 12'bx;
   end else begin
   buf_idx <= buf_idx + 1;
   if(buf_idx == WIDTH * FILTER_SIZE - 1) begin // buffer size = 140 = 28(w) * 5(h)
     buf_idx <= 0;
   end
   
   buffer[buf_idx] <= data_in;  // data input
   
   // Wait until first 140 input data filled in buffer
   if(!state) begin
     if(buf_idx == WIDTH * FILTER_SIZE - 1) begin
       state <= 1'b1;
     end
   end else begin // valid state
     w_idx <= w_idx + 1'b1; // move right

     if(w_idx == WIDTH - FILTER_SIZE + 1) begin
       valid_out_buf <= 1'b0; // unvalid area
     end else if(w_idx == WIDTH - 1) begin
       buf_flag <= buf_flag + 1'b1;
       if(buf_flag == FILTER_SIZE - 1) begin
         buf_flag <= 0;
       end
       w_idx <= 0;

       if(h_idx == HEIGHT - FILTER_SIZE) begin  // done 1 input read -> 28 * 28
         h_idx <= 0;
         state <= 1'b0;
       end 
       
       h_idx <= h_idx + 1'b1;

     end else if(w_idx == 0) begin
       valid_out_buf <= 1'b1; // start valid area
     end

     // Buffer Selection -> 5 * 5
     if(buf_flag == 3'd0) begin
       data_out_0 <= buffer[w_idx];
       data_out_1 <= buffer[w_idx + 1];
       data_out_2 <= buffer[w_idx + 2];
       data_out_3 <= buffer[w_idx + 3];
       data_out_4 <= buffer[w_idx + 4];

       data_out_5 <= buffer[w_idx + WIDTH];
       data_out_6 <= buffer[w_idx + 1 + WIDTH];
       data_out_7 <= buffer[w_idx + 2 + WIDTH];
       data_out_8 <= buffer[w_idx + 3 + WIDTH];
       data_out_9 <= buffer[w_idx + 4 + WIDTH];

       data_out_10 <= buffer[w_idx + WIDTH * 2];
       data_out_11 <= buffer[w_idx + 1 + WIDTH * 2];
       data_out_12 <= buffer[w_idx + 2 + WIDTH * 2];
       data_out_13 <= buffer[w_idx + 3 + WIDTH * 2];
       data_out_14 <= buffer[w_idx + 4 + WIDTH * 2];

       data_out_15 <= buffer[w_idx + WIDTH * 3];
       data_out_16 <= buffer[w_idx + 1 + WIDTH * 3];
       data_out_17 <= buffer[w_idx + 2 + WIDTH * 3];
       data_out_18 <= buffer[w_idx + 3 + WIDTH * 3];
       data_out_19 <= buffer[w_idx + 4 + WIDTH * 3];

       data_out_20 <= buffer[w_idx + WIDTH * 4];
       data_out_21 <= buffer[w_idx + 1 + WIDTH * 4];
       data_out_22 <= buffer[w_idx + 2 + WIDTH * 4];
       data_out_23 <= buffer[w_idx + 3 + WIDTH * 4];
       data_out_24 <= buffer[w_idx + 4 + WIDTH * 4];
       
     end else if(buf_flag == 3'd1) begin
       data_out_0 <= buffer[w_idx + WIDTH];
       data_out_1 <= buffer[w_idx + 1 + WIDTH];
       data_out_2 <= buffer[w_idx + 2 + WIDTH];
       data_out_3 <= buffer[w_idx + 3 + WIDTH];
       data_out_4 <= buffer[w_idx + 4 + WIDTH];

       data_out_5 <= buffer[w_idx + WIDTH * 2];
       data_out_6 <= buffer[w_idx + 1 + WIDTH * 2];
       data_out_7 <= buffer[w_idx + 2 + WIDTH * 2];
       data_out_8 <= buffer[w_idx + 3 + WIDTH * 2];
       data_out_9 <= buffer[w_idx + 4 + WIDTH * 2];

       data_out_10 <= buffer[w_idx + WIDTH * 3];
       data_out_11 <= buffer[w_idx + 1 + WIDTH * 3];
       data_out_12 <= buffer[w_idx + 2 + WIDTH * 3];
       data_out_13 <= buffer[w_idx + 3 + WIDTH * 3];
       data_out_14 <= buffer[w_idx + 4 + WIDTH * 3];

       data_out_15 <= buffer[w_idx + WIDTH * 4];
       data_out_16 <= buffer[w_idx + 1 + WIDTH * 4];
       data_out_17 <= buffer[w_idx + 2 + WIDTH * 4];
       data_out_18 <= buffer[w_idx + 3 + WIDTH * 4];
       data_out_19 <= buffer[w_idx + 4 + WIDTH * 4];

       data_out_20 <= buffer[w_idx];
       data_out_21 <= buffer[w_idx + 1];
       data_out_22 <= buffer[w_idx + 2];
       data_out_23 <= buffer[w_idx + 3];
       data_out_24 <= buffer[w_idx + 4];
     end else if(buf_flag == 3'd2) begin
       data_out_0 <= buffer[w_idx + WIDTH * 2];
       data_out_1 <= buffer[w_idx + 1 + WIDTH * 2];
       data_out_2 <= buffer[w_idx + 2 + WIDTH * 2];
       data_out_3 <= buffer[w_idx + 3 + WIDTH * 2];
       data_out_4 <= buffer[w_idx + 4 + WIDTH * 2];

       data_out_5 <= buffer[w_idx + WIDTH * 3];
       data_out_6 <= buffer[w_idx + 1 + WIDTH * 3];
       data_out_7 <= buffer[w_idx + 2 + WIDTH * 3];
       data_out_8 <= buffer[w_idx + 3 + WIDTH * 3];
       data_out_9 <= buffer[w_idx + 4 + WIDTH * 3];

       data_out_10 <= buffer[w_idx + WIDTH * 4];
       data_out_11 <= buffer[w_idx + 1 + WIDTH * 4];
       data_out_12 <= buffer[w_idx + 2 + WIDTH * 4];
       data_out_13 <= buffer[w_idx + 3 + WIDTH * 4];
       data_out_14 <= buffer[w_idx + 4 + WIDTH * 4];

       data_out_15 <= buffer[w_idx];
       data_out_16 <= buffer[w_idx + 1];
       data_out_17 <= buffer[w_idx + 2];
       data_out_18 <= buffer[w_idx + 3];
       data_out_19 <= buffer[w_idx + 4];

       data_out_20 <= buffer[w_idx + WIDTH];
       data_out_21 <= buffer[w_idx + 1 + WIDTH];
       data_out_22 <= buffer[w_idx + 2 + WIDTH];
       data_out_23 <= buffer[w_idx + 3 + WIDTH];
       data_out_24 <= buffer[w_idx + 4 + WIDTH];
     end else if(buf_flag == 3'd3) begin
       data_out_0 <= buffer[w_idx + WIDTH * 3];
       data_out_1 <= buffer[w_idx + 1 + WIDTH * 3];
       data_out_2 <= buffer[w_idx + 2 + WIDTH * 3];
       data_out_3 <= buffer[w_idx + 3 + WIDTH * 3];
       data_out_4 <= buffer[w_idx + 4 + WIDTH * 3];

       data_out_5 <= buffer[w_idx + WIDTH * 4];
       data_out_6 <= buffer[w_idx + 1 + WIDTH * 4];
       data_out_7 <= buffer[w_idx + 2 + WIDTH * 4];
       data_out_8 <= buffer[w_idx + 3 + WIDTH * 4];
       data_out_9 <= buffer[w_idx + 4 + WIDTH * 4];

       data_out_10 <= buffer[w_idx];
       data_out_11 <= buffer[w_idx + 1];
       data_out_12 <= buffer[w_idx + 2];
       data_out_13 <= buffer[w_idx + 3];
       data_out_14 <= buffer[w_idx + 4];

       data_out_15 <= buffer[w_idx + WIDTH];
       data_out_16 <= buffer[w_idx + 1 + WIDTH];
       data_out_17 <= buffer[w_idx + 2 + WIDTH];
       data_out_18 <= buffer[w_idx + 3 + WIDTH];
       data_out_19 <= buffer[w_idx + 4 + WIDTH];

       data_out_20 <= buffer[w_idx + WIDTH * 2];
       data_out_21 <= buffer[w_idx + 1 + WIDTH * 2];
       data_out_22 <= buffer[w_idx + 2 + WIDTH * 2];
       data_out_23 <= buffer[w_idx + 3 + WIDTH * 2];
       data_out_24 <= buffer[w_idx + 4 + WIDTH * 2];      
     end else if(buf_flag == 3'd4) begin
       data_out_0 <= buffer[w_idx + WIDTH * 4];
       data_out_1 <= buffer[w_idx + 1 + WIDTH * 4];
       data_out_2 <= buffer[w_idx + 2 + WIDTH * 4];
       data_out_3 <= buffer[w_idx + 3 + WIDTH * 4];
       data_out_4 <= buffer[w_idx + 4 + WIDTH * 4];

       data_out_5 <= buffer[w_idx];
       data_out_6 <= buffer[w_idx + 1];
       data_out_7 <= buffer[w_idx + 2];
       data_out_8 <= buffer[w_idx + 3];
       data_out_9 <= buffer[w_idx + 4];

       data_out_10 <= buffer[w_idx + WIDTH];
       data_out_11 <= buffer[w_idx + 1 + WIDTH];
       data_out_12 <= buffer[w_idx + 2 + WIDTH];
       data_out_13 <= buffer[w_idx + 3 + WIDTH];
       data_out_14 <= buffer[w_idx + 4 + WIDTH];

       data_out_15 <= buffer[w_idx + WIDTH * 2];
       data_out_16 <= buffer[w_idx + 1 + WIDTH * 2];
       data_out_17 <= buffer[w_idx + 2 + WIDTH * 2];
       data_out_18 <= buffer[w_idx + 3 + WIDTH * 2];
       data_out_19 <= buffer[w_idx + 4 + WIDTH * 2];

       data_out_20 <= buffer[w_idx + WIDTH * 3];
       data_out_21 <= buffer[w_idx + 1 + WIDTH * 3];
       data_out_22 <= buffer[w_idx + 2 + WIDTH * 3];
       data_out_23 <= buffer[w_idx + 3 + WIDTH * 3];
       data_out_24 <= buffer[w_idx + 4 + WIDTH * 3];   
     end
   end
   end
 end
endmodule

/*------------------------------------------------------------------------
 *
 *  Copyright (c) 2021 by Bo Young Kang, All rights reserved.
 *
 *  File name  : conv1_calc.v
 *  Written by : Kang, Bo Young
 *  Written on : Oct 1, 2021
 *  Version    : 21.2
 *  Design     : 1st Convolution Layer for CNN MNIST dataset
 *               Convolution Sum Calculation
 *
 *------------------------------------------------------------------------*/

/*-------------------------------------------------------------------
 *  Module: conv1_calc
 *------------------------------------------------------------------*/
 
 module conv1_calc #(parameter WIDTH = 28, HEIGHT = 28, DATA_BITS = 8)(
   input clk,
   input rst_n,
   input valid_out_buf,
   input [DATA_BITS - 1:0] data_out_0, data_out_1, data_out_2, data_out_3, data_out_4,
   data_out_5, data_out_6, data_out_7, data_out_8, data_out_9,
   data_out_10, data_out_11, data_out_12, data_out_13, data_out_14,
   data_out_15, data_out_16, data_out_17, data_out_18, data_out_19,
   data_out_20, data_out_21, data_out_22, data_out_23, data_out_24,
   output signed [11:0] conv_out_1, conv_out_2, conv_out_3,
   output valid_out_calc,
   input [0:199] w_11,
   input [0:199] w_12,
   input [0:199] w_13,
   input [0:23] b_1
 );

 localparam FILTER_SIZE = 5;
 localparam CHANNEL_LEN = 3;



 wire signed [19:0] calc_out_1, calc_out_2, calc_out_3;
 wire signed [DATA_BITS:0] exp_data [0:FILTER_SIZE * FILTER_SIZE - 1];
 wire signed [11:0] exp_bias [0:CHANNEL_LEN - 1];
 
 reg signed [DATA_BITS - 1:0] weight_1 [0:FILTER_SIZE * FILTER_SIZE - 1];
 reg signed [DATA_BITS - 1:0] weight_2 [0:FILTER_SIZE * FILTER_SIZE - 1];
 reg signed [DATA_BITS - 1:0] weight_3 [0:FILTER_SIZE * FILTER_SIZE - 1];
 reg signed [DATA_BITS - 1:0] bias [0:CHANNEL_LEN - 1];
 
integer i;
always @(*) begin
    for(i=0;i<=24;i=i+1) begin
        weight_1[i]=w_11[(8*i)+:8];
        weight_2[i]=w_12[(8*i)+:8];
        weight_3[i]=w_13[(8*i)+:8];
    end
    for(i=0;i<=2;i=i+1) begin
        bias[i]=b_1[(8*i)+:8];
    end
end
 
 // Unsigned -> Signed
 assign exp_data[0] = {1'd0, data_out_0};
 assign exp_data[1] = {1'd0, data_out_1};
 assign exp_data[2] = {1'd0, data_out_2};
 assign exp_data[3] = {1'd0, data_out_3};
 assign exp_data[4] = {1'd0, data_out_4};
 assign exp_data[5] = {1'd0, data_out_5};
 assign exp_data[6] = {1'd0, data_out_6};
 assign exp_data[7] = {1'd0, data_out_7};
 assign exp_data[8] = {1'd0, data_out_8};
 assign exp_data[9] = {1'd0, data_out_9};
 assign exp_data[10] = {1'd0, data_out_10};
 assign exp_data[11] = {1'd0, data_out_11};
 assign exp_data[12] = {1'd0, data_out_12};
 assign exp_data[13] = {1'd0, data_out_13};
 assign exp_data[14] = {1'd0, data_out_14};
 assign exp_data[15] = {1'd0, data_out_15};
 assign exp_data[16] = {1'd0, data_out_16};
 assign exp_data[17] = {1'd0, data_out_17};
 assign exp_data[18] = {1'd0, data_out_18};
 assign exp_data[19] = {1'd0, data_out_19};
 assign exp_data[20] = {1'd0, data_out_20};
 assign exp_data[21] = {1'd0, data_out_21};
 assign exp_data[22] = {1'd0, data_out_22};
 assign exp_data[23] = {1'd0, data_out_23};
 assign exp_data[24] = {1'd0, data_out_24};

 //  Re-calibration of extracted weight data according to MSB
 assign exp_bias[0] = (bias[0][7] == 1) ? {4'b1111, bias[0]} : {4'd0, bias[0]};
 assign exp_bias[1] = (bias[1][7] == 1) ? {4'b1111, bias[1]} : {4'd0, bias[1]};
 assign exp_bias[2] = (bias[2][7] == 1) ? {4'b1111, bias[2]} : {4'd0, bias[2]};
 
 // Add Pipeline to match clk
reg [5:0]         valid_d;
 
reg signed [16:0]  r_prod_1 [0:24];
reg signed [16:0]  r_prod_2 [0:24];
reg signed [16:0]  r_prod_3 [0:24];

reg signed [17:0]  r_st_01 [0:12];
reg signed [17:0]  r_st_02 [0:12];
reg signed [17:0]  r_st_03 [0:12];
 
reg signed [18:0]  r_st_11 [0:6] ;
reg signed [18:0]  r_st_12 [0:6] ;
reg signed [18:0]  r_st_13 [0:6] ;
 
reg signed [19:0]  r_st_21 [0:3] ;
reg signed [19:0]  r_st_22 [0:3] ;
reg signed [19:0]  r_st_23 [0:3] ;
 
reg signed [20:0]  r_st_31 [0:1] ;
reg signed [20:0]  r_st_32 [0:1] ;
reg signed [20:0]  r_st_33 [0:1] ;
 
reg signed [21:0]  r_st_41       ;
reg signed [21:0]  r_st_42       ;
reg signed [21:0]  r_st_43       ;
 
 always @(posedge clk or negedge rst_n) begin
    if(~rst_n) begin
        valid_d <= 6'b0;
    end
    else begin
        valid_d <= {valid_d[4:0],valid_out_buf};
    end    
 end
 // for weight 1
 always @(posedge clk) begin
    // product
    for(i=0; i<=24; i=i+1) begin
        r_prod_1[i] <= exp_data[i] * weight_1[i];
    end
    // first stage
    for(i=0; i<=11; i=i+1) begin
        r_st_01[i] <= r_prod_1[2*i] + r_prod_1[2*i+1];
    end
    r_st_01[12] <= r_prod_1[24];
    // second stage
    for(i=0; i<=5; i=i+1) begin
        r_st_11[i] <= r_st_01[2*i] + r_st_01[2*i+1]; 
    end
    r_st_11[6] <= r_st_01[12];
    // third stage
    for(i=0; i<=2; i=i+1) begin
        r_st_21[i] <= r_st_11[2*i] + r_st_11[2*i+1]; 
    end
    r_st_21[3] <= r_st_11[6];
    // forth stage
    for(i=0; i<=1; i=i+1) begin
        r_st_31[i] <= r_st_21[2*i] + r_st_21[2*i+1]; 
    end
    // fifth stage
    r_st_41 <= r_st_31[0] + r_st_31[1];
 end
 // for weight_2
  always @(posedge clk) begin
    // product
    for(i=0; i<=24; i=i+1) begin
        r_prod_2[i] <= exp_data[i] * weight_2[i];
    end
    // first stage
    for(i=0; i<=11; i=i+1) begin
        r_st_02[i] <= r_prod_2[2*i] + r_prod_2[2*i+1];
    end
    r_st_02[12] <= r_prod_2[24];
    // second stage
    for(i=0; i<=5; i=i+1) begin
        r_st_12[i] <= r_st_02[2*i] + r_st_02[2*i+1]; 
    end
    r_st_12[6] <= r_st_02[12];
    // third stage
    for(i=0; i<=2; i=i+1) begin
        r_st_22[i] <= r_st_12[2*i] + r_st_12[2*i+1]; 
    end
    r_st_22[3] <= r_st_12[6];
    // forth stage
    for(i=0; i<=1; i=i+1) begin
        r_st_32[i] <= r_st_22[2*i] + r_st_22[2*i+1]; 
    end
    // fifth stage
    r_st_42 <= r_st_32[0] + r_st_32[1];
 end
 // for weight_3
  always @(posedge clk) begin
    // product
    for(i=0; i<=24; i=i+1) begin
        r_prod_3[i] <= exp_data[i] * weight_3[i];
    end
    // first stage
    for(i=0; i<=11; i=i+1) begin
        r_st_03[i] <= r_prod_3[2*i] + r_prod_3[2*i+1];
    end
    r_st_03[12] <= r_prod_3[24];
    // second stage
    for(i=0; i<=5; i=i+1) begin
        r_st_13[i] <= r_st_03[2*i] + r_st_03[2*i+1]; 
    end
    r_st_13[6] <= r_st_03[12];
    // third stage
    for(i=0; i<=2; i=i+1) begin
        r_st_23[i] <= r_st_13[2*i] + r_st_13[2*i+1]; 
    end
    r_st_23[3] <= r_st_13[6];
    // forth stage
    for(i=0; i<=1; i=i+1) begin
        r_st_33[i] <= r_st_23[2*i] + r_st_23[2*i+1]; 
    end
    // fifth stage
    r_st_43 <= r_st_33[0] + r_st_33[1];
 end
 
 assign calc_out_1 = r_st_41;
 assign calc_out_2 = r_st_42;
 assign calc_out_3 = r_st_43;

 assign conv_out_1 = calc_out_1[19:8] + exp_bias[0];
 assign conv_out_2 = calc_out_2[19:8] + exp_bias[1];
 assign conv_out_3 = calc_out_3[19:8] + exp_bias[2];

 assign valid_out_calc = valid_d[5];
 
endmodule
```
