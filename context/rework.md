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
    "LLM": {
        type: "node",
        kind: "claude code",
        parallelism_settings: [
            num_nodes: 4 // 4 llms active
        ],
        params: [
            "effort": "high"
            "timeout": "600s"
            or
            extern_file: "path to extern config file"
        ],
        input_files: [
            ISA_Spec: "path to pdf/md",
        ],
        output_files: [
            Patch: "llm_node:~/chia/boom-riscv-extension.patch" // this means its on the llm node container's file system, the user specifies the generic name, then each proposal has a tag attached to it: ex: boom-riscv-extension-20261103-14-51-55-2333Z.patch, and llm_node[0-3] is the manual denotation of which container we are trying to access. | for each patch, a boom-riscv-extension-latest.patch symlink should always be maintained to the latest output file from that chia node.
        ],
        connected_pipes:[
            patch_apply: [out] // in/out from the perspective of the node
            
        ],
    },
}





building should be abstracted into multiple yamls, but there should be a combined yaml so that peopel dont need to have 1 yaml/function open


lanaguage is object definition order agnostic


files (in transit or at rest) must be visible to the user (even if residing in container fs)

some additional features:
live loop visualization while building
intellisence like template generation 

nodes & pipes (pre-proc)


each chia function implemented in python has a set of inputs/outputs.. we can then check against our input_files/output_files array + that particular node's configs to do a semantic check if we are missing any params: https://docs.chialoops.ai/en/main/api/firesim.html or https://docs.chialoops.ai/en/main/api/chipyard.html#chia.chipyard.chisel_build_node.ChiselBuildNode (ex: the chisel build node's configs + inputs should equal the function signature's expected inputs)


ray is bad at scheduling cpu intensive tasks
so much python slop/glue, not enough safe guards
we need a new fw that is faster than natural language to agent (templating)
visualization as loop is building (see chia website)
visibility into what each agent/parallel part of loop is doing (file system mount, time machine like explore)
graph viz is good for a flamegraph (or execution time of a loop), terrible for anything else (ex: evolutionary score, or implmentation progress overtime, etc - we need a neutral framework such that we can tag any value and graph them)