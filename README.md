# CRA-DNNBench

Deep neural network experiments for detecting Code-Reuse Attack (CRA)-like execution patterns from encoded instruction traces.

This repository contains Jupyter notebooks used to train, evaluate, and compare neural network architectures on preprocessed binary-execution datasets. The workflow starts with benchmark execution, collects hardware/software execution traces with Linux `perf`, processes the collected `.data` files, builds fixed-length dataset artifacts, then trains DNN models for binary classification.

Current repository contents include `notebooks/`, `results/`, and a minimal README in the public repository.

## Purpose

The goal is to evaluate whether deep learning models can distinguish benign execution traces from CRA-like traces.

The classification task is binary:

- `0`: benign execution sample
- `1`: CRA-like execution sample

Each sample is represented as a fixed-length numeric sequence derived from instruction-level data.

## Repository structure

```text
dnn_cra/
├── notebooks/
│   ├── 1BLSTM.ipynb
│   ├── 2Dense.ipynb
│   ├── 3CNN_LSTM.ipynb
│   ├── 4BGRU.ipynb
│   ├── 5GNN.ipynb
│   ├── 6GCN_GRU.ipynb
│   ├── 7CNN.ipynb
│   └── dnn_run_all.ipynb
├── scripts/
│   └── data_collection_and_stage0.py
├── results/
│   └── results_summary_model.csv
└── README.md
```

## Complete pipeline

```text
Benchmark selection
        |
        v
Data collection with perf record
        |
        v
perf script trace extraction
        |
        v
Stage 0 dataset artifact builder
        |
        v
shared_artifacts/<software>/
        |
        +-- DNN sequence artifact
        +-- time-series artifact
        +-- tabular artifact
        +-- deterministic split indices
        |
        v
Model notebook
        |
        +-- load encoded samples
        +-- load fixed train/validation/test split
        +-- reshape data according to the model
        +-- scale training data
        +-- train the neural model
        +-- evaluate the test set
        +-- save checkpoint and metrics
        |
        v
results/results_summary_model.csv
```

## Step 1: Data collection

Before Stage 0, execute the benchmark workload selected for the experiment. The workload may come from MiBench, CoreMark-PRO, or another benchmark suite that produces a native executable.

The executable should be launched under Linux `perf record`. The generic command is:

```bash
SAMPLE_PERIOD=5000
exe_base="./benchmark_executable"
data="./perf_data/benchmark_executable.data"

perf record \
  -e instructions,branch-instructions,branch-misses,armv7_cortex_a7/branch-loads/,armv7_cortex_a7/branch-load-misses/ \
  -c "$SAMPLE_PERIOD" \
  -o "$data" \
  -- "$exe_base"
```

Use `sudo` when the system requires privileged access to performance counters:

```bash
sudo perf record \
  -e instructions,branch-instructions,branch-misses,armv7_cortex_a7/branch-loads/,armv7_cortex_a7/branch-load-misses/ \
  -c "$SAMPLE_PERIOD" \
  -o "$data" \
  -- "$exe_base"
```

### Notes about perf events

The events below are suitable for an ARMv7 Cortex-A7 target:

```text
instructions
branch-instructions
branch-misses
armv7_cortex_a7/branch-loads/
armv7_cortex_a7/branch-load-misses/
```

For another processor, replace the architecture-specific events with events available on that platform. Check available events with:

```bash
perf list
```

## Step 2: Process collected `.data` files

After `perf record`, process each `.data` file with `perf script`.

Generic example:

```bash
DATAFILES=(./perf_data/*.data)

for data in "${DATAFILES[@]}"; do
  workdir="$(dirname "$data")"
  base="$(basename "$data" .data)"

  trace_out="$workdir/trace.txt"
  itrace_out="$workdir/itrace.txt"

  echo ">>> Processing: $data"

  perf script \
    -i "$data" \
    -F +time,+event,+period,+ip,+sym,+dso \
    > "$trace_out"

  grep -E 'branch-(instructions|misses)|armv7_cortex_a7/branch-(loads|load-misses)/' \
    "$trace_out" \
    > "$itrace_out"
done
```

Optional commands for richer instruction-flow extraction:

```bash
sudo perf script --itrace=ibcr -F +insn,+disasm,+insnlen,+sym,+dso -i "$data" > "$trace_out"
sudo perf script --itrace=b -F +ip,+sym,+flags -i "$data" > "$itrace_out"
perf script --itrace=i -i "$data" > "$trace_out"
perf script --itrace=yb -i "$data" > "$itrace_out"
```

The basic output files are:

```text
trace.txt   # textual perf samples
itrace.txt  # branch-related events extracted from trace.txt
```

## Step 3: Stage 0 dataset artifact builder

The Stage 0 script converts block-chain pickle files into `.npz` artifacts used by the DNN notebooks.

The script added to this repository is:

```text
scripts/data_collection_and_stage0.py
```

It supports three modes:

```text
--collect          run perf record for one benchmark executable
--process-perf     run perf script on one collected .data file
--build-artifacts  build .npz dataset artifacts from pickle files
```

### Stage 0 input files

For each workload name `<software>`, the script expects:

```text
blocksdata<software>.pickle
s1<software>.pickle
s2<software>.pickle
```

Meaning:

```text
blocksdata<software>.pickle  # block_id -> opcode strings containing hex bytes
s1<software>.pickle          # benign block chains
s2<software>.pickle          # CRA-like block chains
```

Example for `sha-test`:

```text
dado_pickle/
├── blocksdatasha-test.pickle
├── s1sha-test.pickle
└── s2sha-test.pickle
```

### Run data collection through the Python script

```bash
sudo python scripts/data_collection_and_stage0.py \
  --collect \
  --exe ./sha-test \
  --perf-out ./perf_data/sha-test.data \
  --sample-period 5000
```

Dry run:

```bash
python scripts/data_collection_and_stage0.py \
  --collect \
  --exe ./sha-test \
  --perf-out ./perf_data/sha-test.data \
  --sample-period 5000 \
  --dry-run
```

### Process one perf `.data` file

```bash
python scripts/data_collection_and_stage0.py \
  --process-perf \
  --perf-data ./perf_data/sha-test.data
```

This generates:

```text
./perf_data/trace.txt
./perf_data/itrace.txt
```

### Build Stage 0 artifacts

```bash
python scripts/data_collection_and_stage0.py \
  --build-artifacts \
  --software sha-test \
  --pickle-dir ./dado_pickle \
  --out-root ./shared_artifacts \
  --T 30 \
  --pad 256 \
  --seed 42
```

### Stage 0 algorithm

The Stage 0 artifact builder performs these operations:

1. Load `blocksdata<software>.pickle`, `s1<software>.pickle`, and `s2<software>.pickle`.
2. Extract hexadecimal byte tokens from block opcode strings containing `hex `.
3. Convert benign chains from `s1` into label `0` samples.
4. Convert CRA-like chains from `s2` into label `1` samples.
5. Remove sequences that appear in the benign set and in the CRA-like set.
6. Remove duplicated sequences inside each class.
7. Remove sequences composed only of unknown bytes, represented as `??`.
8. Convert hexadecimal bytes to integers in `[0, 255]`.
9. Pad or truncate each sequence to a fixed length `T`.
10. Use `PAD=256` as the sentinel value outside the byte domain.
11. Generate an attention mask and real sequence lengths.
12. Generate a normalized time-series view.
13. Generate tabular features with 7 metadata values and a 256-bin byte histogram.
14. Generate deterministic stratified train/validation/test splits.
15. Save all artifacts under `shared_artifacts/<software>/`.

### Stage 0 output files

For workload `<software>`, the script writes the main Stage 0 files and compatibility aliases used by the current notebooks:

```text
shared_artifacts/<software>/
├── Xy_dnn_pad256_T30_<software>.npz
├── Xy_ts_pad256_T30_<software>.npz
├── Xy_tab_features_<software>.npz
├── indices_seed42_<software>.npz
├── X_hex_y_pad256.npz
└── indices_pad256_seed42.npz
```

The last two files are compatibility aliases. They allow the existing notebooks to keep loading `X_hex_y_pad256.npz` and `indices_pad256_seed42.npz` without code changes, as long as `PAD=256` and `SEED=42`.

### DNN artifact

```text
Xy_dnn_pad256_T30_<software>.npz
```

Stored arrays:

```text
X          # integer sequence tensor, shape [N, T]
y          # labels, shape [N]
pad_val    # padding sentinel
T          # sequence length
lengths    # number of valid tokens per sample
attn_mask  # valid-token mask, shape [N, T]
```

### Time-series artifact

```text
Xy_ts_pad256_T30_<software>.npz
```

Stored arrays:

```text
X        # normalized sequence tensor, shape [N, T, 1]
y        # labels, shape [N]
pad_val  # padding sentinel
T        # sequence length
```

### Tabular artifact

```text
Xy_tab_features_<software>.npz
```

Stored arrays:

```text
X           # tabular feature matrix, shape [N, 263]
y           # labels, shape [N]
feat_names  # feature names
```

The 263 tabular features are:

```text
len_real
frac_pad
mean
std
min
max
entropy
hist_000 ... hist_255
```

### Split artifact

```text
indices_seed42_<software>.npz
```

Stored arrays:

```text
train
val
test
```

The split is stratified by class label and reproducible with seed `42`.

## Notebook compatibility

The current notebooks commonly expect:

```text
shared_artifacts/<software>/X_hex_y_pad256.npz
shared_artifacts/<software>/indices_pad256_seed42.npz
```

The Stage 0 script saves these files automatically when using the default `PAD=256` and `SEED=42`. Therefore, the notebooks can keep this loading pattern:

```python
bundle = np.load(SHARED_ARTIFACTS_ROOT / software / "X_hex_y_pad256.npz", allow_pickle=True)
idx = np.load(SHARED_ARTIFACTS_ROOT / software / "indices_pad256_seed42.npz", allow_pickle=True)

X_all = bundle["X_hex"]
y_all = bundle["y"].astype(np.int64)
idx_train = idx["train"]
idx_val = idx["val"]
idx_test = idx["test"]
```

The script also saves newer explicit filenames, such as `Xy_dnn_pad256_T30_<software>.npz` and `indices_seed42_<software>.npz`, for future scripts that prefer workload-specific artifact names.

## Model notebooks

The repository currently includes these model notebooks:

| Notebook | Model family | Purpose |
|---|---|---|
| `1BLSTM.ipynb` | Bidirectional LSTM | Sequence modelling over encoded traces |
| `2Dense.ipynb` | Dense neural network | Fully connected baseline over flattened features |
| `3CNN_LSTM.ipynb` | CNN-LSTM | Local feature extraction followed by sequence modelling |
| `4BGRU.ipynb` | Bidirectional GRU | Recurrent sequence modelling with GRU units |
| `5GNN.ipynb` | Graph neural network | Graph-oriented representation experiment |
| `6GCN_GRU.ipynb` | GCN-GRU | Graph convolution with recurrent modelling |
| `7CNN.ipynb` | 1D CNN | Convolutional baseline over encoded sequences |

## CNN notebook details

The CNN notebook defines a compact 1D convolutional classifier:

```python
class CRAdetectorQuant(nn.Module):
    def __init__(self, seq_len: int, num_classes: int = 2):
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv1d(1, 64, kernel_size=3, padding=1, bias=False),
            nn.ReLU(),
            nn.Conv1d(64, 128, kernel_size=3, padding=1, bias=False),
            nn.ReLU(),
            nn.Conv1d(128, 128, kernel_size=3, padding=1, bias=False),
            nn.ReLU(),
            nn.MaxPool1d(kernel_size=seq_len)
        )
        self.classifier = nn.Linear(128, num_classes, bias=False)

    def forward(self, x):
        if x.dtype != torch.float32:
            x = x.float()
        x = x.unsqueeze(1)
        x = self.features(x)
        x = x.squeeze(-1)
        return self.classifier(x)
```

Typical training configuration:

```python
criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=0.001, weight_decay=1e-4)
num_epochs = 100
batch_size = 32
seed = 42
```

Device selection normally follows this order:

```text
cuda -> mps -> cpu
```

## Batch execution

The notebook `dnn_run_all.ipynb` executes selected model notebooks across several workloads.

Default workloads may include:

```python
datasets = [
    "cjpeg-rose7-preset.exe",
    "core.exe",
    "linear_alg-mid-100x100-sp.exe",
    "loops-all-mid-10k-sp.exe",
    "nnet_test.exe",
    "parser-125k.exe",
    "radix2-big-64k.exe",
    "sha-test.exe",
    "zip-test.exe",
]
```

The batch manager injects execution metadata into each notebook:

```text
NOTEBOOK_NAME
software
BATCH_ID
EXECUTION_NUMBER
RUN_ID
PROJECT_ROOT
RESULTS_ROOT
RUN_DIR
DATASET_ID
DATASET_MANIFEST_PATH
```

## Output files

The batch manager writes results under:

```text
results/batches/<batch_id>/
├── batch_config.json
├── batch_environment.json
├── execution_manifest.csv
├── execution_status.csv
├── batch_results_summary.csv
├── executed_notebooks/
├── logs/
├── errors/
└── runs/
```

The global summary file is:

```text
results/results_summary_model.csv
```

## Recorded metrics

The summary CSV may contain:

```text
accuracy
precision
recall
f1_score
loss
roc_auc
pr_auc
tn
fp
fn
tp
fpr
fnr
n_params
trainable_params
model_size_MB
throughput_samples_per_sec
gpu_peak_MB
n_test_samples
checkpoint_path
run_dir
```

These fields support comparison across workloads and model families.

## Installation

Create a Python environment:

```bash
python -m venv .venv
```

Activate it on Windows:

```bash
.venv\Scripts\activate
```

Activate it on Linux/macOS:

```bash
source .venv/bin/activate
```

Install the main dependencies:

```bash
pip install numpy pandas scikit-learn matplotlib psutil torch nbformat nbconvert jupyter
```

For Stage 0 only:

```bash
pip install numpy scikit-learn
```

Some graph-oriented notebooks may require extra packages.

## How to run one model

Start Jupyter:

```bash
jupyter notebook
```

Open one model notebook, for example:

```text
notebooks/7CNN.ipynb
```

Set the workload name:

```python
software = "sha-test"
```

Check the project paths:

```python
PROJECT_ROOT = Path(r"C:\CNSM_2026")
SHARED_ARTIFACTS_ROOT = PROJECT_ROOT / "shared_artifacts"
```

Run all notebook cells.

## How to run the batch manager

Open:

```text
notebooks/dnn_run_all.ipynb
```

Select model notebooks:

```python
notebooks = [
    "6GCN_GRU.ipynb",
    "7CNN.ipynb",
]
```

Select workloads:

```python
datasets = [
    "cjpeg-rose7-preset.exe",
    "core.exe",
    "sha-test.exe",
]
```

Run all cells.

After execution, inspect:

```text
results/batches/<batch_id>/execution_status.csv
results/batches/<batch_id>/batch_results_summary.csv
results/results_summary_model.csv
```

## Reproducibility notes

The Stage 0 script uses seed `42` by default for Python and NumPy.

The split file `indices_seed42_<software>.npz` ensures that every model can use the same train/validation/test split for each workload.

The batch manager stores metadata for each execution, including run ID, notebook name, workload name, environment information, logs, and executed notebooks.

## Current limitations

The repository does not include raw benchmark traces.

The repository does not include generated `shared_artifacts/` folders.

The default notebook paths may need changes for Linux or macOS.

Architecture-specific `perf` events must be adjusted when the target CPU is not ARMv7 Cortex-A7.

Some notebooks may require dependencies that are absent from a dedicated `requirements.txt` file.

## Recommended future improvements

Add a `requirements.txt` file.

Add a small toy dataset for installation tests.

Add a command-line runner for all model notebooks.

Add a dataset manifest for each workload.

Document the exact raw trace format used before block-chain pickle generation.

Add model-specific architecture summaries.

## Citation

If this repository is used in academic work, cite the related paper, thesis, or technical report associated with this CRA detection pipeline.
