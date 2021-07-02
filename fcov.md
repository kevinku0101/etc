# brief introduction on functional coverage and assertion-based coverage 

## functional coverage

The SystemVerilog functional coverage constructs enable the following:

1. Coverage of variables and expressions, as well as cross coverage between them

2. Automatic as well as user-defined coverage bins

3. Associate bins with sets of values, transitions, or cross products

4. Filtering conditions at multiple levels

5. Events and sequences to automatically trigger coverage sampling

6. Procedural activation and query of coverage

7. Optional directives to control and regulate coverage

   

### covergroup

The covergroup construct encapsulates the specification of a coverage model. Each covergroup specification can include the following components:

1. A clocking event that synchronizes the sampling of coverage points
2. A set of coverage points
3. Cross coverage between coverage points

4. Optional formal arguments

5. Coverage options

A covergroup can be defined in a package, module,program, interface, checker, or class

### coverpoint

1. A covergroup can contain one or more coverage points. A coverage point specifies an integral expression
   that is to be covered.
2. Each coverage point includes a set of bins associated with the sampled values or value transitions of the covered expression.
3. The bins can be explicitly defined by the user or automatically created by SystemVerilog. 
4. Evaluation of the coverage point expression (and of its enabling iff condition, if any) takes place when the covergroup is
   sampled. 

### bins

The bins construct allows creating a separate bin for each value in the given range list or a single bin for the entire range of values

1. bins with specified size or not
2. bins with transitions 
3. wildcard bins
4. ignore_bins 
5. illegal_bins

### cross

 A coverage group can specify cross coverage between two or more coverage points or variables.

1. cross bins 
   - binsof
   - intersect
   - with 
   - match 
   - function inside cross 
2. ignore_bins
3. illegal_bins

## assertion-based coverage

Use the key word cover to build an assertion-based coverage with different types:

1. Immediate assertions, uses the keyword assert (not assert property), and is placed in procedural code and executed as a procedural statement.
2. Concurrent assertions, uses the keywords assert property, is placed outside of a procedural block and is executed once per sample cycle at the end of the cycle. The sample cycle is typically a posedge clk and sampling takes place at the end of the clock cycle, just before everything changes
   on the next posedge clk.

[sva reference](https://www.einfochips.com/blog/system-verilog-assertions-simplified/)

## connect with DUT

1. bind

   ```verilog
   bind bind_target_scope [: bind_target_instance_list] bind_instantiation ;
   | bind bind_target_instance bind_instantiation ;
   bind_target_scope ::=
   module_identifier
   | interface_identifier
   bind_target_instance ::=
   hierarchical_identifier constant_bit_select
   bind_target_instance_list ::=
   bind_target_instance { , bind_target_instance }
   bind_instantiation ::=
   program_instantiation
   | module_instantiation
   | interface_instantiation
   | checker_instantiation
   ```

2. placed in testbench files 

## a demo 

project in 

```sh
$PROJ_DIR/../mm_common_ip_verif/mm_axi_1_to_n
```

run with 

```sh
xrules -r $PROJ_DIR/mm_axi_1_to_n/rules/random/random.rules -d DUMP_FSDB -d ENABLE_COV
```

coverage part are in 

```sh
$PROJ_DIR/mm_common_ip_verif/mm_axi_1_to_n/top/demo_fcov_model.svh
```

```verilog
...
covergroup cg_rst_n @(posedge clk);
    cp_rst_n:coverpoint rst_n {
        bins trans01 = (0=>1);
        //bins trans10 = (1=>0);
    }
endgroup

function automatic bit odd_len_func(bit[W_BURST_LEN_WIDTH-1:0] n);
    return (n%2 == 1);
endfunction

covergroup cg_aw_m_chn @(posedge clk);
    cp_awid_m:coverpoint awid_m iff(rst_n === 1 && awvalid_m === 1){
        bins id0 = {0};
        bins id16 = {16};
        bins others[] = {[1:15],[17:$]};
    }
    cp_awaddr_m:coverpoint awaddr_m[W_ADDR_WIDTH-1:8] iff(rst_n === 1 && awvalid_m === 1){
        bins range[32] = {[0:$]};
        //bins range[1024] = {[0:$]};
    } 
    cp_awlen_m:coverpoint awlen_m iff(rst_n === 1 && awvalid_m === 1){
        bins even[] = {[0:15]} with (item%2==0);
        bins odd[] = {[0:15]} with (odd_len_func(item));
        //wildcard bins odd[] = {{{(W_BURST_LEN_WIDTH-1){1'b?}},1'b1}}; 
        //ignore_bins odd = {[0:15]} with (odd_len_func(item));
        illegal_bins illegal = default;
    }
    //x_awaddr_m_awid_m: cross cp_awid_m,cp_awaddr_m; 
    x_awaddr_m_awid_m: cross cp_awid_m,cp_awlen_m {
        bins x0 = binsof(cp_awid_m.id0) && binsof(cp_awlen_m) intersect{[0:7]};
        //ignore_bins others = binsof(cp_awid_m.others);
        ignore_bins ignore = binsof(cp_awid_m) intersect{[1:15],[17:$]};
    }
endgroup

covergroup cg_toggle(int nbits) with function sample(int bit_idx,bit bit_val);

    cp_bit_val :coverpoint bit_val {
        bins value[]  = {0,1};
        type_option.weight = 0;
    }
    cp_bit_idx :coverpoint bit_idx {
        bins idx[] = {[0:nbits-1]};
        type_option.weight = 0;
    }
    x_bit_idx_bit_val :cross cp_bit_idx,cp_bit_val;
    option.per_instance = 1;
endgroup

cg_rst_n u_cg_rst_n = new();
cg_aw_m_chn u_cg_aw_chn = new();

cg_toggle u_cg_awaddr_toggle = new(W_ADDR_WIDTH);
always @(posedge clk) begin
    if(rst_n === 1 && awvalid_m === 1)begin 
        foreach(awaddr_m[idx])begin 
            u_cg_awaddr_toggle.sample(idx,awaddr_m[idx]);
        end 
    end 
end 

always @(posedge clk) begin
    if(rst_n === 1 && awvalid_m === 1)begin 
        c_awaddr_m :cover(awaddr_m === 'h12340) $display("c_awaddr_m covered");
    end 
end 

property p_awvalid_awready_delay;
    @(posedge clk) disable iff(rst_n !== 1)
    awvalid_m |-> ##[10:100] awready_m;
endproperty

c_awvalid_awready_delay: cover property(p_awvalid_awready_delay);
...
```



### 

