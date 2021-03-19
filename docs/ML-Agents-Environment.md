# ML-Agentsc插件和Unity3D使用总结(二)

## Agents

When you create an Agent, you should usually extend the **base Agent** class. This includes implementing the following methods:

- `Agent.OnEpisodeBegin()` —— 在每个episode开始时执行，包括最初的仿真初始化
- `Agent.CollectObservations(VectorSensor sensor)` —— 每步Agent需要决策时调用，收集环境信息（env）
- `Agent.OnActionReceived()` —— 每次Agent收到动作并执行动作时调用，常在该class里定义reward
- `Agent.Heuristic()`—— 当`Behavior Type`被设成`Heuristic Only`时，系统会使用`Heuristic()`函数生成Agent执行的动作

As a concrete **example**, here is how the **Ball3DAgent** class implements these methods:

- `Agent.OnEpisodeBegin()` — Resets the agent cube and ball to their starting positions. The function randomizes the reset values so that the training generalizes to more than a specific starting position and agent cube orientation.
- `Agent.CollectObservations(VectorSensor sensor)` — Adds information about the orientation of the agent cube, the ball velocity, and the relative position between the ball and the cube. Since the  `CollectObservations()` method calls `VectorSensor.AddObservation()` such that vector size adds up to 8, the Behavior Parameters of the Agent are set with vector observation space with a state size of 8.
- `Agent.OnActionReceived()` — The action results in a small change in the agent cube's rotation at each step. In this example, an Agent receives a small positive reward for each step it keeps the ball on the agent cube's head and a larger, negative reward for dropping the ball. An Agent's episode is also ended when it drops the ball so that it will reset with a new ball for the next simulation step.
- `Agent.Heuristic()` - Converts the keyboard inputs into actions.

## Decisions

每当Agent请求决策时`observation-decision-action-reward`循环将会重复，当`Agent.RequestDecision()`被called时Agent会请求决策，如果需要Agent以特定周期循环发出决策请求，在 Agent's GameObject里增加一个`Decision Requester`：

Making decisions at regular step intervals is generally most appropriate for physics-based simulations：

- For example, an agent in a **robotic simulator** that must provide fine-control of **joint torques** should make its decisions every step of the simulation. 
- On the other hand, an agent that only needs to make decisions **when certain game or simulation events occur**, such as in a **turn-based game**, should call `Agent.RequestDecision()` manually.

## Observations and Sensors

In order for an agent to learn, the observations should include all the information an agent needs to accomplish its task. 

***consider what you would need to calculate an analytical solution to the problem, or what you would expect a human to be able to use to solve the problem***

### Generating Observations

ML-Agents provides multiple ways for an Agent to make observations:

1. Overriding the `Agent.CollectObservations()` method and passing the observations to the provided `VectorSensor`.
2. Adding the `[Observable]` attribute to fields and properties on the Agent.
3. Implementing the `ISensor` interface, using a `SensorComponent` attached to the Agent to create the `ISensor`.

#### Agent.CollectObservations()

- `Agent.CollectObservations()` is best used for aspects of the environment which are numerical and non-visual.

- Your implementation of this function must call `VectorSensor.AddObservation` to add vector observations.

- Data type：`Integers `, `booleans`, `Vector2`, `Vector3`, and `Quaternion`.

**3DBall example：**

```c#
public GameObject ball;

public override void CollectObservations(VectorSensor sensor)
{
    // Orientation of the cube (2 floats)
    sensor.AddObservation(gameObject.transform.rotation.z);
    sensor.AddObservation(gameObject.transform.rotation.x);
    // Relative position of the ball to the cube (3 floats)
    sensor.AddObservation(ball.transform.position - gameObject.transform.position);
    // Velocity of the ball (3 floats)
    sensor.AddObservation(m_BallRb.velocity);
    // 8 floats total
}
```

The observations passed to `VectorSensor.AddObservation()` must always contain the **same number of elements** must always be **in the same order**. If the number of observed entities in an environment can **vary** you can pad the calls with **zeros for any missing entities** in a specific observation, or you can **limit an agent's observations to a fixed subset**. For example, instead of observing every enemy in an environment, you could only observe the closest five.

#### Observable Fields and Properties

Another approach is to **define the relevant observations as fields or properties** on your Agent class, and annotate them with an `ObservableAttribute`. For example, in the Ball3DHardAgent, the difference between positions could be observed by adding a property to the Agent:

```c#
using Unity.MLAgents.Sensors.Reflection;

public class Ball3DHardAgent : Agent {

    [Observable(numStackedObservations: 9)]
    Vector3 PositionDelta
    {
        get
        {
            return ball.transform.position - gameObject.transform.position;
        }
    }
}
```

The behavior of `ObservableAttribute`s are controlled by the "Observable Attribute Handling" in the Agent's `Behavior Parameters`. The possible values for this are:

- **Ignore** (default) - All ObservableAttributes on the Agent will be ignored. If there are no ObservableAttributes on the Agent, this will result in the fastest  initialization time.
- **Exclude Inherited** - Only members on the declared class will be examined; members that are inherited are ignored. This is a reasonable tradeoff between performance and flexibility.
- **Examine All** All members on the class will be examined. This can lead to slower startup times.

#### ISensor interface and SensorComponents

The `ISensor` interface is generally intended for advanced users.  毕设项目暂时用前两个就可以