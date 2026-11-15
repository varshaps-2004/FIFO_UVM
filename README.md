# FIFO_UVM
//dut.sv
module fifo #(parameter WIDTH = 8, DEPTH = 8) (
	input clk,
	input reset,
	input w_en,
	input r_en,
	input [WIDTH - 1 : 0] w_data,
	output reg [WIDTH-1:0]r_data,
	output full,
	output empty
	);

	reg [WIDTH-1:0] mem [DEPTH-1:0];
	reg [$clog2(DEPTH)-1:0]w_ptr,r_ptr;
	reg [3:0] fifo_count;

	//write data into the fifo
	always @(posedge clk or posedge reset) begin
		if(reset) begin
			for(int i = 0;i<DEPTH;i++) begin
				mem[i] <= 0;
			end

			w_ptr <= 3'd0;
			r_ptr <= 3'd0;
			fifo_count <= 4'd0;
			r_data <= 8'b0;
		end
		else begin
			//write data join
			if(w_en && !full && !r_en) begin
			       mem[w_ptr] <= w_data;
			       w_ptr <= w_ptr + 1;
			       fifo_count <= fifo_count + 1;
		       end
		       //read from fifo
			if(r_en && !empty && !w_en) begin
				r_data <= mem[r_ptr];
				r_ptr <= r_ptr + 1;
				fifo_count <= fifo_count - 1;
			end
		end
	end

	assign full = (DEPTH == fifo_count);
	assign empty = (fifo_count == 0);
	endmodule

//interface.sv
interface intf(input logic clk,reset);

	logic [7:0] w_data;
	logic w_en;
	logic r_en;
	logic [7:0] r_data;
	logic full;
	logic empty;
endinterface


//sequence_item.sv

class fifo_seq_item extends uvm_sequence_item;

	//declare the signals
	
	rand bit [7:0]w_data;
	rand bit w_en;
	rand bit r_en;
	bit [7:0]r_data;
	bit full;
	bit empty;


	//Fields Registration
	
	`uvm_object_utils_begin(fifo_seq_item)
	`uvm_field_int(w_data,UVM_DEFAULT)
	`uvm_field_int(w_en,UVM_DEFAULT)
	`uvm_field_int(r_en,UVM_DEFAULT)
	`uvm_field_int(r_data,UVM_DEFAULT)
	`uvm_field_int(full,UVM_DEFAULT)
	`uvm_field_int(empty,UVM_DEFAULT)
	`uvm_object_utils_end


	//Function new constructor
	
	function new(string name = "fifo_seq_item");
		super.new(name);
	endfunction

	constraint c{
		if(w_en)
			w_data inside{[10:200]};
		else
			w_data == 0;
		}
	endclass
  
	//sequence1.sv
  class fifo_seq extends uvm_sequence#(fifo_seq_item);
	`uvm_object_utils(fifo_seq);
	fifo_seq_item req[2];

	//function new constructor
	function new(string name = "fifo_seq");
		super.new(name);
	endfunction

	virtual task body();
		req[0] = fifo_seq_item::type_id::create("req[0]");
		req[1] = fifo_seq_item::type_id::create("req[1]");


		//testcase: alternative write and read
		repeat(8) begin

			`uvm_do_with(req[0],{w_en == 1; r_en == 0;});
			`uvm_do_with(req[1],{w_en == 0; r_en == 1;});
		end
	endtask
endclass
  //sequence2.sv
  class fifo_write_read_seq extends uvm_sequence#(fifo_seq_item);
	`uvm_object_utils(fifo_write_read_seq)
	fifo_seq_item req[2];


	function new(string name = "fifo_write_read_seq");
		super.new(name);
	endfunction

	virtual task body();
		req[0] = fifo_seq_item::type_id::create("req0");
		req[1] = fifo_seq_item::type_id::create("req1");


		//WRITE first
		repeat (8) begin
			`uvm_do_with(req[0],{w_en == 1; r_en == 0;});
		end
	//Read
	repeat (8) begin
	`uvm_do_with(req[1], {w_en == 0; r_en == 1;});
end
endtask
endclass

//sequencer.sv
class fifo_seqr extends uvm_sequencer#(fifo_seq_item);
	`uvm_component_utils(fifo_seqr)

	//function new constructor
	function new(string name = "fifo_seqr",uvm_component parent);
		super.new(name,parent);
	endfunction
endclass

//driver.sv
class fifo_driver extends uvm_driver#(fifo_seq_item);
	`uvm_component_utils(fifo_driver)

	virtual intf vif;
	fifo_seq_item sq1;

	function new(string name = "fifo_driver",uvm_component parent);
		super.new(name,parent);
	endfunction

	function void build_phase(uvm_phase phase);
		super.build_phase(phase);
		sq1 = fifo_seq_item::type_id::create("sq1");
		if(!uvm_config_db#(virtual intf)::get(null,"*","vif",vif))
			`uvm_fatal("[FATAL ERROR]","Failed to get interface signals")
	endfunction


	task run_phase(uvm_phase phase);
		super.run_phase(phase);
		forever
		begin
			@(posedge vif.clk)
			seq_item_port.get_next_item(sq1);
			vif.w_data <= sq1.w_data;
			vif.w_en <= sq1.w_en;
			vif.r_en <= sq1.r_en;
			seq_item_port.item_done();
			`uvm_info("[FROM DRIVER RUN PHASE]",$sformatf("w_data = %0d | w_en = %0d | r_en = %0d", vif.w_data,vif.w_en,vif.r_en),UVM_LOW)
		end
	endtask
  
//input_monitor.sv
class fifo_ip_monitor extends uvm_monitor;

	`uvm_component_utils(fifo_ip_monitor)
	uvm_analysis_port#(fifo_seq_item)port1;
	fifo_seq_item sq2;
	virtual intf vif;

	reg [7:0] mem [7:0];
	reg [$clog2(8)-1:0]w_ptr,r_ptr;
	reg [3:0] fifo_count;

	//function new constructor
	function new(string name1 = "fifo_ip_monitor",uvm_component parent);
		super.new(name1,parent);
	endfunction

	//build phase
	function void build_phase(uvm_phase phase);
		super.build_phase(phase);
		sq2 = fifo_seq_item::type_id::create("sq2");
		port1 = new("port1",this);
		if(!uvm_config_db#(virtual intf)::get(this,"*","vif",vif))
			`uvm_fatal("[FATAL ERROR]","Failed to get interface signal")
	endfunction

	//run phase
	task run_phase(uvm_phase phase);
		super.run_phase(phase);
		forever begin
			@(posedge vif.clk)
			sq2.w_data = vif.w_data;
			sq2.w_en = vif.w_en;
			sq2.r_en = vif.r_en;
			sq2.full = vif.full;
			sq2.empty = vif.empty;
			bfm();
			`uvm_info("[INPUT RUN PHASE]",$sformatf("w_data = %0d | w_en = %0d | r_en = %0d | r_data = %0d",sq2.w_data,sq2.w_en,sq2.r_en,sq2.r_data),UVM_LOW)
			port1.write(sq2);
		end
	endtask

	//bfm
	task bfm();
		if(vif.reset) begin
			for(int i = 0; i < 8; i++) begin
				mem[i] <= 0;
			end
			w_ptr <= 3'd0;
			r_ptr <= 3'd0;
			fifo_count <= 4'd0;
			sq2.r_data <= 8'b0;
		end
		else begin
			//write into the fifo
			if(sq2.w_en == 1 && sq2.full == 0 && sq2.r_en == 0) begin
				mem[w_ptr] <= sq2.w_data;
				w_ptr <= w_ptr + 1;
				fifo_count <= fifo_count + 1;
			end

			//read from the fifo
			if(sq2.r_en == 1 && sq2.w_en == 0) begin
				sq2.r_data <= mem[r_ptr];
				r_ptr <= r_ptr + 1;
				fifo_count <= fifo_count - 1;
			end
		end

		sq2.full = (fifo_count == 8);
		sq2.empty = (fifo_count == 0);
	endtask
endclass


//output_monitor.sv
class fifo_op_monitor extends uvm_monitor;

	`uvm_component_utils(fifo_op_monitor)
	uvm_analysis_port#(fifo_seq_item)port2;
	virtual intf vif;
	fifo_seq_item sq3;


	//function new constructor
	function new(string name = "fifo_op_monitor",uvm_component parent);
		super.new(name,parent);
	endfunction

	//build phase
	function void build_phase(uvm_phase phase);
		super.build_phase(phase);
		sq3 = fifo_seq_item::type_id::create("sq3");
		port2 = new("port2",this);
		if(!uvm_config_db#(virtual intf)::get(this,"*","vif",vif))
			`uvm_fatal("[FATAL ERROR]","failed to get interface signal")
	endfunction


	//task run phase
	task run_phase(uvm_phase phase);
		super.run_phase(phase);
		forever begin
			@(posedge vif.clk)
			sq3.w_data = vif.w_data;
			sq3.w_en = vif.w_en;
			sq3.r_en = vif.r_en;
			sq3.r_data = vif.r_data;
			sq3.full = vif.full;
			sq3.empty = vif.empty;
			`uvm_info("[OUTPUT MONITOR RUN_PHASE]", $sformatf("w_data = %0d | w_en = %0d | r_en = %0d | r_data = %0d", sq3.w_data, sq3.w_en, sq3.r_en, sq3.r_data), UVM_LOW)
			port2.write(sq3);
		end
	endtask
endclass


//scoreboard.sv
`uvm_analysis_imp_decl(_1)
`uvm_analysis_imp_decl(_2)

class fifo_sb extends uvm_scoreboard;

	`uvm_component_utils(fifo_sb)
	fifo_seq_item tr1,tr2;
	uvm_analysis_imp_1#(fifo_seq_item,fifo_sb) rec1;
	uvm_analysis_imp_2#(fifo_seq_item,fifo_sb) rec2;


	function new(string name = "fifo_sb",uvm_component parent);
		super.new(name,parent);
	endfunction

	function void build_phase(uvm_phase phase);
		super.build_phase(phase);
		tr1 = fifo_seq_item::type_id::create("tr1");
		tr2 = fifo_seq_item::type_id::create("tr2");
		rec1 = new("rec1",this);
		rec2 = new("rec2",this);
	endfunction

	virtual task write_1(fifo_seq_item s1);
		tr1 = s1;
		`uvm_info("[IPMN to SB]", $sformatf("w_data = %0d | w_en = %0d | r_en = %0d | r_data = %0d",tr1.w_data,tr1.w_en,tr1.r_en,tr1.r_data),UVM_LOW)
	endtask

	virtual task write_2(fifo_seq_item s2);
		tr2 = s2;
		`uvm_info("[OPMN to SB]", $sformatf("w_data = %0d | w_en = %0d | r_en = %0d | r_data = %0d",tr2.w_data,tr2.w_en,tr2.r_en,tr2.r_data),UVM_LOW)
		//checking both outputs
		if(tr1.r_data == tr2.r_data)
			`uvm_info("[TEST PASSED]","test pass",UVM_LOW)
		else
			`uvm_error("[TEST FAILED]","test failed")
	endtask
endclass

//active_agent.sv
class fifo_active extends uvm_agent;

	`uvm_component_utils(fifo_active);
	fifo_seqr seqr1;
	fifo_driver dr1;
	fifo_ip_monitor ip1;

	//function new constructor
	function new(string name="fifo_active",uvm_component parent);
		super.new(name,parent);
	endfunction

	//function build phase
	function void build_phase(uvm_phase phase);
		super.build_phase(phase);
		seqr1 = fifo_seqr::type_id::create("seqr1",this);
		dr1 = fifo_driver::type_id::create("dr1",this);
		ip1 = fifo_ip_monitor::type_id::create("ip1",this);
	endfunction

	function void connect_phase(uvm_phase phase);
		super.connect_phase(phase);
		dr1.seq_item_port.connect(seqr1.seq_item_export);
	endfunction
endclass


//passive_agent.sv
class fifo_passive extends uvm_agent;
	`uvm_component_utils(fifo_passive)
	fifo_op_monitor op1;

	//function new constructor
	function new(string name = "fifo_passive",uvm_component parent);
		super.new(name,parent);

	endfunction

	//function build phase
	function void build_phase(uvm_phase phase);
		super.build_phase(phase);
		op1 = fifo_op_monitor::type_id::create("op1",this);
	endfunction
endclass

//env.sv
class fifo_env extends uvm_env;
	`uvm_component_utils(fifo_env)
	fifo_active ag1;
	fifo_passive ag2;

	fifo_sb sb1;
	coverage c1;

	function new(string name1 = "fifo_env",uvm_component parent1);
		super.new(name1,parent1);
	endfunction

	function void build_phase(uvm_phase phase);
		super.build_phase(phase);
		ag1 = fifo_active::type_id::create("ag1",this);
		ag2 = fifo_passive::type_id::create("ag2",this);
		sb1 = fifo_sb::type_id::create("sb1",this);
		c1 = coverage::type_id::create("c1",this);
	endfunction


	function void connect_phase(uvm_phase phase);
		super.connect_phase(phase);
		ag1.ip1.port1.connect(sb1.rec1);
		ag2.op1.port2.connect(sb1.rec2);
		ag1.ip1.port1.connect(c1.cov_export);
	endfunction

	function void end_of_elaboration_phase(uvm_phase phase);
		super.end_of_elaboration_phase(phase);
		uvm_top.print_topology();
	endfunction
endclass

//test
class fifo_test extends uvm_test;
	`uvm_component_utils(fifo_test)
	fifo_env env;

	//write and read sequenc
	fifo_seq my_seq;
	fifo_write_read_seq my_seq1;

	function new(string name = "fifo_test",uvm_component parent);
		super.new(name,parent);
	endfunction

	function void build_phase(uvm_phase phase);
		super.build_phase(phase);
		env = fifo_env::type_id::create("env",this);
	endfunction

	task run_phase(uvm_phase phase);
		super.run_phase(phase);
		my_seq = fifo_seq::type_id::create("my_seq");
		my_seq1 = fifo_write_read_seq::type_id::create("my_seq");
		phase.raise_objection(this);
		my_seq1.start(env.ag1.seqr1); // write first after read
		#10;
		my_seq.start(env.ag1.seqr1); //write and read alternatively
		phase.drop_objection(this);
	endtask
endclass


//top.sv
`include "uvm_macros.svh"
import uvm_pkg::*;
`include "files.sv"

module top;

bit clk,reset;
intf vif(clk,reset);

always #5 clk = ~clk;

initial begin
	clk = 0;
	reset = 1;
	#10;
	reset = 0;
	#340 $finish;
end

fifo dut(.clk(vif.clk),.reset(vif.reset),.w_data(vif.w_data),.w_en(vif.w_en),.r_en(vif.r_en),.full(vif.full),.empty(vif.empty),.r_data(vif.r_data));

bind fifo check h1(.clk(vif.clk),.reset(vif.reset),.w_data(vif.w_data),.w_en(vif.w_en),.r_en(vif.r_en),.full(vif.full),.empty(vif.empty),.r_data(vif.r_data));

initial begin
	uvm_config_db#(virtual intf)::set(null,"*","vif",vif);
end

initial begin
	run_test("fifo_test");
end
endmodule

//coverage.sv
class coverage extends uvm_component;

	`uvm_component_utils(coverage)

	fifo_seq_item fsi;
	uvm_analysis_imp#(fifo_seq_item,coverage)cov_export;

	covergroup cg;
		w_data:coverpoint fsi.w_data{
			bins low = {[0:80]};
			bins middle = {[80:160]};
			bins high = {[160:255]};
			}
		w_en:coverpoint fsi.w_en;
		r_en:coverpoint fsi.r_en;
	endgroup

	function new(string name = "coverage",uvm_component parent);
		super.new(name,parent);
		cov_export = new("cov_export",this);
		cg = new();
	endfunction

	function void write(fifo_seq_item t1);
		fsi = t1;
		cg.sample();
	endfunction

	function void report_phase(uvm_phase phase);
		`uvm_info(get_type_name(), $sformatf("coverage = %0.02f%%",cg.get_coverage()),UVM_LOW)
	endfunction
endclass

//assertion.sv
module check (
	input logic clk,reset,
	input logic [7:0] w_data,
	input logic w_en,r_en,
	input logic [7:0] r_data,
	input logic full,empty
	);

	property p1;
		@(posedge clk) disable iff(reset)
		!$isunknown(w_data) && !$isunknown(r_data);
	endproperty

	a1:assert property (p1)
	else $error("Assertion p1 is failed: w_datais unknown after disable reset");

	property p2;
		@(posedge clk) disable iff(reset)
		w_en != r_en;
	endproperty

	a2:assert property (p2)
	else $error("Assertion p2 failed: w_en and r_en both high at the same time");
	endmodule

  //file.sv
  `include "fifo_sequence_item.sv"
`include "fifo_seq.sv"
`include "fifo_read_seq.sv"
`include "fifo_seqr.sv"
`include "fifo_driver.sv"
`include "fifo_ip_monitor.sv"
`include "fifo_active.sv"
`include "fifo_op_monitor.sv"
`include "fifo_passive.sv"
`include "fifo_scoreboard.sv"
`include "coverage.sv"
`include "fifo_env.sv"
`include "fifo_test.sv"
`include "fifo_interface.sv"
`include "assertion.sv"
`include "dut.sv"


//run.do

vlib work
vlog -sv fifo_top.sv
vsim -voptargs=+acc work.top
add wave -r sim:/top/*
run -all

  
