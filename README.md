# Supera module
`Supera` is designed to slice up the data products output from `LArSoft` and package them to be consumed by the [SPINE](https://github.com/DeepLearnPhysics/spine) workflow. The bulk of the work is done by `SuperaMCParticleCluster` which consumes inputs from `simenergydeposit(lite)`, `cluster3d`, and MC truth products such as `MCTrack`, `MCShower`, and `MCParticle`. This module relies on the [larcv2](https://github.com/DeepLearnPhysics/larcv2) framework to configure, run, and save the output.

The input data products for this module are the `recob::SpacePoints`, in particular, the module `Cluster3D` has been optimized to generate them in the `reco` stage. Then, with the reconstructed output from either data or simulation we can run `Supera` to generate `larcv.root` files which will contain information about the reconstructed 3D points, as well as the MC truth information for simulated events. To run we need to execute:
```
lar -c run_supera_xx.fcl -s reco_file.root
```
Where `run_supera_xx.fcl` is a common `fcl` file that calls `supera_xx.fcl`. Make sure both files exist and their paths are correct. Moreover, the details on how `Supera` will be executed are specified in `supera_xx.fcl` so make sure to check this file. Examples on these configurations files can be found in the folder `job`. Here you can find a commented file.
<details>
<summary>Click to expand supera_mpvmpr.fcl</summary>
    ```
        ProcessDriver: {       

        Verbosity:    2         # [0,1,2,3,4,5] from most to least verbose
        EnableFilter: true      # Enable event filtering
        RandomAccess: false     # Enable random access to events
        ProcessType:  ["SuperaMCTruth","SuperaMCTruth","SuperaBBoxInteraction","SuperaMCParticleCluster","SuperaSimEnergyDeposit","SuperaSpacePoint","Tensor3DFromCluster3D","ThresholdTensor3D","CombineTensor3D","ParticleCorrector","EmptyTensorFilter","RescaleChargeTensor3D","SuperaOptical"] # Modules to be run in order. You can find all the files in the Supera main folder
        ProcessName:  ["MultiPartVrtx","MultiPartRain","SuperaBBoxInteraction","SuperaMCParticleCluster","SuperaSimEnergyDeposit","SuperaSpacePoint","Tensor3DFromCluster3D","ThresholdTensor3D","CombineTensor3D","ParticleCorrector","EmptyTensorFilter","RescaleChargeTensor3D","SuperaOptical"] # Process to be run, list must match ProcessType length as they are mapped 1 to 1

        IOManager: {        
            Verbosity:   2        # [0,1,2,3,4,5] from most to least verbose
            Name:        "IOManager" 
            IOMode:      1
            OutFileName: "out_test.root" # Output file name
            InputFiles:  []
            InputDirs:   []
            StoreOnlyType: []
            StoreOnlyName: []
        }

        ProcessList: { # Details on the configurationof each of the modules in ProcessType/ProcessName
            SuperaOptical: {
                    OpFlashProducers: ["opflashCryoE"]  # producer to be used from the reco_file.root given as input
                    OpFlashOutputs: ["cryoE"]           # output name to be used in the larcv.root file
                    }
            EmptyTensorFilter: {
                Profile: true
                Tensor3DProducerList: ["reco"]
                MinVoxel3DCountList:  [1]
                }
            RescaleChargeTensor3D: {
                HitKeyProducerList:    ["reco_hit_key0","reco_hit_key1","reco_hit_key2"]
                HitChargeProducerList: ["reco_hit_charge0","reco_hit_charge1","reco_hit_charge2"]
                OutputProducer:        "reco_rescaled"  # new producer name
                ReferenceProducer:     "pcluster"       # producer to be used as reference for the rescaling
                }
            ThresholdTensor3D: {
                Profile: true
                TargetProducer: "reco"
                OutputProducer: "pcluster_semantics_ghost"
                PaintValue: 5
                }
            CombineTensor3D: {
                Profile: true
                Tensor3DProducers: ["pcluster_semantics_ghost","pcluster_semantics"]
                OutputProducer:    "pcluster_semantics_ghost"
                PoolType: 0
                }
            SuperaMCParticleCluster: {
                Profile: true
                OutputLabel: "pcluster"
                LArMCParticleProducer: "largeant"          # input MCParticle producer from the reco_file.root given as input
                LArMCShowerProducer: "mcreco"              # input MCShower producer from the reco_file.root given as input
                LArMCTrackProducer:  "mcreco"              # input MCTrack producer from the reco_file.root given as input
                DeltaSize: 10                              # in cm, distance to merge delta rays
                LArSimEnergyDepositLiteProducer: "sedlite" # input SimEnergyDepositLite producer from the reco_file.root given as input
                Meta3DFromCluster3D: "mcst"
                Meta2DFromTensor2D:  ""
                Verbosity: 1
                UseSimEnergyDeposit: false                 # if true, it will use SimEnergyDeposit instead of SimEnergyDepositLite
                UseSimEnergyDepositLite: true              # if true, it will use SimEnergyDepositLite
                UseSimEnergyDepositPoints: true            # if true, it will use the points from SimEnergyDepositLite
                CryostatList: [0,0,0,0,1,1,1,1]            # for two cryostats (DEPENDS ON YOU GEOMETRY!)
                TPCList: [0,1,2,3,0,1,2,3]                 # TPCList: [0,1,2,3] # for single cryostat (DEPENDS ON YOU GEOMETRY!)
                PlaneList: []
                SemanticPriority: [1,2,0,3,4] # 0-4 for shower track michel delta LE-scattering

                SuperaTrue2RecoVoxel3D: {
                    DebugMode: true
                    Profile: true
                    Verbosity: 1
                    Meta3DFromCluster3D: "pcluster"
                    LArSimChProducer: "largeant"         # input SimChannel producer from the reco_file.root given as input
                    LArSpacePointProducers: ["cluster3DCryoE"]
                    OutputTensor3D:  "masked_true"
                    OutputCluster3D: "masked_true2reco"
                    TwofoldMatching: true
                    UseTruePosition: true
                    HitThresholdNe: 100                  # TO BE OPTIMIZED FOR YOUR DETECTOR
                    HitWindowTicks: 15 #5                # TO BE OPTIMIZED FOR YOUR DETECTOR
                    HitPeakFinding: false
                    PostAveraging: true 
                    PostAveragingThreshold_cm: 0.425     # TO BE OPTIMIZED FOR YOUR DETECTOR
                    DumpToCSV: false
                    RecoChargeRange: [-1000,50000]
                    VoxelDistanceThreshold: 3.           # TO BE OPTIMIZED FOR YOUR DETECTOR
                    }

                }
            MultiPartVrtx: {
                Profile: true
                Verbosity: 0
                LArMCTruthProducer: "generator" # input MCTruth producer from the reco_file.root given as input
                OutParticleLabel: "mpv"         # output label for the multiparticle vertex
                Origin: 0
                }
            MultiPartRain:   {
                Profile: true
                Verbosity: 2
                LArMCTruthProducer: "rain"      # input MCTruth producer from the reco_file.root given as input
                OutParticleLabel: "mpr"         # output label for the multiparticle rain
                Origin: 0
                }
            SuperaBBoxInteraction: {
                Verbosity: 2
                Profile: true
                LArMCTruthProducer: "generator"  # input MCTruth producer from the reco_file.root given as input
                LArSimEnergyDepositLiteProducer: "sedlite"
                UseSEDLite: true
                Origin: 0
                Cluster3DLabels: ["mcst","pcluster","sed","masked_true2reco"]
                Tensor3DLabels:  ["reco","pcluster_index","masked_true"]
                BBoxSize: [1843.2,1843.2,1843.2] # Covers the whole detector with the smallest possible cube -> yields 6144 = 1024*6 px
                BBoxBottom: [-412.89,-181.86,-894.951] # geometry from icarus_complete_20210527_no_overburden.gdml taking readout window into account
                UseFixedBBox: true
                VoxelSize: [0.3,0.3,0.3]        # Wire pitch in cm of the detector
                CryostatList: [0,0,0,0,1,1,1,1]
                TPCList: [0,1,2,3,0,1,2,3]
                }
            SuperaSimEnergyDeposit: {
                Verbosity: 1
                Profile: true
                LArSimEnergyDepositLiteProducer: "sedlite"
                LArMCShowerProducer: "mcreco"
                UseSEDLite: true
                ParticleProducer: "pcluster"
                OutCluster3DLabel: "sed"
                StoreLength: false
                StoreCharge: false
                StorePhoton: false
                StoreDiffTime: false
                StoreAbsTime: true
                StoreDEDX: false
                TPCList: [0,1,2,3,0,1,2,3]
                CryostatList: [0,0,0,0,1,1,1,1]
                }

            ParticleCorrector: {
                Verbosity: 2
                Profile: true
                Cluster3DProducer: "pcluster_highE"
                ParticleProducer:  "pcluster"
                OutputProducer:    "corrected"
                VoxelMinValue:     -1000
            }


            Tensor3DFromCluster3D: {
                Verbosity: 2
                Profile: true
                Cluster3DProducerList: ["pcluster","sed"]
                OutputProducerList:    ["pcluster","sed"]
                PITypeList:  [1,1]
                FixedPIList: [0.,0.]
                }

            SuperaSpacePoint: {
                Profile: true
                Verbosity: 2
                SpacePointProducers: ["cluster3DCryoE"]
                OutputLabel:        "reco"
                #DropOutput: ["hit_charge","hit_amp"]
                StoreWireInfo: true
                RecoChargeRange: [-1000, 50000]
                }

            }
        }
    ```
</details>

------------------------------------------------------------------------------------
Once you start configuring your own `fcl` files, make sure to check the following parameters and modules:

## 1. Verbosity
Verbosity can be set for any module used in `Supera` by setting `Verbosity : n`

* `n = 0` shows `LARCV_DEBUG` and above messages
* `n = 1` shows `LARCV_INFO` and above messages
* `n = 2` shows `LARCV_NORMAL` and above messages
* `n = 3` shows `LARCV_WARNING` and above messages
* `n = 4` shows `LARCV_ERROR` and above messages
* `n = 5` only shows `LARCV_CRITICAL` and messages

## 2. SuperaMCParticleCluster
This class is responsible and **key** for managing Monte Carlo (MC) **particle clusters** within the LArCV framework. The class is authored by `drinkkazu`. You need to be aware of:

### Features
* It provides functions to configure, initialize, process, and finalize instances of the class.
* It allows creation of particles and particle groups.
* It includes methods for analyzing simulated energy deposits and channels.
* It has functions for merging shower touchings, conversions, ionizations, and deltas.
* It includes functions for merging shower family touchings.
* The class also contains private variables and methods to work with particle clusters.
### Dependencies
The class depends on several other modules and classes in the LArCV framework, including:

* `SuperaBase`
* `SuperaTrue2RecoVoxel3D`
* `FMWKInterface`
* `MCParticleList`
* Various data formats from `larcv/core/DataFormat`
* `SuperaMCParticleClusterData`

The class is built upon a factory pattern, with the corresponding `SuperaMCParticleClusterProcessFactory` class provided for creating instances of `SuperaMCParticleCluster`.

# Running Supera for different experiments/geometries

In general make sure that your wire-LArTPC experiment has a folder in `experiments` (for now, `icarus`, `sbnd`, `dune`). These subfolders are used to store the specific configurations that may vary between experiments and so allow us to maintain the core of `Supera` intact. 

As a previous step you need to have access to `larcv2` framework, either you can refer to the version in `cvfms` or you can clone, install and make the version in the `DeepLearnPhysics` github repository. For the latter, you can do:
```
git clone https://github.com/DeepLearnPhysics/larcv2.git
cd larcv2
source configure.sh
make # Just the first time to build the framework
```
In both cases you need you will need to source the `configure.sh` file before running `Supera` so that it configures `larcv2` paths accordingly.

A general configuration (experiment agnostic) to be run before executing `lar` command is:
```
# General configuration
source /cvmfs/XX.opensciencegrid.org/products/XX/setup_XX.sh
source /whatever/your/path/is/larcv2/configure.sh
source localProducts_larsoft*/setup
setup larsoft vXX_YY_ZZ  -q e26:prof 
```

After this you can clone and build `Supera` in your desired folder (e.g. `icaruscode`, `sbndcode`,`dunereco`) as usual. If you manage to build and run: congratulations! 🥳 You can now optimize the workflow for your analysis.

## ICARUS

## MicroBooNE

## SBND

## DUNE & ProtoDUNE

**DUNE does not have CRT as it is not a surface detector. ProtoDUNE does but for now they are not included**

Once you have a compiled `LArSoft` version, you can build `Supera` module in your `dunereco` folder.
```
cd $MRB_SOURCE/dunereco/dunereco
git clone https://github.com/DeepLearnPhysics/Supera.git
# add Supera to CMakeLists.txt of dunereco
cd Supera
source setup.sh dune
# mrbsetenv
# mrb i ## only first time
```

Several problems when building `Supera` for `DUNE` and `ProtoDUNE` appeared because of a version mismatch between the `ROOT` version and the compilers used to compile `LArSoft` and `larcv2`. If you encounter any problem similar to this try to:
```
# Tested for larsoft v10_01_03:
source /cvmfs/dune.opensciencegrid.org/products/dune/setup_dune.sh

setup root v6_28_12 -q e26:p3915:prof # ---> Enforce a specific root version

source srcs/larcv2/configure.sh
source localProducts_larsoft*/setup
setup larsoft vXX_YY_ZZ -q e26:prof 
```

You can cross-check that your versions are compatible as:

```
>> g++ --version
    g++ (GCC) 12.1.0
    Copyright (C) 2022 Free Software Foundation, Inc.
    This is free software; see the source for copying conditions.  There is NO
    warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
>> gcc --version
    gcc (GCC) 12.1.0
    Copyright (C) 2022 Free Software Foundation, Inc.
    This is free software; see the source for copying conditions.  There is NO
    warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
>> root-config --version
    6.28/12
```