---
title: "分支预测器研究（四）：RISC-V BOOM 分支预测概览"
date: 2025-11-27 15:03:13
tags:
  - "branch prediction"
  - "RISC-V BOOM"
  - "microarchitecture"
categories:
  - "Computer Architecture"
---

## Branch Prediction

BOOM uses two levels of branch prediction:
- a fast Next-Line Predictor (NLP)
- a slower but more complex Backing Predictor (BPD)

In this case, the NLP is a Branch Target Buffer and the BPD is a more complicated structure like a GShare predictor.

Branch Prediction:

- The Next-Line Predictor (NLP)
- - NLP Predictions
- - NLP Updates
- The Backing Predictor (BPD)
- - Making Predictions
- - Jump and Jump-Register Instructions
- - Updating the Backing Predictor
- - Managing the Global History Register (GHR)
- - The Fetch Target Queue (FTQ) for Predictions
- - Rename Snapshot State
- - The Abstract Branch Predictor Class
- - The Two-bit Counter Tables
- - The GShare Predictor
- - The TAGE Predictor
- - Other Predictors

It seams that BPD is much more complicated than NLP.

## The Next-Line Predictor (NLP)

BOOM core’s Front-end fetches instructions and predicts every cycle where to fetch the next instructions. If a misprediction is detected in
- BOOM’s Back-end
- BOOM’s own Backing Predictor (BPD) wants to redirect the pipeline in a different direction

, a request is sent to the Front-end and it begins fetching along a new instruction path.

The Next-Line Predictor (NLP) takes in the current PC being used to fetch instructions (the **Fetch PC**) and predicts combinationally where the next instructions should be fetched for the next cycle. If predicted correctly, there are no pipeline bubbles.

The NLP is an amalgamation of a fully-associative:
- Branch Target Buffer (BTB)
- Bi-Modal Table (BIM)
- Return Address Stack (RAS)

which work together to make a fast, but reasonably accurate prediction.

## NLP Predictions

The **Fetch PC** first performs a tag match to find a uniquely matching BTB entry. If a hit occurs, the BTB entry will make a prediction in concert with the RAS as to whether there is a branch, jump, or return found in the **Fetch Packet** and which instruction in the **Fetch Packet** is to blame. The BIM is used to determine if that prediction made was a branch taken or not taken. The BTB entry also contains a predicted PC target, which is used as the **Fetch PC** on the next cycle.

Fetch Packet
- A bundle returned by the Front-end which contains some set of consecutive instructions with a mask denoting which instructions are valid, amongst other meta-data related to instruction fetch and branch prediction. The **Fetch PC** will point to the first valid instruction in the **Fetch Packet**, as it is the PC used by the Front End to fetch the **Fetch Packet**.

Fetch PC
- The PC corresponding to the head of a **Fetch Packet** instruction group.



The Fetch PC scans the BTB’s “PC tags” for a match. If a match is found (and the entry is valid), the Bi-Modal Table (BIM) and RAS are consulted for the final verdict. If the entry is a “ret” (return instruction), then the target comes from the RAS. If the entry is a unconditional “jmp” (jump instruction), then the BIM is not consulted. The “bidx”, or branch index, marks which instruction in a superscalar Fetch Packet is the cause of the control flow prediction. This is necessary to mask off the other instructions in the Fetch Packet that come over the taken branch.

The hysteresis bits in the BIM are only used on a
- BTB entry hit
- the predicting instruction is a branch.

If the BTB entry contains **a return instruction**, the RAS stack is used to provide the predicted return PC as the next Fetch PC. The actual RAS management (of when to or the stack) is governed externally.

For area-efficiency, the high-order bits of the PC tags and PC targets are stored in a compressed file.

## NLP Updates
Each branch passed down the pipeline remembers not only its own PC, but also its Fetch PC (the PC of the head instruction of its Fetch Packet ).

### BTB Updates
The BTB is updated only when the Front-end is redirected to take a branch or jump by either the Branch Unit (in the Execute stage) or the BPD (later in the Fetch stages).

If there is no BTB entry corresponding to the taken branch or jump, an new entry is allocated for it.

### RAS Updates
The RAS is updated during the Fetch stages once the instructions in the Fetch Packet have been decoded. If the taken instruction is a call, the return address is pushed onto the RAS. If the taken instruction is a return, then the RAS is popped.

### Superscalar Predictions
When the NLP makes a prediction, it is actually using the BTB to tag match against the predicted branch’s Fetch PC, and not the PC of the branch itself. The NLP must predict across the entire Fetch Packet which of the many possible branches will be the dominating branch that redirects the PC. For this reason, we use a given branch’s Fetch PC rather than its own PC in the BTB tag match.


Each BTB entry corresponds to a single Fetch PC, but it is helping to predict across an entire Fetch Packet. However, the BTB entry can only store meta-data and target-data on a single control-flow instruction. While there are certainly pathological cases that can harm performance with this design, the assumption is that there is a correlation between which branch in a Fetch Packet is the dominating branch relative to the Fetch PC, and - at least for narrow fetch designs - evaluations of this design has shown it is very complexity-friendly with no noticeable loss in performance. Some other designs instead choose to provide a whole bank of BTBs for each possible instruction in the Fetch Packet.


## The Backing Predictor (BPD)

When the Next-Line Predictor (NLP) is predicting well, the processor’s Back-end is provided an unbroken stream of instructions to execute. The NLP is able to provide fast, single-cycle predictions by being expensive (in terms of both area and power), very small (only a few dozen branches can be remembered), and very simple (the Bi-Modal Table (BIM) hysteresis bits are not able to learn very complicated or long history patterns).

To capture more branches and more complicated branching behaviors, BOOM provides support for a Backing Predictor (BPD).

The BPD ‘s goal is to provide very high accuracy in a (hopefully) dense area. The BPD only makes taken/not-taken predictions; it therefore relies on some other agent to provide information on what instructions are branches and what their targets are. The BPD can either use the BTB for this information or it can wait and decode the instructions themselves once they have been fetched from the i-cache. This saves on needing to store the PC tags and branch targets within the BPD.

The BPD is accessed throughout the Fetch stages and in parallel with the instruction cache access and BTB. This allows the BPD to be stored in sequential memory (i.e., SRAM instead of flip-flops). With some clever architecting, the BPD can be stored in single-ported SRAM to achieve the density desired.


## Making Predictions
When making a prediction, the BPD must provide the following:
- is a prediction being made?
- a bit-vector of taken/not-taken predictions

As per the first bullet-point, the BPD may decide to not make a prediction. This may be because the predictor uses tags to inform whether its prediction is valid or there may be a structural hazard that prevented a prediction from being made.

The BPD provides a bit-vector of taken/not-taken predictions, the size of the bit-vector matching the Fetch Width of the pipeline (one bit for each instruction in the Fetch Packet ). A later Fetch stage will will decode the instructions in the Fetch Packet , compute the branch targets, and decide in conjunction with the BPD ‘s prediction bit-vector if a Front-end redirect should be made.

## Jump and Jump-Register Instructions
The BPD makes predictions only on the direction (taken versus not-taken) of **conditional branches**. Non-conditional “jumps” (JAL) and “jump-register” (JALR) instructions are handled separately from the BPD.

The NLP learns any “taken” instruction’s PC and target PC - thus, the NLP is able to predict jumps and jump-register instructions.

If the NLP does not make a prediction on a JAL instruction, the pipeline will redirect the Front-end in F4.

Jump-register instructions that were not predicted by the NLP will **be sent down the pipeline with no prediction made**. As JALR instructions require reading the register file to deduce the jump target, there’s nothing that can be done if the NLP does not make a prediction.

## Updating the Backing Predictor
Generally speaking, the BPD is updated during the Commit stage. This prevents the BPD from being polluted by wrong-path information. However, as the BPD makes use of global history, this history must be reset whenever the Front-end is redirected. Thus, the BPD must also be (partially) updated during Execute when a misprediction occurs to reset any speculative updates that had occurred during the Fetch stages.

When making a prediction, the BPD passes to the pipeline a “response info packet”. This “info packet” is stored in the Fetch Target Queue (FTQ) until commit time. [11] Once all of the instructions corresponding to the “info packet” is committed, the “info packet” is set to the BPD (along with the eventual outcome of the branches) and the BPD is updated. The Fetch Target Queue (FTQ) for Predictions covers the FTQ , which handles the snapshot information needed for update the predictor during Commit. Rename Snapshot State covers the Branch Rename Snapshots , which handles the snapshot information needed to update the predictor during a misspeculation in the Execute stage.


## Managing the Global History Register (GHR)
The Global History Register (GHR) is an important piece of a branch predictor. It contains the outcomes of the previous N branches (where N is the size of the GHR ).

When fetching branch i, it is important that the direction of the previous i-N branches is available so an accurate prediction can be made. Waiting until the Commit stage to update the GHR would be too late (dozens of branches would be inflight and not reflected!). Therefore, the GHR must be updated speculatively, once the branch is fetched and predicted.

If a misprediction occurs, the GHR must be reset and updated to reflect the actual history. This means that each branch (more accurately, each Fetch Packet ) must snapshot the GHR in case of a misprediction.

There is one final wrinkle - exceptional pipeline behavior. While each branch contains a snapshot of the GHR , any instruction can potential throw an exception that will cause a Front-end redirect. Such an event will cause the GHR to become corrupted. For exceptions, this may seem acceptable - exceptions should be rare and the trap handlers will cause a pollution of the GHR anyways (from the point of view of the user code). However, some exceptional events include “pipeline replays” - events where an instruction causes a pipeline flush and the instruction is refetched and re-executed. For this reason, a commit copy of the GHR is also maintained by the BPD and reset on any sort of pipeline flush event.

## The Fetch Target Queue (FTQ) for Predictions
The Reorder Buffer (see The Reorder Buffer (ROB) and the Dispatch Stage ) maintains a record of all inflight instructions. Likewise, the FTQ maintains a record of all inflight branch predictions and PC information. These two structures are decoupled as FTQ entries are incredibly expensive and not all ROB entries will contain a branch instruction. As only roughly one in every six instructions is a branch, the FTQ can be made to have fewer entries than the ROB to leverage additional savings.

Each FTQ entry corresponds to one Fetch cycle. For each prediction made, the branch predictor packs up data that it will need later to perform an update. For example, a branch predictor will want to remember what index a prediction came from so it can update the counters at that index later. This data is stored in the FTQ .

When the last instruction in a Fetch Packet is committed, the FTQ entry is deallocated and returned to the branch predictor. Using the data stored in the FTQ entry, the branch predictor can perform any desired updates to its prediction state.

There are a number of reasons to update the branch predictor after Commit. It is crucial that the predictor only learns correct information. In a data cache, memory fetched from a wrong path execution may eventually become useful when later executions go to a different path. But for a branch predictor, wrong path updates encode information that is pure pollution – it takes up useful entries by storing information that is not useful and will never be useful. Even if later iterations do take a different path, the history that got it there will be different. And finally, while caches are fully tagged, branch predictors use partial tags (if any) and thus suffer from deconstructive aliasing.

Of course, the latency between Fetch and Commit is inconvenient and can cause extra branch mispredictions to occur if multiple loop iterations are inflight. However, the FTQ could be used to bypass branch predictions to mitigate this issue. Currently, this bypass behavior is not supported in BOOM.

## Rename Snapshot State
The FTQ holds branch predictor data that will be needed to update the branch predictor during Commit (for both correct and incorrect predictions). However, there is additional state needed for when the branch predictor makes an incorrect prediction and must be updated immediately. For example, if a misprediction occurs, the speculatively-updated GHR must be reset to the correct value before the processor can begin fetching (and predicting) again.

This state can be very expensive but it can be deallocated once the branch is resolved in the Execute stage. Therefore, the state is stored in parallel with the Branch Rename Snapshot s. During Decode and Rename, a Branch Tag is allocated to each branch and a snapshot of the rename tables are made to facilitate single-cycle rollback if a misprediction occurs. Like the branch tag and Rename Map Table snapshots, the corresponding Branch Rename Snapshot can be deallocated once the branch is resolved by the Branch Unit in Execute.


The Branch Predictor Pipeline. Although a simple diagram, this helps show the I/O within the Branch Prediction Pipeline. The Front-end sends the “next PC” (shown as req) to the pipeline in the F0 stage. Within the “Abstract Predictor”, hashing is managed by the “Abstract Predictor” wrapper. The “Abstract Predictor” then returns a Backing Predictor (BPD) response or in other words a prediction for each instruction in the Fetch Packet .

## The Abstract Branch Predictor Class
To facilitate exploring different global history-based BPD designs, an abstract “BrPredictor” class is provided. It provides a standard interface into the BPD and the control logic for managing the global history register. This abstract class can be found in Fig. 9 labeled “Abstract Predictor”. For a more detailed view of the predictor with an example look at Fig. 12.

### Global History
As discussed in Managing the Global History Register, global history is a vital piece of any branch predictor. As such, it is handled by the abstract BranchPredictor class. Any branch predictor extending the abstract BranchPredictor class gets access to global history without having to handle snapshotting, updating, and bypassing.

### Operating System-aware Global Histories
Although the data on its benefits are preliminary, BOOM does support OS-aware global histories. The normal global history tracks all instructions from all privilege levels. A second user-only global history tracks only user-level instructions.

## The Two-bit Counter Tables

The basic building block of most branch predictors is the “Two-bit Counter Table” (2BC). As a particular branch is repeatedly taken, the counter saturates upwards to the max value 3 (0b11) or strongly taken. Likewise, repeatedly not-taken branches saturate towards zero (0b00). The high-order bit specifies the prediction and the low-order bit specifies the hysteresis (how “strong” the prediction is).

These two-bit counters are aggregated into a table. Ideally, a good branch predictor knows which counter to index to make the best prediction. However, to fit these two-bit counters into dense SRAM, a change is made to the 2BC finite state machine – mispredictions made in the weakly not-taken state move the 2BC into the strongly taken state (and vice versa for weakly taken being mispredicted). The FSM behavior is shown in Fig. 11.

Although it’s no longer strictly a “counter”, this change allows us to separate out the read and write requirements on the prediction and hystersis bits and place them in separate sequential memory tables. In hardware, the 2BC table can be implemented as follows:

The P-bit:
- Read - every cycle to make a prediction
- Write - only when a misprediction occurred (the value of the h-bit).

The H-bit:
- Read - only when a misprediction occurred.
- Write - when a branch is resolved (write the direction the branch took).

By breaking the high-order p-bit and the low-order h-bit apart, we can place each in 1 read/1 write SRAM. A few more assumptions can help us do even better. Mispredictions are rare and branch resolutions are not necessarily occurring on every cycle. Also, writes can be delayed or even dropped altogether. Therefore, the h-table can be implemented using a single 1rw-ported SRAM by queueing writes up and draining them when a read is not being performed. Likewise, the p-table can be implemented in 1rw-ported SRAM by banking it – buffer writes and drain when there is not a read conflict.

A final note: SRAMs are not happy with a “tall and skinny” aspect ratio that the 2BC tables require. However, the solution is simple – tall and skinny can be trivially transformed into a rectangular memory structure. The high-order bits of the index can correspond to the SRAM row and the low-order bits can be used to mux out the specific bits from within the row.

## The GShare Predictor
GShare is a simple but very effective branch predictor. Predictions are made by hashing the instruction address and the GHR (typically a simple XOR) and then indexing into a table of two-bit counters. Fig. 10 shows the logical architecture and Fig. 12 shows the physical implementation and structure of the GShare predictor. Note that the prediction begins in the F0 stage when the requesting address is sent to the predictor but that the prediction is made later in the F3 stage once the instructions have returned from the instruction cache and the prediction state has been read out of the GShare’s p-table.

## The TAGE Predictor

BOOM also implements the TAGE conditional branch predictor. TAGE is a highly-parameterizable, state-of-the-art global history predictor. The design is able to maintain a high degree of accuracy while scaling from very small predictor sizes to very large predictor sizes. It is fast to learn short histories while also able to learn very, very long histories (over a thousand branches of history).

TAGE (TAgged GEometric) is implemented as a collection of predictor tables. Each table entry contains a prediction counter, a usefulness counter, and a tag. The prediction counter provides the prediction (and maintains some hysteresis as to how strongly biased the prediction is towards taken or not-taken). The usefulness counter tracks how useful the particular entry has been in the past for providing correct predictions. The tag allows the table to only make a prediction if there is a tag match for the particular requesting instruction address and global history.

Each table has a different (and geometrically increasing) amount of history associated with it. Each table’s history is used to hash with the requesting instruction address to produce an index hash and a tag hash. Each table will make its own prediction (or no prediction, if there is no tag match). The table with the longest history making a prediction wins.

On a misprediction, TAGE attempts to allocate a new entry. It will only overwrite an entry that is “not useful” (ubits == 0).

### TAGE Global History and the Circular Shift Registers (CSRs)

Each TAGE table has associated with it its own global history (and each table has geometrically more history than the last table). The histories contain many more bits of history that can be used to index a TAGE table; therefore, the history must be “folded” to fit. A table with 1024 entries uses 10 bits to index the table. Therefore, if the table uses 20 bits of global history, the top 10 bits of history are XOR’ed against the bottom 10 bits of history.

Instead of attempting to dynamically fold a very long history register every cycle, the history can be stored in a circular shift register (CSR). The history is stored already folded and only the new history bit and the oldest history bit need to be provided to perform an update. Listing 2 shows an example of how a CSR works.

Listing 2 The circular shift register. When a new branch outcome is added, the register is shifted (and wrapped around). The new outcome is added and the oldest bit in the history is “evicted”.

Each table must maintain three CSRs. The first CSR is used for computing the index hash and has a size n=log(num_table_entries). As a CSR contains the folded history, any periodic history pattern matching the length of the CSR will XOR to all zeroes (potentially quite common). For this reason, there are two CSRs for computing the tag hash, one of width n and the other of width n-1.

For every prediction, all three CSRs (for every table) must be snapshotted and reset if a branch misprediction occurs. Another three commit copies of these CSRs must be maintained to handle pipeline flushes.

### Usefulness counters (u-bits)
The “usefulness” of an entry is stored in the u-bit counters. Roughly speaking, if an entry provides a correct prediction, the u-bit counter is incremented. If an entry provides an incorrect prediction, the u-bit counter is decremented. When a misprediction occurs, TAGE attempts to allocate a new entry. To prevent overwriting a useful entry, it will only allocate an entry if the existing entry has a usefulness of zero. However, if an entry allocation fails because all of the potential entries are useful, then all of the potential entries are decremented to potentially make room for an allocation in the future.

To prevent TAGE from filling up with only useful but rarely-used entries, TAGE must provide a scheme for “degrading” the u-bits over time. A number of schemes are available. One option is a timer that periodically degrades the u-bit counters. Another option is to track the number of failed allocations (incrementing on a failed allocation and decremented on a successful allocation). Once the counter has saturated, all u-bits are degraded.

### TAGE Snapshot State
For every prediction, all three CSRs (for every table) must be snapshotted and reset if a branch misprediction occurs. TAGE must also remember the index of each table that was checked for a prediction (so the correct entry for each table can be updated later). Finally, TAGE must remember the tag computed for each table – the tags will be needed later if a new entry is to be allocated. [16]

## Other Predictors
BOOM provides a number of other predictors that may provide useful.

### The Base Only Predictor
The Base Only Predictor uses the BTBs BIM to make a prediction on whether the branch was taken or not.

### The Null Predictor
The Null Predictor is used when no BPD predictor is desired. It will always predict “not taken”.

### The Random Predictor
The Random Predictor uses an LFSR to randomize both “was a prediction made?” and “which direction each branch in the Fetch Packet should take?”. This is very useful for both torturing-testing BOOM and for providing a worse-case performance baseline for comparing branch predictors.



