# EGGSys_RTL
KNU Eggsystem_2026_RTL_teamproject_CNN
`timescale 1ns/1ps

module conv1_tb;

    reg clk;
    reg rst_n;
    reg [7:0] data_in;

    wire [11:0] conv_out_1;
    wire [11:0] conv_out_2;
    wire [11:0] conv_out_3;
    wire valid_out_conv;

    // 가중치 25개를 각각 16으로 설정
    wire [0:199] weights;
    assign weights = {25{8'd16}};

    conv1_layer dut (
        .clk(clk),
        .rst_n(rst_n),
        .data_in(data_in),

        .conv_out_1(conv_out_1),
        .conv_out_2(conv_out_2),
        .conv_out_3(conv_out_3),
        .valid_out_conv(valid_out_conv),

        .w_11(weights),
        .w_12(weights),
        .w_13(weights),
        .b_1(24'd0)
    );

    // 주기 10ns인 클록 생성
    always #5 clk = ~clk;

    initial begin
        clk = 0;
        rst_n = 0;
        data_in = 8'd16;

        // 두 번의 클록 상승 동안 초기화
        repeat (2) @(negedge clk);
        rst_n = 1;

        // 같은 픽셀값을 계속 공급하며 동작 확인
        repeat (200) @(negedge clk);

        $finish;
    end

endmodule
