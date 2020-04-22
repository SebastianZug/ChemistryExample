<!--

author:   Sebastian Zug
email:    sebastian.zug@informatik.tu-freiberg.de
version:  0.0.3
language: de
comment:  In dieser Vorlesungen werden die Features von LiaScript mit Blick auf die Systemmodellierung vorgestellt
narrator: Deutsch Female

import: https://raw.githubusercontent.com/LiaTemplates/Rextester/master/README.md

attribute: Danke an Andre Dietrich für seinen Kurs "Einführung Regelungstechnik" aus dem Teile übernommen wurden.

script:   https://cdn.jsdelivr.net/chartist.js/latest/chartist.min.js
          https://d3js.org/d3-random.v2.min.js
          https://d3js.org/d3.v4.min.js
          https://cdn.plot.ly/plotly-latest.min.js

link: https://cdnjs.cloudflare.com/ajax/libs/animate.css/3.7.0/animate.min.css

@eval
@Rextester._eval_(@uid, @Python, , , ,```
    var lines = data.Result.replace(/\n/g, ' ')
                           .replace(/\s{2,}/g, ' ')
                           .match(/(-?\d+\.\d*\s*)+/g)

    var outcome = [];
    for (var i=0; i<lines.length; i++){
      outcome[i] = lines[i].split(' ').map(function(item) {
          return parseFloat(item);
      });
    }
    @input(1);
    Plotly.newPlot(span_id, traces, layout);
    console.log("Aus Maus");
```)
@end
-->

# Example for Macro usage in LiaScript

This presentation illustrates the application of LiaScript Makros for visualizing basic system models. It implements an PID control solution for a PT1 setting.

The use case demonstrates the combination of an "arbitrary" programming language for implementing model behavior and controller as well as JavaScript for visualization purposes.

The source code is available on  the interactive mode can be reached by 

## Concepts

The implementation has to be done in python while the visualization needs to be realized in js. Hence, we need a parser that extracts the actual values and provides the diagrams. The macro `@eval` calls another macro definition `@Rextester._eval_` that coordinates these activities.

<!--
style="width: 60%; min-width: 420px; max-width: 720px;"
-->
```ascii
  +-------------------------------------------------------+
  |  +--------------------------------------+             |
  |  | Rextester Python Implementation      |   @input(0) |
  |  +--------------------------------------+             |
  |                  | print(u, v)                        |
  |                  v                                    |
  |  +--------------------------------------+             |
  |  | JavaScript Snippet for visualization |   @input(1) |
  |  +--------------------------------------+             |
  +-------------------------------------------------------+
  @Rextester._eval_(@uid, @Python, , , , code)
```

The macro `@eval` is contained in the header. `_input(0)_` is called automatically. The first ten lines of the macro code include the parsing and extraction process based on `_input(0)_` output. The next step executes the visualization script by its reference `_input(1)_`. Extract the code by clicking on the corresponding blue bar in the code section. The specific implementation is hidden by default. The last step initiates the actual drawing process.

@@eval

## PT1 Model

The example visualizes the output $y$ of PT1-element in an interval of 50 time steps. It is activated by a binary trigger $u$ at 10 and 30 time units. Test different parameter settings for an evaluation of their effect! Feel free to adapt the parameters.

`System.py` implements the differential equation of a PT1 system:

$$T \cdot \dot{y}(t) + y(t) = K \cdot u(t)$$

by calculating discrete steps according to

$$y_n = T^\star  \cdot (K u_n - y_{n-1}) + y_{n-1}$$

with
$$T^\star = \frac{1}{\frac{T}{\Delta T}+1}$$


```python                          System.py
import numpy as np

# Parameters of the system
T = 0.01           # time constant of the system
deltaT = 0.001     # sample frequency
K = 1              # Final value

# Input values
u = np.zeros(50); u[10:20]=1; u[30:-1]=1
T_star = 1 / ((T / deltaT) + 1)

# Simulation of the system
y = np.zeros(len(u))
for index, entry in enumerate(u):
    if index>0:
        y[index] = T_star*(K*u[index]-y[index-1])+y[index-1]

print(u)
print(y)
```
```js -Visualization.js
var traces = [
  {
    x: d3.range(0, 100),
    y: outcome[0],
    mode: 'lines',
    line: {shape: 'vh'},
    type: 'scatter',
    name: 'Activation u',
  },
  {
    x: d3.range(0, 100),
    y: outcome[1],
    type: 'scatter',
    name: 'System response y',
  }
];

var layout = {
    height : 300,
    width :  650,
    yaxis: {
      range: [0, 1],
      title: {
        text: 'System value',
      },
    },
    xaxis: {
      title: {
        text: 'Samples',
      },
    },
    margin: { l: 60, r: 10, b: 35, t: 10, pad: 4},
    showlegend: true,
    legend: { x: 1, xanchor: 'right', y: 0},
    tracetoggle: false
};
```
@eval


## Control application

A PID controller can be formed simply by adding the individual control elements and the different factors ($K_P$, $K_I$, $K_D$) are the adjusting parameters defining system's behavior. With a value equal to 0 the respective control element can be switched off.

$$ u(t) = \underbrace{K_P \cdot e(t)}_{\text{Propotional part}}
        + \overbrace{K_I \cdot \sum^{t}_{t = 0} e(t) + u(0)}^{\text{Integral part}}
        + \underbrace{K_D \cdot (e(t) - e(t-1))}_{\text{Differential part}}
$$

```python                          System.py
import numpy as np

# Parameters of the system
T = 0.05           # time constant of the system
deltaT = 0.0005     # sample frequency
K = 3              # Final value

T_star = 1 / ((T / deltaT) + 1)

# Control parameters
Kp = 3.0
Ki = 0.0
Kd = 0.0

e_now  = 0   # Regelabweichung e(t)
e_old  = 0   # Regelabweichung e(t-1)
e_sum  = 0

# Simulation parameters
samples = 50  # number of steps
u = np.zeros(samples)
y = np.zeros(samples)
error = np.zeros(samples)
target = 1.5

# Simulation of the system
def nextPlantStep (u, t):
  if t>0:
    y[t] = T_star*(K*u-y[t-1])+y[t-1]

for t in range(1, samples-1):
  nextPlantStep(u[t], t);
  e = target - y[t-1];
  e_old = e_now;
  e_now = e;
  e_sum = e_sum + e_now;

  #Calculation of new control output u
  u[t+1] = Kp * e_now + Ki * e_sum + Kd * (e_now - e_old);
  error[t]= e;
y[-1] = y[-2]

np.set_printoptions(precision=5, suppress=True)
print(np.round(u, 4))
print(np.round(y, 4))
print(np.ones(samples)*target)
```
```js -Visualization.js
var traces = [
  {
    x: d3.range(0, 100),
    y: outcome[0],
    mode: 'lines',
    line: {shape: 'vh'},
    type: 'scatter',
    name: 'Activation u',
  },
  {
    x: d3.range(0, 100),
    y: outcome[1],
    type: 'scatter',
    name: 'System response y',
  },
  {
    x: d3.range(0, 100),
    y: outcome[2],
    type: 'scatter',
    name: 'Target',
  }
];

var layout = {
    height : 300,
    width :  650,
    yaxis: {
      range: [-2, 4],
      title: {
        text: 'System value',
      },
    },
    xaxis: {
      title: {
        text: 'Samples',
      },
    },
    margin: { l: 60, r: 10, b: 35, t: 10, pad: 4},
    showlegend: true,
    legend: { x: 1, xanchor: 'right', y: 0},
    tracetoggle: false
};
```
@eval

Based on this setting we can monitor different reactions:

| $K-p$ | $K_i$ | $K_d$ | Observation                                               |
| ----- | ----- | ----- | --------------------------------------------------------- |
| 3.0   | 0.0   | 0.0   | The output values do not reach the intended target value. |
| 18.0  | 0.0   | 0.0   | The system oscillates.                                      |
| ...   |       |       |                                                           |

