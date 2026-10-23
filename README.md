# In-Network Aggregation Placement for Distributed Training

This repository studies **In-Network Aggregation (INA) placement** in distributed machine-learning networks. The goal is to automatically design heuristics that choose which switches should host INA functions, so worker gradients can be merged closer to their sources before reaching parameter servers.

The project is implemented on top of the LLM4AD framework. LLM4AD is used only as the algorithm-design and evolutionary-search backend; the research focus of this repository is the network optimization problem described below.

> **Upstream framework:** [LLM4AD – Large Language Model for Algorithm Design](https://github.com/Optima-CityU/LLM4AD)

## Research problem

In distributed training, workers generate gradient updates and send them to parameter servers (PS). Directly forwarding every worker flow to a central PS can overload upper-layer links and increase job completion time. An INA-capable switch can aggregate gradients belonging to the same training job and forward a smaller combined flow.

The central task is to design a function that assigns a priority score to each candidate switch. A high score should indicate that deploying INA at that switch is likely to:

- reduce network traffic;
- avoid congested links and aggregation bottlenecks;
- shorten the distance or forwarding cost of worker-to-PS traffic; and
- improve the completion time of distributed training jobs.

The evaluator uses the generated heuristic to rank switches and select a specified number of INA deployment locations. Candidate heuristics can use worker/PS placement, path length, traffic volume, gradient size, switch role, and topology capacity.

## Network scenarios

Two topology families are supported:

### Fat-Tree

Candidate switches include ToR/edge, aggregation, and core layers. The heuristic receives the switch layer as additional context and should balance local aggregation opportunities against upstream traffic reduction.

### Spine-Leaf

Candidate switches include ToR/leaf and spine switches. The heuristic evaluates how well each switch collects worker traffic and reduces the load toward parameter servers.

## Optimization variants

| Variant | Purpose | Main files |
| --- | --- | --- |
| INA placement | Rank switches and select INA deployment locations | `llm4ad/task/optimization/fixedina_construct/` |
| Discrete worker assignment | Convert continuous optimization guidance into integer worker-to-aggregation assignments | `llm4ad/task/optimization/fixedina_construct4y/` |

The underlying traffic and capacity models use NetworkX for topology operations and Gurobi for mathematical optimization. The evaluator measures the quality of generated heuristics over multiple network instances and parameter settings.

## LLM4AD search workflow

The project uses the Evolution of Heuristics (EoH) method supplied by LLM4AD:

1. Define the INA placement objective and a function template.
2. Ask an LLM to generate candidate switch-scoring programs.
3. Execute each candidate in the network evaluator.
4. Score candidates according to the resulting network performance.
5. Evolve and combine promising heuristics across generations.
6. Use the best heuristic to select INA switches.

The LLM proposes algorithmic logic, while the evaluator supplies the network-specific objective and reproducible measurements.

## Repository structure

```text
llm4ad/task/optimization/fixedina_construct/    # INA placement evaluator
llm4ad/task/optimization/fixedina_construct4y/  # Discrete assignment variant
llm4ad/method/eoh/                              # EoH search implementation
llm4ad/tools/llm/                               # LLM API interfaces
example/fixedina/run_eoh.py                     # Placement experiment launcher
example/fixedina4y/run_eoh.py                   # Assignment experiment launcher
requirements.txt                                # Python dependencies
```

Generated logs, sampled programs, plots, and experiment results are ignored by Git because they can become large and may contain prompts or sensitive configuration values.

## Installation

Use Python 3.9 or newer and below Python 3.13. A virtual environment or Conda environment is recommended.

```bash
git clone https://github.com/hyntdlemon1900/LLM4AD.git
cd LLM4AD
python -m pip install -r requirements.txt
python -m pip install -e .
```

The optimization model imports `gurobipy`. Install Gurobi and its Python package separately, and configure a valid Gurobi license before running experiments.

## Configure the LLM

Configure an OpenAI-compatible or HTTPS LLM endpoint in your local experiment launcher. Never commit an API key to source code, logs, or this repository. Prefer environment variables or a secret manager:

```bash
export LLM_API_HOST="api.example.com"
export LLM_API_KEY="replace-with-your-key"
export LLM_MODEL="your-model-name"
```

Any API key previously placed in source files, logs, or chat messages should be revoked and replaced.

## Run experiments

From the repository root, after configuring the LLM and Gurobi:

```bash
python example/fixedina/run_eoh.py
```

To run the discrete worker-assignment variant:

```bash
python example/fixedina4y/run_eoh.py
```

The main parameters include:

- `topo_name`: `"FatTree"` or `"SpineLeaf"`;
- `ina_num`: number of switches to select for INA deployment;
- `timeout_seconds`: candidate evaluation timeout; and
- `max_sample_nums`, `max_generations`, and `pop_size`: EoH search controls.

Experiment outputs are local artifacts and should not be committed.

## Development notes

- Treat LLM-generated programs as untrusted input and keep evaluator timeouts enabled.
- Test heuristics on both Fat-Tree and Spine-Leaf cases when applicable.
- Check edge cases such as disconnected nodes, missing paths, zero traffic, and empty candidate sets.
- Do not commit API keys, `.env` files, local model paths, Gurobi license files, or generated experiment data.

## License

See [LICENSE](LICENSE) for the license of this repository and its upstream components.
