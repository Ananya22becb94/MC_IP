# MC_IP
RTL Project based on MC16x2 specifications
PHASE 4 IMPLEMENTATION

Phase 4 integrates the two pseudo-channels of one memory channel:

PC0 scheduler ─┐
               ├──> shared row bus ──> DFI formatter ──> DFI row output
PC1 scheduler ─┘

PC0 scheduler ─┐
               ├──> shared column bus ─> DFI formatter ─> DFI column output
PC1 scheduler ─┘

PC0 write/read data lane ─┐
                          ├──> DFI formatter ──> 64-bit DFI data bus
PC1 write/read data lane ─┘

The important architectural rule is:

• PC0 and PC1 have independent schedulers.
• They share the channel row command bus.
• They share the channel column command bus.
• Row and column arbitration are independent.
• PC0/PC1 data lanes remain independent.

────────────────────

1.  mc_arbiter : selects one request

 mc_arbiter  is the basic reusable arbitration block.

It receives:

req_valid
req_legal
req_priority

For each request slot:

•  req_valid[i] = 1  means a request exists.
•  req_legal[i] = 1  means timing/resource checks allow it to issue.
•  req_priority[i] = 1  is used for urgent requests such as refresh.

The effective candidate set is:

eligible = req_valid & req_legal;

Only eligible requests participate.

Selection rules

The arbiter applies the Phase 4 arbitration policy:

1. Priority requests win first.
2. Otherwise, the request with the largest wait counter wins.
3. If wait counters tie, the lowest index wins.
4. The selected request produces a one-hot grant.

For example, for two pseudo-channels:

PC0 wait counter = 4
PC1 wait counter = 7

PC1 wins.

If both counters are equal:

PC0 wait counter = 5
PC1 wait counter = 5

PC0 wins because it has the lower index.

Wait-counter behavior

Every eligible request that is not accepted increments its counter:

eligible loser -> wait counter + 1

The counter saturates at its maximum value and does not wrap around.

When a selected request is accepted:

selected request + grant_ack -> wait counter = 0

If a request disappears:

req_valid = 0 -> wait counter = 0

This prevents starvation because a continuously waiting request eventually reaches the maximum counter value.

Outputs

The arbiter produces:

grant       // one-hot selected request
grant_idx   // binary selected index
grant_valid // indicates that a request was selected

For two PCs:

grant = 2'b01 -> PC0 selected
grant = 2'b10 -> PC1 selected
grant = 2'b00 -> no request selected

────────────────────

2.  mc_ch_arb : arbitrates PC0 and PC1 on the shared channel buses

 mc_ch_arb  instantiates two independent  mc_arbiter  blocks:

u_row_arbiter
u_col_arbiter

The row arbiter handles:

PC0 row candidate vs PC1 row candidate

The column arbiter handles:

PC0 column candidate vs PC1 column candidate

This is intentional. The specification allows both of these to happen in the same cycle:

PC0 row command + PC1 column command

The row bus and column bus are separate physical buses, so they do not contend with each other.

Row arbitration

Inputs include:

row_req_valid
row_req_legal
row_req_refresh
row_cmd

The row arbiter selects either PC0 or PC1 and produces:

row_grant
row_valid
row_pc_sel
row_cmd_out

Column arbitration

The column side behaves similarly:

col_req_valid
col_req_legal
col_req_refresh
col_cmd

Outputs are:

col_grant
col_valid
col_pc_sel
col_cmd_out

Example

Suppose:

PC0 has a legal ACT command
PC1 has a legal ACT command

The row arbiter chooses one:

row_grant = 2'b01
row_pc_sel = 0

or:

row_grant = 2'b10
row_pc_sel = 1

At the same time, the column arbiter may independently choose a command from the other PC:

row_grant = 2'b01   // PC0 row command
col_grant = 2'b10   // PC1 column command

That is legal and expected.

PC-select field

The selected command is associated with its pseudo-channel:

row_pc_sel = row_grant_idx[0];
col_pc_sel = col_grant_idx[0];

The output command is also updated with the selected PC bit:

row_cmd_out[ROW_CMD_WIDTH-1] = row_pc_sel;
col_cmd_out[COL_CMD_WIDTH-1] = col_pc_sel;

This allows the shared DFI bus to tell the memory PHY whether the command belongs to PC0 or PC1.

────────────────────

3.  mc_dfi_fmt : registers and formats the DFI outputs

 mc_dfi_fmt  is the DFI boundary stage.

It receives already-selected command fields:

row_valid
row_pc_sel
row_cmd
row_bg
row_ba
row_addr

and packs them into the DFI row format:

dfi_row = {
    row_pc_sel,
    row_cmd,
    row_bg,
    row_ba,
    row_addr
};

The resulting row format is:

dfi_row[21]    = PC select
dfi_row[20:18] = row command
dfi_row[17:16] = bank group
dfi_row[15:14] = bank
dfi_row[13:0]  = row address

The column bus is packed similarly:

dfi_col = {
    col_pc_sel,
    col_cmd,
    col_bg,
    col_ba,
    col_addr
};

The column format is:

dfi_col[11]    = PC select
dfi_col[10:9]  = column command
dfi_col[8:7]   = bank group
dfi_col[6:5]   = bank
dfi_col[4:0]   = column address

One-cycle registered boundary

The formatter uses:

always_ff @(posedge dfi_clk or negedge dfi_reset_n)

Therefore, the arbitration result is captured on a clock edge and appears at the DFI output after that edge.

Conceptually:

Cycle N:
  mc_ch_arb selects command

Clock edge:
  mc_dfi_fmt captures command

Cycle N+1:
  dfi_row_val / dfi_row or dfi_col_val / dfi_col are visible

This provides a clean registered interface toward the PHY.

Write data path

The formatter combines the two PC write lanes:

wrdata = {wrdata_pc[1], wrdata_pc[0]};

So:

dfi_wrdata[31:0]  = PC0 write data
dfi_wrdata[63:32] = PC1 write data

The write mask is combined in the same way:

dfi_wrdata_mask[3:0] = PC0 mask
dfi_wrdata_mask[7:4] = PC1 mask

The write enables remain independently controlled:

dfi_wrdata_en[0] = PC0 write enable
dfi_wrdata_en[1] = PC1 write enable

This means PC0 and PC1 can transfer write data concurrently even though only one row command and one column command can be issued per bus per cycle.

Read data path

The formatter receives the combined 64-bit read bus:

dfi_rddata

and splits it:

rddata_pc0 = dfi_rddata[31:0];
rddata_pc1 = dfi_rddata[63:32];

Read-valid indications also remain per pseudo-channel:

rddata_valid[0] = PC0 read data valid
rddata_valid[1] = PC1 read data valid

────────────────────

4.  mc_channel_integration : connects everything together

 mc_channel_integration  is the Phase 4 top-level wrapper for one channel.

Its responsibilities are:

1. Receive command candidates from PC0 and PC1.
2. Pack each candidate into a common command format.
3. Send candidates to  mc_ch_arb .
4. Decode the selected command.
5. Send the selected fields to  mc_dfi_fmt .
6. Combine write data lanes.
7. Split returned read data lanes.
8. Return issue/grant information to the PC schedulers.

────────────────────

5. Command packing inside  mc_channel_integration 

Each PC supplies separate command fields:

row_cmd[pc]
row_bg[pc]
row_ba[pc]
row_addr[pc]

The integration module packs them into the DFI-shaped row format:

row_cmd_packed[pc] = {
    1'b0,
    row_cmd[pc],
    row_bg[pc],
    row_ba[pc],
    row_addr[pc]
};

The initial PC-select bit is zero because the command has not yet been selected.

The same happens for column commands:

col_cmd_packed[pc] = {
    1'b0,
    col_cmd[pc],
    col_bg[pc],
    col_ba[pc],
    col_addr[pc]
};

After arbitration,  mc_ch_arb  sets the PC-select bit according to the winning PC.

────────────────────

6. Command selection flow

The complete row command flow is:

PC0 row fields ─┐
                ├─> row_cmd_packed ─> row arbiter
PC1 row fields ─┘                         |
                                          v
                                  row_cmd_selected
                                  row_pc_sel
                                  row_valid
                                          |
                                          v
                                      mc_dfi_fmt
                                          |
                                          v
                                      dfi_row

The column flow is identical and independent:

PC0 column fields ─┐
                   ├─> col_cmd_packed ─> column arbiter
PC1 column fields ─┘                           |
                                               v
                                      col_cmd_selected
                                      col_pc_sel
                                      col_valid
                                               |
                                               v
                                           mc_dfi_fmt
                                               |
                                               v
                                           dfi_col

The integration module extracts the fields from the selected packed command:

fmt_row_cmd  = row_cmd_selected[20:18];
fmt_row_bg   = row_cmd_selected[17:16];
fmt_row_ba   = row_cmd_selected[15:14];
fmt_row_addr = row_cmd_selected[13:0];

For columns:

fmt_col_cmd  = col_cmd_selected[10:9];
fmt_col_bg   = col_cmd_selected[8:7];
fmt_col_ba   = col_cmd_selected[6:5];
fmt_col_addr = col_cmd_selected[4:0];

Those decoded fields are then passed to  mc_dfi_fmt .

────────────────────

7. Grant and acceptance flow

The integration module connects:

.row_grant_ack({NUM_PC{1'b1}})
.col_grant_ack({NUM_PC{1'b1}})

For two PCs, this becomes:

row_grant_ack = 2'b11
col_grant_ack = 2'b11

This means that when  mc_ch_arb  selects a valid row or column command, the formatter is assumed to accept it immediately.

The wait counter is therefore reset for the selected PC on the next clock.

The issue outputs expose the selected PC to the upstream schedulers:

assign row_issue = row_grant;
assign col_issue = col_grant;

For example:

row_issue = 2'b01

means PC0's row command was accepted.

col_issue = 2'b10

means PC1's column command was accepted.

The per-PC scheduler can use these signals to:

• Remove the command from its queue.
• Update bank state.
• Load timing counters.
• Reset its own command-level wait state.
• Advance write/read tracking.

────────────────────

8. Example cycle

Assume the following candidates:

PC0:
  legal row ACT
  legal column RD

PC1:
  legal row PRE
  legal column WR

Assume both wait counters are equal, so PC0 wins ties.

Arbitration result

row_grant = 2'b01
col_grant = 2'b01

PC0 wins both buses.

The formatter receives:

row_valid  = 1
row_pc_sel = 0
col_valid  = 1
col_pc_sel = 0

After the formatter clock edge:

dfi_row_val = 1
dfi_row      = PC0 ACT command

dfi_col_val = 1
dfi_col      = PC0 RD command

On the next cycle, PC1's wait counters have increased because PC1 was legal but lost.

If PC1 has waited longer than PC0 in a later cycle, PC1 can win.

Another possible cycle is:

row_grant = 2'b01   // PC0 row
col_grant = 2'b10   // PC1 column

Then both commands issue concurrently:

dfi_row = PC0 row command
dfi_col = PC1 column command

This is one of the key Phase 4 requirements.

────────────────────

9. Reset behavior

 mc_arbiter  uses synchronous active-low reset:

if (!rst_n)

This clears all wait counters.

 mc_dfi_fmt  uses asynchronous active-low reset:

always_ff @(posedge dfi_clk or negedge dfi_reset_n)

This clears:

• DFI row valid and data.
• DFI column valid and data.
• Write data outputs.
• Read-data enables.
• Captured read data.
• Read-valid outputs.

During reset, the formatter outputs are driven to known zero values.

────────────────────

10. What Phase 4 does not do

The Phase 4 integration does not itself implement:

• AXI handling.
• Command queue storage.
• DRAM timing legality calculations.
• Bank-state tracking.
• Refresh counter generation.
• Write-data buffering.
• Read-data reordering.
• DFI low-power control.
• Full 16-channel replication.

Those functions belong to the upstream per-PC cores or later integration phases.

Phase 4 assumes each PC scheduler has already produced:

valid command
timing-legal indication
refresh indication
decoded address fields
write/read data information

Phase 4 only decides which ready command reaches the shared channel buses and formats it for the DFI interface.

────────────────────

Overall relationship

mc_arbiter
    Basic credit-counter selector
        |
        +--> used twice inside mc_ch_arb
                  |
                  +--> row bus arbitration
                  +--> column bus arbitration
                            |
                            v
                    mc_channel_integration
                            |
                            v
                       mc_dfi_fmt
                            |
                            v
                    Registered DFI outputs

In short:

•  mc_arbiter  decides which request wins.
•  mc_ch_arb  decides which pseudo-channel owns the shared row or column bus.
•  mc_channel_integration  connects the schedulers, arbiter, and formatter.
•  mc_dfi_fmt  converts the winning command and data into registered DFI signals.
