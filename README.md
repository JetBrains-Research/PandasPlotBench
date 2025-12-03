[![JetBrains Research project](https://jb.gg/badges/research.svg)](https://confluence.jetbrains.com/display/ALL/JetBrains+on+GitHub)

# PandasPlotBench

This is the benchmark to assess the capability of models in writing the code for visualizations given the description of the Pandas DataFrame.

🛠️ **Task**. Given the plotting task and the description of a Pandas DataFrame, write the code to build a plot.

The dataset can be found on our [HuggingFace page](https://huggingface.co/datasets/JetBrains-Research/PandasPlotBench). It is based on the [MatPlotLib gallery](https://matplotlib.org/stable/gallery/index.html).

The paper can be found in arXiv: https://arxiv.org/abs/2412.02764v1.

The `paper_supp_info` directory contains the supplementary materials for our paper, as well as a file with a demonstration of data points (tasks and plots).

📩 If you have any questions or requests concerning this dataset, please contact the author at [timur.galimzyanov@jetbrains.com](mailto:timur.galimzyanov@jetbrains.com).

# Install

1. Clone the repo: `git clone https://github.com/JetBrains-Research/PandasPlotBench.git`
2. Navigate to the directory: `cd plotting-benchmark`
3. Run: `poetry install`
   * **Important**: if you're going to use benchmarking on a local machine (includes using `code_bert_score`), run `poetry install --extras "local_gpu"` instead.
4. Edit the config if needed (`configs/config.yaml`).
5. Set up environment variables for the proprietary model keys if necessary (see details in the [Usage](#usage) section).
6. Run the benchmark (see details in the [Usage](#usage) section):
`poetry run python run_benchmark.py`

You can run the benchmark on a subset of the datapoints by passing the `--limit` parameter with either the number of datapoints to run or a list of IDs:

`poetry run python run_benchmark.py --limit=2`

# Dataset

Each datapoint contains a plotting task, a small CSV with the data to be plotted, and the ground truth images. 
Each task is divided into two parts:
1. **Plot description**. The main part, describing the target plot.
2. **Plot style description**. General guidelines for the styling of the plot.

The tasks can be changed dynamically using the `TaskChanger` class (see the [Usage](#usage) section).

The dataset can be loaded via [`load_dataset`](https://huggingface.co/docs/datasets/v3.1.0/en/package_reference/loading_methods#datasets.load_dataset):

```
from datasets import load_dataset
dataset = load_dataset("JetBrains-Research/PandasPlotBench", split="test")
```

# Usage

For the code generation models, you can use three options:

1. **VLLM**. Just pass the HuggingFace model name in the `model_plot_gen.names` list.
2. **OpenAI models**. Add "openai/" prefix to the OpenAI model name to select this option. In this case, you should set the `OPENAI_KEY` environment variable with a corresponding token. 
3. **TogetherAI models**. Add "together/" prefix to the TogetherAI model name to select this option. In this case, you should set the `TOGETHERAI_KEY` environment variable with a corresponding token.

For image-based scoring, we use the OpenAI GPT4-v model (the default is `gpt-4o-2024-05-13`). Thus, you have to set the `OPENAI_KEY` environment variable with a corresponding token.
You can provide the keys in the `.env` file at the root of the repository, and they will be loaded automatically.

## Basic usage
```
from plotting_benchmark.benchmark import PlottingBenchmark

benchmark = PlottingBenchmark(config_path="configs/config.yaml")

benchmark.run_benchmark()
```

### Method's arguments:

- `ids` — limits datapoints IDs to be benchmarked: _e.g._, `ids = [3, 5, 7]`
- `reuse_results` — if `True`, does not generate plots, reuses results saved in `results_filename`.
- `load_intermediate` — if `True`, does not generate plots, loads intermediate results from `current_results.jsonl`
that stores intermediate results in the case of a crash.
- `only_stats` — if `True`, does not run benchmarking, and rather just calculates the stats from `results_filename`.

### Resources

The config template and LLM instructs can be found in the `plotting_benchmark/resources` directory.


## Results

The results are saved in the `out_folder` that is set up in the config.
For each benchmarked model, the following files are saved:

- `results_{modelname}_{plottinglib}_{df_descriptor}.json` — a dataset with the results for each datapoint (plots in encoded PNG, scores, generated code).
- `all_plotsall_plots_{modelname}_{plottinglib}_{df_descriptor}.ipynb` — a notebook with all the plots of the dataset (code, figures, possible errors).
- `benchmark_stat.jsonl` — statistics for the benchmark scores. The results of each model start from new line.
 

## Custom task changer

You can experiment with the wording of tasks, _i.e._, change data description or the setup part to control plotting libraries.
To do that, create a `CustomTaskChanger` inheriting from `TaskChanger`. Here is a template for a custom task changer:

```
import pandas as pd
from plotting_benchmark.benchmark import PlottingBenchmark
from plotting_benchmark.task_changer import TaskChanger

class MyTaskChanger(TaskChanger):

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
   
    def setup_changer(self, task_text: str, df: pd.DataFrame) -> str:
        return "Use assembler language and [PLOLIB] library to draw a plot"

    def data_descr_changer(self, task_text: str, df: pd.DataFrame) -> str:
        return generate_custon_dataframe_description(task_text, df)
        
    def plot_descr_changer(self, task_text: str, df: pd.DataFrame) -> str:
        # Be carefull with that - it is the main task, describing the plot.
        return task_text

    def style_changer(self, task_text: str, df: pd.DataFrame) -> str:
        return "Draw a beautiful plot"

benchmark = PlottingBenchmark(
    config_path="configs/config.yaml", task_changer_class=MyTaskChanger
)

benchmark.run_benchmark()
```

# Important notes

1. Our approach relies on running LLM-generated code. In addition to safety, there is an issue of having the right libraries installed. The generated code could use libraries that are not installed, and this will lead to failing to build a plot. Perhaps, we should list the installed graphical libraries in the prompt.
2. `time_used_per_item` in statistics includes the waiting time in a case of a time-out.
