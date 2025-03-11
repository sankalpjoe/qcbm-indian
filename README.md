# Quantum Circuit Born Machine for Financial Data Generation

This repository implements a Quantum Circuit Born Machine (QCBM) to generate synthetic financial market data. The model learns the statistical distribution of stock market returns and can generate new samples that have similar properties to the original dataset.

## Overview

The QCBM uses a parameterized quantum circuit trained via a genetic algorithm to learn the distribution of financial returns data. Once trained, the circuit can be sampled to produce new synthetic market data with statistical properties resembling the original dataset.

## Requirements

- Python 3.7+
- NumPy
- Matplotlib
- Amazon Braket SDK (`amazon-braket-sdk`)
- StatsModels
- Pandas

Install dependencies:

```bash
pip install amazon-braket-sdk numpy matplotlib pandas statsmodels
```

## How It Works

The implementation consists of several key components:

### 1. Data Preparation

The code loads financial return data (in this case, the BSE100 index) and prepares it for encoding into quantum circuits:

```python
# Load data
df = pd.read_csv('bse100.csv', header=1)
D = df[['date', 'BSE100']]  # Use BSE100 column for data

# Find data range for scaling
DAX_min = D['BSE100'].min()
DAX_max = D['BSE100'].max()
```

### 2. Quantum Circuit Design

The model uses a 12-qubit circuit with 7 layers of parameterized gates:

```python
qcbm = Circuit()
wires = 12  # 12 qubits for 12-bit precision
depth = 7   # 7 layers of gates

# Setup parameterized gates
theta = [[FreeParameter("t_%s_%s" % (l,q)) for q in range(wires)] for l in range(depth)]

# Build circuit with Rx, Rz rotations and CNOT entangling gates
for q in range(wires):
    qcbm.rx(q, theta[0][q])
    qcbm.rz(q, theta[1][q])
# ... (additional circuit construction)
```

### 3. Genetic Algorithm Training

The circuit is trained using a genetic algorithm that:
- Initializes a population of random parameter sets
- Evaluates each parameter set by sampling the circuit and comparing to real data
- Selects the best performing parameters
- Creates new parameter sets through mutation
- Repeats for multiple generations

```python
# Genetic algorithm configuration
L = 100  # number of generations
M = 25   # number of best solutions kept each generation
D = 40   # number of offspring per solution
alpha = 1.0  # initial mutation rate (decreases over time)

# Training loop
for gen in range(L):
    alpha *= np.exp(-beta)  # Decrease mutation rate
    Mthetas = Nthetas_sorted[:M]  # Keep best M parameter sets
    Nthetas_new = mutation(Mthetas, D, alpha)  # Create mutations
    cost_new, thetas_new = thetas_by_cost(Nthetas_new, data, DAX_min, DAX_max)
    # ... (update parameters based on cost)
```

### 4. Data Generation

After training, the optimized circuit can generate synthetic data:

```python
thetas_opt = Nthetas_sorted[0]  # Best parameters from training
thetas_dict = thetas_to_dict(thetas_opt)
task = device.run(qcbm, shots=K, inputs=thetas_dict)  # Run quantum circuit
result = task.result()
counts = result.measurement_counts  # Get measurement results

# Convert binary measurement results to continuous values
samples = []
for bitstring, count in counts.items():
    value = bitstring_to_data(bitstring, DAX_min, DAX_max)
    samples.extend([value] * count)
```

### 5. Evaluation

The quality of the generated data is evaluated using QQ plots to compare distributions:

```python
pp_x = sm.ProbPlot(data)  # Original data
pp_y = sm.ProbPlot(samples)  # Generated data
qqplot_2samples(pp_x, pp_y, xlabel='Original', ylabel='QCBM samples')
```

## Running the Code

1. **Local Simulation**:

```python
device = LocalSimulator()
# ... run training or sampling code
```

2. **Using Quantum Hardware** (requires AWS account with Braket access):

```python
device = AwsDevice("arn:aws:braket:us-west-1::device/qpu/rigetti/Aspen-M-3")
# ... run circuit on real quantum hardware
```

## Key Components Explained

### Encoding/Decoding

- `data_to_bitstring`: Converts continuous financial data to binary representations
- `bitstring_to_data`: Converts binary outputs back to continuous values

### Circuit Sampling

- `run_qcbm`: Executes the quantum circuit with given parameters and converts results to data samples

### Genetic Algorithm Components

- `random_thetas`: Generates random circuit parameters
- `mutation`: Creates modified versions of parameter sets
- `thetas_by_cost`: Evaluates parameter sets and sorts by cost function value

## References

This implementation is based on the algorithm presented in:
- Quantum Machine Learning and Optimisation in Finance by A. Jacquier and O. Kondratyev (2022, Packt Publishing)
