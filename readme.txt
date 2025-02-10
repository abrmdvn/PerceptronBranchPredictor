In the CSV file:
For history length: lo, mi and hi correspond to 20, 40, 60;
For perceptron table size: lo and hi corespnd to 16 and 64.

Dynamic Branch Prediction Using Perceptrons
https://github.com/abrahmd/perceptron_branch_predictor.git
Abrahm DeVine & Mengxiao Hu
CSE220 Fall 2024

Introduction
In this lab, we implemented a perceptron-based branch predictor, inspired by the approach detailed in Dynamic Branch Prediction with Perceptrons by Daniel Jimenéz and Calvin Lin from UT Austin. Our implementation explores the impact of two parameters, the size of the global bit history table and the number of perceptrons stored, on performance optimization compared to GShare.

Implementation
We implemented our perceptron-based branch predictor in Scarab, leveraging existing architecture to test the performance of our predictor across multiple workloads. Our implementation begins by fetching the branch address and hashing it to index a table of perceptrons. Given this perception and the global history, we generate a prediction for the given branch. Once the branch outcome is known, we use this outcome and our prediction to train the perceptron before inserting it back into the perceptron table. This implementation required the following file additions and modifications.
perceptron.cc & perceptron.h
These files hold the implementation of the branch predictor. It defines a Perceptron_Entry struct, that represents a perceptron with a bias term and a vector of weights. Additionally, a Perceptron_State struct holds a vector of Perceptron_Entry structs, representing a table of perceptrons. The function bp_perceptron_init() sets the size of the perceptrons to the global history table length and sets all the terms to zero.
	The function bp_perceptron_pred(Op* op)  holds the logic for making a prediction. It takes information about the branch instruction, hashes its address, retrieves a perceptron form the Perceptron_State struct, and creates a prediction y using the following formula:

Here, w0is the bias term which we add to the dot product of the weight vector (w) and the global branch history bits (x).
	The function bp_perceptron_update(Op*op) is called after the outcome of the branch is determined. The function trains and updates the corresponding perceptron based on this outcome. If the outcome matches the prediction, the perceptron will only be trained if its magnitude is less than the threshold , defined in the paper as = 1.93h + 14. If the outcome does not match the prediction, the perceptron will be trained according to the following formula:

Here, t=1 if the branch was taken and t=-1 otherwise. The weights vector will thus increase in magnitude for taken branches and decrease otherwise.
bp.c & bp.h
These files are part of the Scarab implementation and handle branch prediction. The functions that we changed here correspond directly to the functions implemented in percepron.cc. 
In the function  init_bp_data(), we call our  bp_perceptron_init()  function. 
In bp_predicit(Bp_data* bp_data, Op* op, uns br_num, Addr fetch_addr), different branch types are handled. Specifically, we check ifop->table_info->cf_type is equal to CF_CBR to identify conditional branches. This is where we call our bp_perceptron_pred(Op* op) to make predictions. 
Finally, in bp_resolve_op(Bp_data* bp_data, Op* op), we have the outcome of the branch. With this outcome, we call our  bp_perceptron_update(Op*op)  function to train the corresponding perceptron.
bp.param.def
	In this file, we added parameter definitions for the perceptron-based branch predictor. The parameters hist_length and perceptron_table_size are used to specify the history length and the size of the perceptron table, respectively. 

Results
Figures 1 and Figure 2 show on-path conditional branch prediction accuracy for 23 different workloads. We compare accuracy for varying degrees of history length and perceptron table size, observing that increasing each parameter results in higher accuracy.
Blue bar: Represents results for implementation with a history length of 20 and table size of 16. This is our lowest performing implementation. 
Red bar: We see, when we increase the table size to 64, accuracy always increases compared to the blue bar(or in some cases, remains unchanged). This is due to decreased aliasing when hashing the branch address to the perceptron table. 
Yellow bar: If we instead increase the history length to 60, we also see performance improvement compared to the blue bar. Considering a longer history improves performance. However, this increases aliasing, occasionally resulting in worse performance against the red bar.
Green bar: Our best result comes with a history length of 60 and a table size of 64. This allows us to consider a long history while balancing aliasing issues. We found that increasing these parameters further had diminishing returns. 
Orange Bar: GShare performance.
Our branch predictor never outperformed GShare. However, it is within a few percentage points on every trial. Given the simplicity of implementation, we conclude that perceptron-based branch predictors can yield highly accurate results that are competitive with industry-standard predictors.


Figure 1: On path conditional branch prediction accuracy for 11 different workloads. In the legend, HL stands for “history length”, the number of bits in the global bit history register. TS stands for “table size”, the size of the perceptron table.

Figure 2: On path conditional branch prediction accuracy for 12 different workloads. In the legend, HL stands for “history length”, the number of bits in the global bit history register. TS stands for “table size”, the size of the perceptron table.

