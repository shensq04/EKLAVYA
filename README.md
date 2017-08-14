# Neural Nets Can Learn Function Type Signatures From Binaries

## Authors
EKLAVYA is designed by Zheng Leong Chua, Shiqi Shen, Prateek Saxena, Zhenkai Liang.

## Dataset
The dataset available from this page is the collection of function type signatures, which include function banaries, arguments counts and types. It is a good dataset for people who want to try learning techniques or heuristive methods in binary analysis while spending less effort on collecting and preprocessing.

 The dataset consists of three parts, which is described below:

- [binary.tar.gz](https://drive.google.com/open?id=0B2qBKMQRQLHGS1JESVQ0TlF1eWs). The compressed file saves 5168 binaries. These binaries are from 8 distinct packages: binutils, coreutils, findutils, sg3utils, utillinux, inetutils, diffutils, and usbutils.
These 5168 binaries are obtained by using two commonly used compilers: *gcc* and *clang*, with different optimization levels ranging from *O0* to *O3* for both x86 and x64.

- [pickles.tar.gz](https://drive.google.com/open?id=0B2qBKMQRQLHGdGpuTUlmMmZJYXM). The compressed file saves the assembly code of each function and the ground truth of the function arguments by parsing the DWARF debug information. 

- [clean_pickles.tar.gz](https://drive.google.com/open?id=0B2qBKMQRQLHGOFphWjkzcnV2LTQ). The compressed file saves the assembly code and the ground truth of the function arguments for sanitized functions. For this dataset, we removed the functions which are duplicates of other functions in the dataset. Given that the same piece of code compiled with different binaries will result in different offsets generated,we chose to remove all direct address used by instructions found in the function. For example, the instruction *'je 0x98'* are represented as *'je '*. After the substituion, we hash the function and remove functions with the same hashes. Other than duplicates, we also removed functions with less than four instructions as these small functions typically do not have any operation on arguments. 


### Function Representation
A function is represented as a dictionary having the following fields:

- **num_args**: Integer - Number of arguments.
- **args_type**: List - Type of each argument, as String, in the order they appear in the function declaration.
- **ret_type**: String - Type of the value returned by the function.
- **inst_strings**: List - The assembly code of each instruction composing the function, as strings.
- **inst_bytes**: List - The bytecode of each instruction composing the function, as an array of values. (Each instruction is represented by one array of bytes.)
- **boundaries**: Tuple (Integer, Integer) - The starting and ending address of the function.

**Example:**
```
FuncDict = {
	"num_args": 3,
	"args_type": ["int", "char", "struct structure1*"], 
	"ret_type": "int", 
	"inst_strings": ["mov eax, 1", "nop", "push 3"],
	"inst_bytes": [[0x01, 0x02], [0xff], [0x20, 0x30, 0x40]],
	"boundaries": (0x80010, 0x800f0)
}
```

### Binary Representation
A binary saved in **pickles.tar.gz** and **clean_pickles.tar.gz** is represented as a Dict object, having the following fields:

- **functions**: Dict - Ths dictionary contains as keys function names and as values function dictionaries which is described above. The information of dupilicate functions are removed from the dictionary.
- **structures**: Dict -The dictionary describing structures used in the binary files. Dictionary's keys are names of the structures and the values are lists containing the type of each field in the structure, in the order they are declared.
- **text_addr**: String - Address of the text section in the binary.
- **binRawBytes**: String - Raw contents of the binary file.
- **arch**: String - Binary architecture.
- **binary_filename**: String - Name of the binary elf file used to generate the binInfo.
- **function_calls**: Dict - The dictionary containing information about function calls. A caller is described using its name and an array of instruction indices, starting from 0. (If **10** is in the array, the **10th** instruction in the caller is a call to the callee.)
	- Key: callee function's name
	- Value: array of objects describing function callers

**Example:**
```
BinaryFileDict = {
    "functions": {
        "function1": funcDict1,
        "function2": funcDict2
    },
    "structures": {
        "structure1": ["int", "char [16]", "float"],
        "structure2": ["long", "long", "short"]
    },
    "text_addr": 0x800000,
    "binRawBytes": "\x00\x12\x22...",
    "arch": "i386",
    "binary_filename": "gcc-32-O1-coreutils-ls",
    "function_calls": {
         "func1": [
             {
             	 # caller's name
                 "caller": "func2",
                 # the indices of calling instructions
                 "call_instr_indices": [10, 17, 29] 
             },
             {
                 "caller": "func2",
                 "call_instr_indices": [19]
             }
        ]
    }
}
```
## Code

### Requirement

- tensorflow
- numpy

### Train Embedding Model

1. Prepare the input file for training embedding model
```
python prep_embed_input.py [options] -i input_folder
```
[Link to prep_embed_input.py](code/embedding/prep_embed_input.py)

- **input_folder**: String - The data folder saving all binary information.

Options:

- **-o**: String - The input file for training embedding model
- **-e**: String - The file saving all error information
- **-m**: String - The file saving the map information (int --> instruction (int list))

2. Train the embedding model
```
python train_embed.py -i input_path
```
[Link to train_embed.py](code/embedding/train_embed.py)

- **input_path**: String - The input file for training embedding model 

Options:
- **-o**: String - The output folder saving the trained embedding information
- **-tn**: Integer - Number of threads
- **-sw**: Integer - Saving frequency (Number of epochs). The trained information will be saved every several epochs.
- **-e**: Integer - Dimension of the embedding vector for each instruction.
- **-ne**: Integer - Number of epochs for training the embedding model
- **-l**: Float - Learning rate
- **-nn**: Integer - Number of negative samples
- **-b**: Integer - Batch size
- **-ws**: Integer - Window size
- **-mc**: Integer - Ignoring all words with total frequency lower than this.
- **-s**: Float - Subsampling threshold

3. Save the embedding vector
```
python save_embeddings.py [options] -p embed_pickle_path -m model_path
```
[Link to save_embeddings.py](code/embedding/save_embeddings.py)

- **embed_pickle_path**: String - The file saving all training parameters
- **model_path**: String - The file saving the trained embedding model

Options:
- **-o**: String: The output file saving the embedding vector for each instruction

### Train RNN Model
```
python train.py [options] -d data_folder -o output_dir -f split_func_path -e embed_path
```
[Link to train.py](code/RNN/train/train.py)

- **data_folder**: The folder saves the binary information.  
- **output_dir**: The directory is used to save the trained model & log information.
- **[split_func_path](#split_func_path-format)**: The file saves the training & testing function names.
- **embed_path**: The file is the output file of **save_embeddings.py**, which saves the embedding vector of each instruction.

Options:

- **-t**: Type of output labels. Possible value: num_args, type#0, type#1, ... (Default value: num_args)
	- **num_args**: The trained model is used to predict the number of arguments for each function.
	- **type#0**: The trained model is used to predict the type of first argument for each function.
	- **type#1**: The trained model is used to predict the type of second argument for each function.
	- ...
- **-dt**: Type of input data. Possible value: caller and callee. (Default value: callee)
	- **caller**: The input data is from caller.
	- **callee**: The input data is function body.
- **-pn**: Number of Processes. (Default value: 40)
- **-ed**: Dimension of embedding vector for each instruction. (Default value: 256)
- **-ml**: Maximum length of input sequences. (Default value: 500)
- **-nc**: Number of Classes. (Default value: 16)
- **-en**: Number of Epochs. (Default value: 100)
- **-s**: The frequency for saving the trained model. If the value is 100, the trained model is going to be saved every 100 batches. (Default value: 100)
- **-do**: Dropout value. (Default value: 0.8)
- **-nl**: Number of layers in RNN. (Default value: 3)
- **-ms**: Maximum number of model saved in the directory. (Default value: 100)
- **-b**: Batch size. (Default value: 256)
- **-p**: The frequency for showing the accuracy & cost value. (Default value: 20)

#### split_func_path Format
The split_func_path file saves the function names for training & testing dataset. If you are going to predict the type signatures from callees (function bodies), the function name is represented as "binary_file_name#func_name". If you are going to predict the type signatures from callers, the function name is represented as "binary_file_name#callee_name#caller_name#call_insn_indice".

Examples of split_func_path file for callees:
```
splitFuncDict = {
    'train':[
                'gcc-32-O1-binutils-objdump.pkl#OP_Rounding',
                'clang-32-O1-coreutils-csplit.pkl#keep_new_line',
                'gcc-32-O3-coreutils-mv.pkl#copy_internal',
                ...
            ],
    'test': [
                'gcc-32-O3-findutils-find.pkl#parse_amin',
                'gcc-32-O2-findutils-find.pkl#pred_size',
                'clang-32-O1-findutils-find.pkl#debug_strftime',
                ...
            ]
}
```

Examples of split_func_path file for callers:

```
splitFuncDict={
    'train':[
                'clang-32-O3-utillinux-dmesg.pkl#strnchr#print_record#283',
                'gcc-32-O3-coreutils-numfmt.pkl#process_line#main#386',
                'gcc-32-O3-coreutils-numfmt.pkl#process_line#main#557',
                ...
            ],
    'test': [
                'clang-32-O0-utillinux-utmpdump.pkl#gettok#undump#123',
                'gcc-32-O0-utillinux-ionice.pkl#ioprio_print#main#315',
                'clang-32-O1-inetutils-ping.pkl#ping_set_packetsize#ping_echo#24',
                ...
            ]
}
```

<!-- #### embed_path Format
The embed_path file saves a dictionary saving the embedding vectors for all instruction. The key of this dictionary is a special string for each instruction. For example, if the bytes vector for one instruction is [232, 164, 254, 0, 0], the key for this instruction is '[232, 164, 254, 0, 0]'. The value of each instruction 

Example of embed_path:
embedDict = {
    '[232, 164, 254, 0, 0]': {
        'vector': [-0.15424642, -0.03994527, -0.06539968, 0.099554, ...]
    }, 
    '[186, 195, 128, 10, 8]': {
        'vector': [0.09991222, 0.05001251, 0.11093043, 0.0041295, ...]
    },
    ...
} -->

### Testing RNN Model
Usage: 
```
python eval.py [options] -d data_folder -f split_func_path -e embed_path -m model_dir -o output_dir
```
[Link to eval.py](code/RNN/test/eval.py)

- **data_folder**: The folder saves the binary information.  
- **[split_func_path](#split_func_path-format)**: The file saves the training & testing function names.
- **embed_path**: The file is the output file of **save_embeddings.py**, which saves the embedding vector of each instruction.
- **model_dir**: The directory is used to save the trained model & log information.
- **output_dir**: The directory is used to saved the predicted results and true labels of each function for each model.

Options:

- **-t**: Type of output labels. Possible value: num_args, type#0, type#1, ... (Default value: num_args)
	- **num_args**: The trained model is used to predict the number of arguments for each function.
	- **type#0**: The trained model is used to predict the type of first argument for each function.
	- **type#1**: The trained model is used to predict the type of second argument for each function.
	- ...
- **-dt**: Type of input data. Possible value: caller and callee. (Default value: callee)
	- **caller**: The input data is from caller.
	- **callee**: The input data is function body.
- **-pn**: Number of Processes. (Default value: 40)
- **-ed**: Dimension of embedding vector for each instruction. (Default value: 256)
- **-ml**: Maximum length of input sequences. (Default value: 500)
- **-nc**: Number of Classes. (Default value: 16)
- **-do**: Dropout value.(Default value: 1.0)
- **-nl**: Number of layers in RNN. (Default value: 3)
- **-b**: Batch size. (Default value: 256)



## Disclaimer
The code is research-quality proof of concept, and is still under development for more features and bug-fixing.

## References
Neural Nets Can Learn Function Type Signatures From Binaries

Zheng Leong Chua, Shiqi Shen, Prateek Saxena, Zhenkai Liang.

In the 26th USENIX Security Symposium (Usenix Security 2017)

## Project Members
Zheng Leong Chua, Shiqi Shen, Prateek Saxena, Zhenkai Liang, Valentin Ghita.