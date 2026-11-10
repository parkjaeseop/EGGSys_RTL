`timescale 1ns/1ps

module cnn_stage1_tb;

    reg clk, rst_n, valid_in;
    reg [7:0] data_in;

    reg [0:199] w_11, w_12, w_13;
    reg [0:23] b_1;

    wire signed [11:0] conv_out_1, conv_out_2, conv_out_3;
    wire valid_out_conv;

    wire [11:0] max_value_1, max_value_2, max_value_3;
    wire valid_out_2;

    integer conv_count, pool_count, errors, case_id, channel;
    reg [11:0] observed, expected;

    cnn_stage1 dut (
        .clk(clk),
        .rst_n(rst_n),
        .valid_in(valid_in),
        .data_in(data_in),

        .w_11(w_11),
        .w_12(w_12),
        .w_13(w_13),
        .b_1(b_1),

        .conv_out_1(conv_out_1),
        .conv_out_2(conv_out_2),
        .conv_out_3(conv_out_3),
        .valid_out_conv(valid_out_conv),

        .max_value_1(max_value_1),
        .max_value_2(max_value_2),
        .max_value_3(max_value_3),
        .valid_out_2(valid_out_2)
    );

    always #5 clk = ~clk;

    // Reference Conv calculation using image coordinates.
    function signed [11:0] reference_conv;
        input integer ch, row, col;

        integer r, c, total;
        reg signed [7:0] weight_value, bias_value;
        reg signed [19:0] total_20;

        begin
            total = 0;

            for (r = 0; r < 5; r = r + 1) begin
                for (c = 0; c < 5; c = c + 1) begin
                    case (ch)
                        0: weight_value =
                            w_11[8*(5*r+c) +: 8];

                        1: weight_value =
                            w_12[8*(5*r+c) +: 8];

                        2: weight_value =
                            w_13[8*(5*r+c) +: 8];
                    endcase

                    total = total
                        + (7*(row+r) + col+c) * weight_value;
                end
            end

            case (ch)
                0: bias_value = b_1[0:7];
                1: bias_value = b_1[8:15];
                2: bias_value = b_1[16:23];
            endcase

            total_20 = total;
            reference_conv = (total_20 >>> 8) + bias_value;
        end
    endfunction

    // Reference 2x2 MaxPool + ReLU.
    function [11:0] reference_pool;
        input integer ch, row, col;

        integer r, c;
        reg signed [11:0] candidate, largest;

        begin
            largest = 0;

            for (r = 0; r < 2; r = r + 1) begin
                for (c = 0; c < 2; c = c + 1) begin
                    candidate =
                        reference_conv(ch, 2*row+r, 2*col+c);

                    if (candidate > largest)
                        largest = candidate;
                end
            end

            reference_pool = largest;
        end
    endfunction

    // Check outputs after DUT registers have settled.
    always @(negedge clk) begin
        if (!rst_n) begin
            conv_count = 0;
            pool_count = 0;

        end else begin

            if (valid_out_conv) begin
                for (channel = 0; channel < 3;
                     channel = channel + 1) begin

                    case (channel)
                        0: observed = conv_out_1;
                        1: observed = conv_out_2;
                        2: observed = conv_out_3;
                    endcase

                    expected = reference_conv(
                        channel, conv_count/24, conv_count%24
                    );

                    if (observed !== expected) begin
                        errors = errors + 1;

                        $display(
                            "FAIL Conv case=%0d ch=%0d index=%0d got=%0d expected=%0d",
                            case_id, channel+1, conv_count,
                            $signed(observed), $signed(expected)
                        );
                    end
                end

                conv_count = conv_count + 1;
            end

            if (valid_out_2) begin
                for (channel = 0; channel < 3;
                     channel = channel + 1) begin

                    case (channel)
                        0: observed = max_value_1;
                        1: observed = max_value_2;
                        2: observed = max_value_3;
                    endcase

                    expected = reference_pool(
                        channel, pool_count/12, pool_count%12
                    );

                    if (observed !== expected) begin
                        errors = errors + 1;

                        $display(
                            "FAIL Pool case=%0d ch=%0d index=%0d got=%0d expected=%0d",
                            case_id, channel+1, pool_count,
                            observed, expected
                        );
                    end
                end

                if (pool_count < 4)
                    $display(
                        "CASE %0d Pool[%0d] = %0d, %0d, %0d",
                        case_id, pool_count,
                        max_value_1, max_value_2, max_value_3
                    );

                pool_count = pool_count + 1;
            end
        end
    end

    task run_case;
        input integer test_number;

        integer pixel_index, weight_index, errors_before;

        begin
            @(negedge clk);

            rst_n = 0;
            valid_in = 0;
            case_id = test_number;
            errors_before = errors;

            w_11 = {25{8'd16}};
            w_12 = {25{8'd16}};
            w_13 = {25{8'd16}};
            b_1 = 0;

            // CASE 2: negative channel and input pauses.
            if (test_number == 2) begin
                w_12 = {25{8'hf0}};
                w_13 = 0;
                b_1[16:23] = 8'd8;
            end

            // CASE 3: different weights at different kernel positions.
            if (test_number == 3) begin
                for (weight_index = 0; weight_index < 25;
                     weight_index = weight_index + 1) begin

                    w_11[8*weight_index +: 8] =
                        weight_index - 12;

                    w_12[8*weight_index +: 8] =
                        (weight_index % 5 == 2) ? 8'd64 : 8'd0;

                    w_13[8*weight_index +: 8] =
                        (weight_index == 0) ? 8'h80 :
                        ((weight_index == 24) ? 8'd127 : 8'd0);
                end

                b_1[0:7] = 8'hfd;
                b_1[8:15] = 8'd5;
                b_1[16:23] = 8'hf9;
            end

            repeat (2) @(negedge clk);
            rst_n = 1;

            for (pixel_index = 0; pixel_index < 784;
                 pixel_index = pixel_index + 1) begin

                if (test_number == 2 && pixel_index % 9 == 4) begin
                    valid_in = 0;
                    @(negedge clk);
                end

                valid_in = 1;

                data_in =
                    7*(pixel_index/28) + pixel_index%28;

                @(negedge clk);
            end

            valid_in = 0;
            data_in = 0;

            repeat (10) @(negedge clk);

            if (conv_count != 576 || pool_count != 144) begin
                errors = errors + 1;

                $display(
                    "FAIL count: Conv=%0d Pool=%0d",
                    conv_count, pool_count
                );
            end

            if (errors == errors_before)
                $display(
                    "CASE %0d PASS: Conv=576 Pool=144",
                    test_number
                );
        end
    endtask

    initial begin
        clk = 0;
        rst_n = 0;
        valid_in = 0;
        data_in = 0;

        w_11 = 0;
        w_12 = 0;
        w_13 = 0;
        b_1 = 0;

        errors = 0;
        conv_count = 0;
        pool_count = 0;
        case_id = 0;

        run_case(1);
        run_case(2);
        run_case(3);

        if (errors == 0)
            $display("ALL TESTS PASS");
        else
            $display("TESTS FAILED: %0d errors", errors);

        $finish;
    end

    initial begin
        #30000;
        $display("FAIL: simulation timeout");
        $finish;
    end

endmodule
