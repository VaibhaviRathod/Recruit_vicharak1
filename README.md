`timescale 1ns / 1ps

module cpu_19bit (
    input clk,
    input reset,
    output reg [18:0] pc
);
    
    reg [18:0] instruction_memory [0:511];
    reg [18:0] data_memory [0:1023]; 
    reg [18:0] registers [15:0];

    wire [4:0] opcode;
    wire [3:0] rd;
    wire [3:0] rs1;
    wire [3:0] rs2;
    wire [13:0] addr_imm; 

    reg [18:0] alu_in1, alu_in2;
    wire [18:0] alu_result;
    reg alu_zero;
    reg [7:0] sp;
    reg [18:0] stack_mem [0:255];
    reg reg_write_en;
    reg mem_write_en;
    reg mem_read_en;
    reg [18:0] write_back_data;
    reg [18:0] pc_next;

    wire [18:0] instruction = instruction_memory[pc];

    assign opcode = instruction[18:14];
    assign rd = instruction[13:10];
    assign rs1 = instruction[9:6];
    assign rs2 = instruction[5:2];
    assign addr_imm = instruction[13:0]; 


    localparam OPCODE_ADD = 5'b00000;
    localparam OPCODE_SUB = 5'b00001;
    localparam OPCODE_MUL = 5'b00010;
    localparam OPCODE_DIV = 5'b00011;
    localparam OPCODE_INC = 5'b00100;
    localparam OPCODE_DEC = 5'b00101;
    localparam OPCODE_AND = 5'b00110;
    localparam OPCODE_OR = 5'b00111;
    localparam OPCODE_XOR = 5'b01000;
    localparam OPCODE_NOT = 5'b01001;
    localparam OPCODE_JMP = 5'b01010;
    localparam OPCODE_BEQ = 5'b01011;
    localparam OPCODE_BNE = 5'b01100;
    localparam OPCODE_CALL = 5'b01101;
    localparam OPCODE_RET = 5'b01110;
    localparam OPCODE_LD = 5'b01111;
    localparam OPCODE_ST = 5'b10000;
    localparam OPCODE_FFT = 5'b10001;
    localparam OPCODE_ENC = 5'b10010;
    localparam OPCODE_DEC_CUSTOM = 5'b10011;

   
    always @(*) begin
        alu_zero = 1'b0;
        case (opcode)
            OPCODE_ADD: alu_result = registers[rs1] + registers[rs2];
            OPCODE_SUB: alu_result = registers[rs1] - registers[rs2];
            OPCODE_MUL: alu_result = registers[rs1] * registers[rs2];
            OPCODE_DIV: alu_result = (registers[rs2] != 0) ? registers[rs1] / registers[rs2] : 0;
            OPCODE_INC: alu_result = registers[rd] + 1;
            OPCODE_DEC: alu_result = registers[rd] - 1;
            OPCODE_AND: alu_result = registers[rs1] & registers[rs2];
            OPCODE_OR: alu_result = registers[rs1] | registers[rs2];
            OPCODE_XOR: alu_result = registers[rs1] ^ registers[rs2];
            OPCODE_NOT: alu_result = ~registers[rs1];
            default: alu_result = 0;
        endcase

        alu_zero = (alu_result == 0);
    end

    always @(posedge clk or posedge reset) begin
        if (reset) begin
            pc <= 0;
            sp <= 255; 
            reg_write_en <= 0;
            mem_write_en <= 0;
            mem_read_en <= 0;
           
            integer i;
            for(i=0; i<16; i=i+1)
                registers[i] <= 0;
            for(i=0; i<512; i=i+1)
                instruction_memory[i] <= 0;
            for(i=0; i<1024; i=i+1)
                data_memory[i] <= 0;
            for(i=0; i<256; i=i+1)
                stack_mem[i] <= 0;
        end else begin
            reg_write_en <= 0;
            mem_write_en <= 0;
            mem_read_en <= 0;
            write_back_data <= 0;

            case (opcode)

                OPCODE_ADD, OPCODE_SUB, OPCODE_MUL, OPCODE_DIV,
                OPCODE_AND, OPCODE_OR, OPCODE_XOR,
                OPCODE_NOT, OPCODE_INC, OPCODE_DEC: begin
                    reg_write_en <= 1;
                    write_back_data <= alu_result;
                    registers[rd] <= alu_result;
                    pc <= pc + 1;
                end

               
                OPCODE_JMP: begin
                    
                    pc <= addr_imm;
                end

                OPCODE_BEQ: begin

                    if (registers[rd] == registers[rs1])
                        pc <= addr_imm;
                    else
                        pc <= pc + 1;
                end

                OPCODE_BNE: begin

                    if (registers[rd] != registers[rs1])
                        pc <= addr_imm;
                    else
                        pc <= pc + 1;
                end

                OPCODE_CALL: begin
                    
                    if (sp > 0) begin
                        stack_mem[sp] <= pc + 1;
                        sp <= sp - 1;
                        pc <= addr_imm;
                    end else begin
                        
                        pc <= pc + 1;
                    end
                end

                OPCODE_RET: begin

                    if (sp < 255) begin
                        sp <= sp + 1;
                        pc <= stack_mem[sp + 1];
                    end else
                        pc <= pc + 1;  
                end


                OPCODE_LD: begin
                    
                    registers[rd] <= data_memory[addr_imm];
                    pc <= pc + 1;
                end

                OPCODE_ST: begin

                    data_memory[addr_imm] <= registers[rd];
                    pc <= pc + 1;
                end

                OPCODE_FFT: begin

                    data_memory[registers[rd]] <= data_memory[registers[rs1]];
                    pc <= pc + 1;
                end

                OPCODE_ENC: begin
                    
                    data_memory[registers[rd]] <= data_memory[registers[rs1]] ^ 19'h1FFFF;
                    pc <= pc + 1;
                end

                OPCODE_DEC_CUSTOM: begin

                    data_memory[registers[rd]] <= data_memory[registers[rs1]] ^ 19'h1FFFF; 
                    pc <= pc + 1;
                end

                default: begin
                    
                    pc <= pc + 1;
                end
            endcase
        end
    end

endmodule


