`timescale 1ns/1ps

module conv1_tb;

    reg clk;
    reg rst_n;
    reg [7:0] data_in;

    wire [11:0] conv_out_1;
    wire [11:0] conv_out_2;
    wire [11:0] conv_out_3;
    wire valid_out_conv;

    // ??? 25?? ?? 16?? ??
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

    // ?? 10ns? ?? ??
    always #5 clk = ~clk;

integer i;

initial begin
    clk = 0;
    rst_n = 0;
    data_in = 0;

    repeat (2) @(negedge clk);
    rst_n = 1;

    // 원본 버퍼의 초기 인덱스가 넘어가는 시간
    @(negedge clk);

    // 28×28 이미지의 픽셀 784개 공급
    for (i = 0; i < 784; i = i + 1) begin
        data_in = 7 * (i / 28) + (i % 28);
        @(negedge clk);
    end

    data_in = 0;

    // 뒤쪽 계산을 관찰할 시간
    repeat (160) @(negedge clk);

    $finish;
end
endmodule
