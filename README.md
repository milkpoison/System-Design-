# System-Design-
module mat_mult_2x4_4x2 #(
    parameter DATA_WIDTH = 32,
    parameter SUM_WIDTH  = 32 + 3 + 1 // 32*32=64bit，累加4次，总和70bit，最后截断为32bit输出
)(
    input  logic                        clk,
    input  logic                        rst_n,
    input  logic                        start,
    // X[2][4] : 2 rows,4 columns, 32bit signed
    input  logic signed [DATA_WIDTH-1:0] X[0:1][0:3],
    // YT[4][2] : Y转置，4 rows,2 columns
    input  logic signed [DATA_WIDTH-1:0] YT[0:3][0:1],
    output logic                        done,
    // Result 2x2，4个32bit整数输出
    output logic signed [31:0]          res_out[0:3]
);

typedef enum logic [1:0] {
    IDLE,
    CALC,
    FINISH
} state_t;
state_t curr_state, next_state;

logic [1:0] i,j,k;
logic signed [SUM_WIDTH-1:0] acc;
logic signed [SUM_WIDTH-1:0] res_buf[0:1][0:1];

always_ff @(posedge clk or negedge rst_n) begin
    if(!rst_n)
        curr_state <= IDLE;
    else
        curr_state <= next_state;
end

always_comb begin
    next_state = curr_state;
    case(curr_state)
        IDLE:   if(start) next_state = CALC;
        CALC:   if(i == 1 && j == 1 && k == 3) next_state = FINISH;
        FINISH: next_state = IDLE;
    endcase
end

always_ff @(posedge clk or negedge rst_n) begin
    if(!rst_n) begin
        i <= 0; j <= 0; k <= 0;
        acc <= '0;
        res_buf <= '{default:'0};
        done <= 1'b0;
        res_out <= '{default:'0};
    end else begin
        case(curr_state)
            IDLE: begin
                done <= 1'b0;
                if(start) begin
                    i <= 0; j <= 0; k <= 0;
                    acc <= '0;
                end
            end

            CALC: begin
                acc <= acc + X[i][k] * YT[k][j];
                k <= k + 1'd1;

                if(k == 2'd3) begin
                    automatic logic signed [SUM_WIDTH-1:0] total = acc + X[i][k] * YT[k][j];
                    res_buf[i][j] <= total;
                    acc <= '0;
                    k <= 0;
                    j <= j + 1'd1;
                    if(j == 1'd1) begin
                        j <= 0;
                        i <= i + 1'd1;
                    end
                end
            end

            FINISH: begin
                done <= 1'b1;
                // 把2x2矩阵展平成4个输出：res[0][0], res[0][1], res[1][0], res[1][1]
                res_out[0] <= res_buf[0][0][31:0];
                res_out[1] <= res_buf[0][1][31:0];
                res_out[2] <= res_buf[1][0][31:0];
                res_out[3] <= res_buf[1][1][31:0];
            end
        endcase
    end
end

endmodule
