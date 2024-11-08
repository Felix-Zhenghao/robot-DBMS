# DOC VERSION: 1.0

> The first two part should be done when INSERT a new task and subtask. Convenient interfaces bridging the relational database and simulation framworks like MimicGen should be provided to do that. It may be faster to rewrite the simulation framework so that interfaces are more intuitive to connect to DB.

This document describes what may be done to use the data generation code of MimicGen, and connect the robotDB with the simulation framework.

# Use the original environment launch code
In mimicgen, the simulation environment is launched by [this](https://github.com/NVlabs/mimicgen/blob/main/mimicgen/scripts/generate_dataset.py#L230-L243).

The most important stuff is `env_meta`. The other parameter can be easily resolved and queried. Documentation of `env_meta` can be found [here](https://robomimic.github.io/docs/modules/environments.html#environments). My paraphrase:

- `env_meta` is a dictionary like this: `env_meta = {"env_name": str, "type": EnvType, "env_kwargs": dict}`
    - `env_name` is just a string and should be the same as `env_name`.
    - `type` is a enum can be found [here](https://github.com/Felix-Zhenghao/robomimic/blob/master/robomimic/envs/env_base.py#L9-L16).
    - `env_kwargs` is a long dict, example can be found [here](https://robomimic.github.io/docs/modules/environments.html#initialize-an-environment-from-a-dataset).

Now that we have the metadata of the environment. But we actually need to put robot and objects into the environment. This is pretty complicated. The inheritation path is for a task `Square_D0` is, for instance, `robosuite.environments.base.MujocoEnv` -> `robosuite.environments.robot_env.RobotEnv` -> `robosuite.environments.manipulation.manipulation_env.ManipulationEnv` -> `robosuite.environments.manipulation.single_arm_env.SingleArmEnv` -> `mimicgen.envs.robosuite.single_arm_env_mg.SingleArmEnv_MG` -> `mimicgen.envs.robosuite.nut_assembly.NutAssembly_D0 & Square_D0`. In this case, the env_name should be `Square_D0`. The implementation of [`SingleArmEnv_MG`](https://github.com/NVlabs/mimicgen/blob/main/mimicgen/envs/robosuite/single_arm_env_mg.py#L20) and [`Square_D0`](https://github.com/NVlabs/mimicgen/blob/main/mimicgen/envs/robosuite/nut_assembly.py#L62) can be referenced.

This is the most complicated part of connecting robotDB with MimicGen or Robosuite.

# Define subtasks
Can refer to [here](https://mimicgen.github.io/docs/tutorials/datagen_custom.html#generating-data-for-new-simulation-frameworks) to define a subtask of a task once we create the environment of that task.

And example of configuration used by MimicGen data generation is as follows. It is generated by [`generate_core_configs.py`](https://github.com/NVlabs/mimicgen/blob/main/mimicgen/scripts/generate_core_configs.py):
```
{
    "name": "square",
    "type": "robosuite",
    "experiment": {
        "name": "demo_src_square_task_D1",
        "source": {
            "dataset_path": "/Users/chizhenghao/Desktop/low-cost-arm/mimicgen/mimicgen/../datasets/source/square.hdf5",
            "filter_key": null,
            "n": 10,
            "start": null
        },
        "generation": {
            "path": "/tmp/core_datasets/square",
            "guarantee": true,
            "keep_failed": true,
            "num_trials": 1000,
            "select_src_per_subtask": false,
            "transform_first_robot_pose": false,
            "interpolate_from_last_target_pose": true
        },
        "task": {
            "name": "Square_D1",
            "robot": null,
            "gripper": null,
            "interface": null,
            "interface_type": null
        },
        "max_num_failures": 25,
        "render_video": true,
        "num_demo_to_render": 10,
        "num_fail_demo_to_render": 25,
        "log_every_n_attempts": 50,
        "seed": 1
    },
    "obs": {
        "collect_obs": true,
        "camera_names": [
            "agentview",
            "robot0_eye_in_hand"
        ],
        "camera_height": 84,
        "camera_width": 84
    },
    "task": {
        "task_spec": {
            "subtask_1": {
                "object_ref": "square_nut",
                "subtask_term_signal": "grasp",
                "subtask_term_offset_range": [
                    10,
                    20
                ],
                "selection_strategy": "nearest_neighbor_object",
                "selection_strategy_kwargs": {
                    "nn_k": 3
                },
                "action_noise": 0.05,
                "num_interpolation_steps": 5,
                "num_fixed_steps": 0,
                "apply_noise_during_interpolation": false
            },
            "subtask_2": {
                "object_ref": "square_peg",
                "subtask_term_signal": null,
                "subtask_term_offset_range": null,
                "selection_strategy": "nearest_neighbor_object",
                "selection_strategy_kwargs": {
                    "nn_k": 3
                },
                "action_noise": 0.05,
                "num_interpolation_steps": 5,
                "num_fixed_steps": 0,
                "apply_noise_during_interpolation": false
            }
        }
    }
}
```

# Use the original traj transformation code
Now, we want to utilize the original traj transformation code in `DataGenerator.generate()`. We can use the code ***after*** [line 334](https://github.com/NVlabs/mimicgen/blob/main/mimicgen/datagen/data_generator.py#L304) if we have prepared the data needed.

These information are listed [here](https://github.com/NVlabs/mimicgen/blob/main/mimicgen/datagen/data_generator.py#L297-L302). They are: the eef_pose of each timestamp from the source demo; the target pose of each timestamp from the source demo; the gripper_action of each timestamp from the source demo; and the target object position from the source demo.

Once we have this information, we need information of the current environment, that is, the obj_position of the current environment. We can use the code [here](https://github.com/NVlabs/mimicgen/blob/main/mimicgen/datagen/data_generator.py#L318-L322) by giving the `cur_object_pose`.

The data generation code of mimicgen has complicated source demo selection strategy code. A simple alternative is to select random source demo and stick to the same source demo for each subtask. 
