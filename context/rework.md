we want to rework the abstraction of the CHIA framework a bit:

Problem: currently, CHIA loops written using AI agents have a problem of overusing python to connect CHIA functions such that the loops become incomprehensible, or overuses python to build out the loop/harness itself, and in some cases, avoids using the built in CHIA abstractions completely (ex: directly accessing SQL, using bash tool to directly call commands).

We want CHIA tool to be a platform where someone can quickly connect up individual tools needed as part of the hardware/software co-design cycle and minimize the use of python as glue.

The goal is to create a way of communicating these loops faster than describing them to an agent to build out in CHIA via natural language. Examples of these loops are shown in 2606.27350v3.pdf. When agents write these loops, agents may be good at parsing python, but humans are not.

however, we still would like the flexiblity of the tool, in the sense that it should be easy to sweep a parameter in a particular design, and then "cross bar that -- for x parameter in set of {y1...y_n} designs". however, we don't want all of these parameters to end up in 1 config file, which overwhelms the user with too many knobs.

to fix this, we should come up with a fast templating language that allows a human to quickly construct a loop using the existing CHIA functions (see /scratch/jfx/chia-abs-exp/chia). (If users need ChiaFunctions that don't exist, they should be expected to create it in python, in absence of the loop, and then they are able to use it in this templating language).

This templating language can be yaml or a language in a similar vain. it should be hierarchial, with a clear specification of ordering (never any ambigiity about which order the yamls are parsed in. 

```
// loop.chia
{
    "LLM": { // commmand clicking this should jump to the respective node in the visualization and "highlight-flash" it
        type: "node",
        kind: "claude code", // refering to a ChiaFunction == node function. command clicking this name should open the respective python function implemenation in vscode
        parallelism_settings: [
            <!-- num_nodes: 4 // 4 llms active -->
            num_nodes: 1 // 1 llm active
        ],
        params: [
            "effort": "high"
            "timeout": "600s"
            or
            extern_file: "path to extern config file"
        ],
        input: [
            ISA_Spec: "path to pdf/md",
        ],
        output: [
            Patch: "llm_node:~/chia/boom-riscv-extension.patch" // this means its on the llm node container's file system, the user specifies the generic name, then each proposal has a tag attached to it: ex: boom-riscv-extension-20261103-14-51-55-2333Z.patch, and llm_node[0-3] is the manual denotation of which container we are trying to access. | for each patch, a boom-riscv-extension-latest.patch symlink should always be maintained to the latest output file from that chia node.
        ],
        connections:[
            straight_pipe_llm_to_boom_build: [out] // in/out from the perspective of the node
            test_fail: [in]
        ],
    },
    "boom_build": {
        type: "node",
        kind: "chipyard",
        parallelism_settings: [
            num_nodes: 1
        ],
        params: [],
        input: [
            Patch: "llm_node:~/chia/boom-riscv-extension.patch"
        ],
        output: [
            BuildArtifact: "boom_build:~/chipyard/sims/verilator/simulator.boom-riscv-extension"
        ],
        connections: [
            LLM: [in],
            cosin instr.test: [out]
        ]
    },
    "cosim-instr.test compile": {
        type: "node",
        kind: "riscv-compile",
        parallelism_settings: [
            num_nodes: 1 // num containers
            num_threads: 32 // -j32
            num_concurrent_execs: 4 // 4 make cmds concurrently
        ],
        params: [],
        input: [
            test_db: "postgress:instr. Test db"
        ]
        output: [
            compiled_riscv_elf: "riscv-compile:~/chipyard/tests/...."
        ]
        connections: [
            cosim instr.test: [out]
        ]
    },
    "cosim instr. test": {
        type: "node",
        kind: "verilator",
        parallelism_settings: [
            num_nodes: 10
        ],
        params: [],
        input: [
            BuildArtifact: "boom_build:~/chipyard/sims/verilator/simulator.boom-riscv-extension"
            compiled_riscv_elf: "riscv-compile:~/chipyard/tests/...."
        ],
        output: [
            log: "",
            out: "",
        ],
        connections: [
            //  conenction is inferred from input & output files in addition to what is specified. what is specified here overrides any auto-inferred results
            all_instr_pass: [out]
        ]
    }
    "all_instr_pass": {
        type: "pipe" // used for moving things between nodes with light data processing
        kind: "quad-pipe-all-instr-pass", // python function defines the behavior of this
        connections: [
            cosim instr. test: [in],
            test_fail: [out]
            cosim_instr._test_pass: [out]
            synthesis: [out]
        ]
    }
    "verilator_run": {
        type: "node",
        kind: "verilator",
        parallelism_settings: [
            num_nodes: 10
        ],
        params: [],
        input: [
            BuildArtifact: "boom_build:~/chipyard/sims/verilator/simulator.boom-riscv-extension"
            riscv_asm_tests_db: "sql:riscv_asm_tests"
        ]
        output: [
            log: "",
            out: "",
        ]
        connections: [ //these should be more of like "previous step done"/"start this step" signals rather than transfering files/dependent collateral -- that should be done thru input
            cosim_instr._test_pass: [in]
        ]
    }
    "synthesis": {
        type: "node",
        kind: "synth",
        parallelism_settings: [
            num_nodes: 1
        ],
        params: [],
        input: [
            BuildArtifact: "boom_build:~/chipyard/sims/verilator/simulator.boom-riscv-extension" // these (along with HammerResult) are structs and we will need store them in code, and if they point to a file path it'll be referenced in a struct
        ],
        output: [
            HammerResult: ""
        ]
    }
    "base isa ok?": {
        type: "pipe"
        kind: "tri-pipe-base-isa-ok",
        connections: [
            "synthesis": [in]
            "test_fail": [out]
            "test_pass: [out]
        ]
    }
}




<!-- 
"patch_apply_llm_to_build": {
        type: "pipe",
        kind: "apply patch", // refering to a ChiaPipe function, if we need to 4 in:1 out type of pipe, this is handeled in the pipe's python implementation.
        connected_nodes: [
            LLM: [in],
            boom_build: [out]
        ]
    },
 -->

 <!--     "straight_pipe_llm_to_boom_build": {
        type: "pipe",
        kind: "straight pipe", // refering to a ChiaPipe function (this is a pipe in | pipe out with no additional processing), if we need to 4 in:1 out type of pipe, this is handeled in the pipe's python implementation.
        connected_nodes: [
            LLM: [in],
            boom_build: [out]
        ]
    },
     -->

chiafunctions/chiapipes are not allowed to call other user defined python functions.

when user starts a new instantiation of a pipe or node, intellisense should parse existing nodes and suggest what possible nodes can exist, if user hit tab, most of the I/O section & parallelism section should already be filled out when the instantiation template for that node is put down.

probably remove defintion of chiafunction (rn chia function can contain many ray nodes/individual schedulable units).. this is confusing (surfacing up parallelism headaches to the user)
* prob should just use node semantics

building should be abstracted into multiple yamls, but there should be a combined yaml so that peopel dont need to have 1 yaml/function open


lanaguage is object definition order agnostic


files (in transit or at rest) must be visible to the user (even if residing in container fs)

some additional features:
live loop visualization while building (mermaid?)
intellisence like template generation 

nodes & pipes (pre-proc)


each chia function implemented in python has a set of inputs/outputs.. we can then check against our input/output array + that particular node's configs to do a semantic check if we are missing any params: https://docs.chialoops.ai/en/main/api/firesim.html or https://docs.chialoops.ai/en/main/api/chipyard.html#chia.chipyard.chisel_build_node.ChiselBuildNode (ex: the chisel build node's configs + inputs should equal the function signature's expected inputs)


ray is bad at scheduling cpu intensive tasks
so much python slop/glue, not enough safe guards
we need a new fw that is faster than natural language to agent (templating)
visualization as loop is building (see chia website)
visibility into what each agent/parallel part of loop is doing (file system mount, time machine like explore)
graph viz is good for a flamegraph (or execution time of a loop), terrible for anything else (ex: evolutionary score, or implmentation progress overtime, etc - we need a neutral framework such that we can tag any value and graph them)