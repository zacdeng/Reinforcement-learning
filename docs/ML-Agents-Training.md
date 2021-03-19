

# ML-Agentsc插件和Unity3D使用总结(一)



## Training & Inference

#### Start Training

主要通过 `mlagents-learn` 插件来进行网络的训练，使用一个YAML文件来存储训练中所用的所有超参数（contains all the configurations and hyperparameters to be used during training）

To view a description of all the CLI（commend-line interface，命令行界面） options accepted by `mlagents-learn`, use the `--help`:

```
mlagents-learn --help
```

The basic command for training is:

```shell
mlagents-learn <trainer-config-file> --env=<env_name> --run-id=<run-identifier>
```

- `<trainer-config-file>` is the file path of the trainer configuration YAML. **Details**: [Training Configurations](https://github.com/Unity-Technologies/ml-agents/blob/main/docs/Training-ML-Agents.md#training-configurations) 
- `<env_name>`**(Optional)** is the name (including path) of your [Unity executable](https://github.com/Unity-Technologies/ml-agents/blob/main/docs/Learning-Environment-Executable.md) containing the agents to be trained. 
- `<run-identifier>` is a unique name you can use to identify the results of your training runs.

See the [Getting Started Guide](https://github.com/Unity-Technologies/ml-agents/blob/main/docs/Getting-Started.md#training-a-new-model-with-reinforcement-learning) for a sample execution of the `mlagents-learn` command.

#### Observing Training

不管使用PPO还是SAC算法，最终都会生成三个文件在`results/<run-identifier>` folder目录下，分别包括：

- Summaries：生成训练指标（training metrics）可以使用TensorBoard来可视化数据

```shell
tensorboard --logdir results --port 6006
```

用web输入`localhost:6006`查看TensorBoard分析出的数据，其中最重要的是`Environment/Cumulative Reward`指标，其会随着训练不断增加逼近100（100是agents能积累的最大值）

具体参数参考[Using TensorBoard to Observe Training](https://github.com/Unity-Technologies/ml-agents/blob/main/docs/Using-Tensorboard.md)

- Models：these contain the **model checkpoints** that are updated throughout training and the **final model file** (`.onnx`). 
- Timers file (under `results/<run-identifier>/run_logs`)

#### Stopping and Resuming Training

To interrupt training and save the current progress, hit `Ctrl+C` once and wait for the model(s) to be saved out

To resume a previously interrupted or completed training run, use the `--resume` flag and make sure to specify the previously used run ID.

```shell
mlagents-learn <trainer-config-file> --env=<env_name> --run-id=<run-identifier> --resume
```

If you would like to re-run a previously interrupted or completed training run and re-use the same run ID (in this case, overwriting the previously generated artifacts), then use the `--force` flag.

```shell
mlagents-learn <trainer-config-file> --env=<env_name> --run-id=<run-identifier> --force
```



---



## Training Configurations

#### 训练参数调整

We provide sample configuration files for our example environments in the [config/](https://github.com/Unity-Technologies/ml-agents/blob/main/config) directory. The `config/ppo/3DBall.yaml` was used to train the 3D Balance Ball in the [Getting Started](https://github.com/Unity-Technologies/ml-agents/blob/main/docs/Getting-Started.md) guide. That configuration file uses the **PPO** trainer, but we also have configuration files for **SAC** and **GAIL**.

[Configurations Files](https://github.com/Unity-Technologies/ml-agents/blob/main/docs/Training-Configuration-File.md)

#### 添加CLI参数至YAML文件中

即把终端的命令直接放到YAML文件中执行，Reminder that a detailed description of all the CLI arguments can be found by using the help utility:

```shell
mlagents-learn --help
```

一些示例：

##### Environment settings

```yaml
env_settings:
  env_path: FoodCollector
  env_args: null
  base_port: 5005  # ip port
  num_envs: 1  #训练数量
  seed: -1
```

##### Engine settings

```yaml
engine_settings:
  width: 84
  height: 84
  quality_level: 5
  time_scale: 20
  target_frame_rate: -1
  capture_frame_rate: 60  #应该是60fps为1步
  no_graphics: false
```

##### Checkpoint settings

```yaml
checkpoint_settings:
  run_id: foodtorch
  initialize_from: null
  load_model: false
  resume: false
  force: true
  train_model: false
  inference: false
```

##### Torch settings

```yaml
torch_settings:
  device: cpu  #torch.device
```

#### Behavior Configurations

The primary section of the trainer config file is a set of configurations for each Behavior in your scene.  PPO sample is as follow：

```yaml
behaviors:
  BehaviorPPO:
    trainer_type: ppo

    hyperparameters:
      # Hyperparameters common to PPO and SAC
      batch_size: 1024
      buffer_size: 10240
      learning_rate: 3.0e-4
      learning_rate_schedule: linear

      # PPO-specific hyperparameters
      # Replaces the "PPO-specific hyperparameters" section above
      beta: 5.0e-3
      epsilon: 0.2
      lambd: 0.95
      num_epoch: 3

    # Configuration of the neural network (common to PPO/SAC)
    network_settings:
      vis_encode_type: simple
      normalize: false
      hidden_units: 128
      num_layers: 2
      # memory
      memory:
        sequence_length: 64
        memory_size: 256

    # Trainer configurations common to all trainers
    max_steps: 5.0e5
    time_horizon: 64
    summary_freq: 10000
    keep_checkpoints: 5
    checkpoint_interval: 50000
    threaded: true
    init_path: null

    # behavior cloning
    behavioral_cloning:
      demo_path: Project/Assets/ML-Agents/Examples/Pyramids/Demos/ExpertPyramid.demo
      strength: 0.5
      steps: 150000
      batch_size: 512
      num_epoch: 3
      samples_per_update: 0

    reward_signals:
      # environment reward (default)
      extrinsic:
        strength: 1.0
        gamma: 0.99

      # curiosity module
      curiosity:
        strength: 0.02
        gamma: 0.99
        encoding_size: 256
        learning_rate: 3.0e-4

      # GAIL
      gail:
        strength: 0.01
        gamma: 0.99
        encoding_size: 128
        demo_path: Project/Assets/ML-Agents/Examples/Pyramids/Demos/ExpertPyramid.demo
        learning_rate: 3.0e-4
        use_actions: false
        use_vail: false

    # self-play
    self_play:
      window: 10
      play_against_latest_model_ratio: 0.5
      save_steps: 50000
      swap_steps: 2000
      team_change: 100000
```

以上是一些简单的参数调整，如有需要参考[Training-ML-Agents.md](https://github.com/Unity-Technologies/ml-agents/blob/main/docs/Training-ML-Agents.md)



---



## Python API

记得下手册！！！！！！ 