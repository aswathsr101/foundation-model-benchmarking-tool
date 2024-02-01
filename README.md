# Foundation Model benchmarking tool (FMBT) built using Amazon SageMaker

A key challenge with FMs is the ability to benchmark their performance in terms of inference latency, throughput and cost so as to determine which model running with what combination of the hardware and serving stack provides the best price-performance combination for a given workload.

Stated as **business problem**, the ask is “_*What is the dollar cost per transaction for a given generative AI workload that serves a given number of users while keeping the response time under a target threshold?*_”

But to really answer this question, we need to answer an **engineering question** (an optimization problem, actually) corresponding to this business problem: “*_What is the minimum number of instances N, of most cost optimal instance type T, that are needed to serve a workload W while keeping the average transaction latency under L seconds?_*”

*W: = {R transactions per-minute, average prompt token length P, average generation token length G}*

This foundation model benchmarking tool (a.k.a. `FMBT`) is a tool to answer the above engineering question and thus answer the original business question about how to get the best price performance for a given workload. Here is one of the plots generated by `FMBT` to help answer the above question (_the numbers on the y-axis, transactions per minute and latency have been removed from the image below, you can find them in the actual plot generated on running `FMBT`_).

![business question](./img/business_summary.png)

## Description

The `FMBT` is a Python based tool to benchmark performance of **any model** on **any supported instance type** (`g5`, `p4d`, `Inf2`). We deploy models on Amazon SageMaker and use the endpoint to send inference requests to and measure metrics such as inference latency, error rate, transactions per second etc. for different combinations of instance type, inference container and settings such as tensor parallelism etc. Because `FMBT` works for any model therefore it can be used not only testing _third party models_ available on SageMaker, _open-source models_ but also _proprietary models_ trained by enterprises on their own data.

`FMBT` can be run on any platform that supports Jupyter Notebooks (such as SageMaker, AWS Cloud9 and others) or containers (Amazon EC2, Amazon EKS, AWS Fargate). It is important to run this tool on an AWS platform so that internet round trip time does not get included in the end-to-end response time latency.

>In a production system you may choose to deploy models outside of SageMaker such as on EC2 or EKS but even in those scenarios the benchmarking results from this tool can be used as a guide for determining the optimal instance type and serving stack (inference containers, configuration).

The workflow for `FMBT` is as follows:

```
Create configuration file
        |
        |-----> Deploy model on SageMaker
                    |
                    |-----> Run inference against deployed endpoint(s)
                                     |
                                     |------> Run code for report creation within your s3 bucket automatically
```


1. Create a dataset of different prompt sizes and select one or more such datasets for running the tests.
    1. Currently use datasets from [LongBench](https://github.com/THUDM/LongBench) and filter out individual items from the dataset based on their size in tokens (for example, prompts less than 500 tokens, between 500 to 1000 tokens and so on and so forth).

1. Deploy **any model** that is deployable on SageMaker on **any supported instance type** (`g5`, `p4d`, `Inf2`).
    1. Models could be either available via SageMaker JumpStart (list available [here](https://sagemaker.readthedocs.io/en/stable/doc_utils/pretrainedmodels.html)) as well as models not available via JumpStart but still deployable on SageMaker through the low level boto3 (Python) SDK (Bring Your  Own Script).
    1. Model deployment is completely configurable in terms of the inference container to use, environment variable to set, `setting.properties` file to provide (for inference containers such as DJL that use it) and instance type to use.

1. Benchmark FM performance in terms of inference latency, transactions per minute and dollar cost per transaction for any FM that can be deployed on SageMaker.
    1. Tests are run for each combination of the configured concurrency levels i.e. transactions (inference requests) sent to the endpoint in parallel and dataset. For example, run multiple datasets of say prompt sizes between 3000 to 4000 tokens at concurrency levels of 1, 2, 4, 6, 8 etc. so as to test how many transactions of what token length can the endpoint handle while still maintaining an acceptable level of inference latency.

1. Generate a report that compares and contrasts the performance of the model over different test configurations and stores the reports in an Amazon S3 bucket.
    1. The report is generated in the [Markdown](https://en.wikipedia.org/wiki/Markdown) format and consists of plots, tables and text that highlight the key results and provide an overall recommendation on what is the best combination of instance type and serving stack to use for the model under stack for a dataset of interest.
    1. The report is created as an artifact of reproducible research so that anyone having access to the model, instance type and serving stack can run the code and recreate the same results and report.

1. Multiple [configuration files](https://github.com/aws-samples/jumpstart-models-benchmarking-test-harness/tree/main/configs) that can be used as reference for benchmarking new models and instance types.

## Getting started

The code is a collection of Jupyter Notebooks that are run in a sequence to benchmark a desired model on a desired set of instance types. All data that includes metrics, reports and results are stored in S3.

### Prerequisites

Follow the prerequisites below to set up your environment before running the code:

1. This code requires Python 3.11. If you are using SageMaker for running this code then select `Data Science 3.0` kernel for your SageMaker notebooks.

1. **Use S3 for Data and Results Storage** : Configure two buckets within your AWS account: 

    * ***Read bucket***: This bucket contains `tokenizer files`, `prompt template`, `source data` and `deployment scripts` directories. `FMBT` needs to have read access to this bucket. View the structure of this bucket below:
    
    ```
    S3 Bucket 
        ├── source_data/
            ├── <source-data-file-name>.json
        |
        ├── prompt_template/
            ├── prompt_template.txt
        |    
        ├── scripts/
            ├── <deployment-script-name>.py
        |
        ├── tokenizer/
            ├── tokenizer.json
            ├── config.json
    ```
     
     * View the details of bucket structure below:

        1. **Source Data Directory**: Create a `source_data` directory that stores the dataset you want to benchmark with. `FMBT` uses `Q&A` datasets from the [`LongBench dataset`](https://github.com/THUDM/LongBench). _Support for bring your own dataset will be added soon_.

            * Download the different files specified in the [LongBench dataset](https://github.com/THUDM/LongBench) into the `source_data` directory. Following is a good list to get started with:

                * `2wikimqa`
                * `hotpotqa`
                * `narrativeqa`
                * `triviaqa`
        
        1. **Prompt Template Directory**: Create a `prompt_template` directory that contains a `prompt_template.txt` file. This file contains the prompt template that your specific model supports. `FMBT` already supports the prompt template compatible with `Llama` models as shown [here](https://github.com/aws-samples/foundation-model-benchmarking-tool/blob/s3_metrics/prompt_template/prompt_template.txt)

        1. **Scripts Directory** (`Deploying models not natively available via SageMaker JumpStart`): for deploying models that are not natively available via SageMaker JumpStart i.e. anything not included in [this](https://sagemaker.readthedocs.io/en/stable/doc_utils/pretrainedmodels.html) list `FMBT` also supports a `bring your own script (BYOS)` mode. Here are the steps to use BYOS.

            1. Create a Python script to deploy your model on a SageMaker endpoint. This script needs to have a `deploy` function that [`1_deploy_model.ipynb`](./1_deploy_model.ipynb) can invoke. See [`p4d_hf_tgi.py`](./scripts/p4d_hf_tgi.py) for reference.

            1. Place your deployment script in the `scripts` directory in your ***read bucket***. If your script deploys a model directly from HuggingFace and needs to have access to a HuggingFace auth token, then create a file called `hf_token.txt` and put the auth token in that file. The [`.gitignore`](.gitgnore) file in this repo has rules to not commit the `hf_token.txt` to the repo. Today, `FMBT` provides inference scripts for:

                * [All SageMaker Jumpstart Models](https://docs.aws.amazon.com/sagemaker/latest/dg/jumpstart-foundation-models.html)
                * [Text-Generation-Inference (TGI) container supported models](https://huggingface.co/text-generation-inference)
                * [Deep Java Library DeepSpeed container supported models](https://docs.djl.ai/docs/serving/serving/docs/lmi/configurations_large_model_inference_containers.html)
                

                View inference scripts for the options above in the [SCRIPTS](https://github.com/aws-samples/foundation-model-benchmarking-tool/tree/s3_metrics/scripts) directory.

        1. **Tokenizer Directory**:  We currently use the `Llama 2 Tokenizer` for counting prompt and generation tokens. The use of the LLama2 model is governed by the Meta license. In order to download the tokenizer, visit the website and accept the License before requesting access here: [meta approval form](https://ai.meta.com/resources/models-and-libraries/llama-downloads/). In order to download the tokenizer, visit the website and accept the License before requesting access here: [meta approval form](https://ai.meta.com/resources/models-and-libraries/llama-downloads/). Once you have been approved, create the `llama2_tokenizer` directory and download the following files from [Hugging Face website](https://huggingface.co/meta-llama/Llama-2-7b/tree/main) into this directory:

            * `tokenizer.json`
            * `config.json`

            
        * You can also use your model specific tokenizer by uploading the tokenizer files from `HuggingFace` into the `tokenizer` directory. `FMBT` processes these files from the S3 bucket.

* ***Write bucket***: All data that contains metrics, prompt payloads and model endpoint information is stored in this bucket. `FMBT` requires write permissions to store the results in this bucket. View the structure of this bucket after results are generated:

```
Metrics Bucket  
    |
    ├── General Experiment Name 
    ├── data/
        ├── metrics/
        ├── prompts/
        ├── models/

```

### Steps to run

1. The `FMBT` is currently intended to run on SageMaker (or any other compute platform where Python 3.11 and JupyterLab can be installed).

1. Clone the [`FMBT`](https://github.com/aws-samples/jumpstart-models-benchmarking-test-harness.git) code repo (you would likely want to fork the repo to create your own copy).

1. Create a config file in the [configs](https://github.com/aws-samples/jumpstart-models-benchmarking-test-harness/tree/main/configs) directory.
    1. The configuration file is a YAML file containing configuration for all steps of the benchmarking process. It is recommended to create a copy of an existing config file and tweak it as necessary to create a new one for your experiment.
    1. Change the config file name in the [config_filename.txt](https://github.com/aws-samples/jumpstart-models-benchmarking-test-harness/blob/main/config_filepath.txt) to point to your config file.

1. Create the read and write buckets as mentioned above. Mention the respective directories for your read and write buckets within the `configs` files. 

1. Run the [`0_setup.ipynb`](https://github.com/aws-samples/jumpstart-models-benchmarking-test-harness/blob/main/0_setup.ipynb) to install the required [Python packages](https://github.com/aws-samples/jumpstart-models-benchmarking-test-harness/blob/main/requirements.txt).

1. Setup your custom tokenizer and datasets needed for download as per instructions in this [README](https://github.com/aws-samples/jumpstart-models-benchmarking-test-harness/tree/main?tab=readme-ov-file#solution-prerequisites).

1. Run the [`1_generate_data.ipynb`](https://github.com/aws-samples/jumpstart-models-benchmarking-test-harness/blob/main/1_generate_data.ipynb) to create the prompt payloads ready for testing. The generated prompt payloads are written in the `metrics` directory of the results bucket.

1. Run the [`2_deploy_model.ipynb`](https://github.com/aws-samples/jumpstart-models-benchmarking-test-harness/blob/main/2_deploy_model.ipynb) to deploy models on different endpoints with the desired configuration as per the configuration file.

1. Run the [`3_run_inference.ipynb`](https://github.com/aws-samples/jumpstart-models-benchmarking-test-harness/blob/main/3_run_inference.ipynb) to run inference on the deployed endpoints and collect metrics. These metrics are saved in the metrics directory (these raw metrics are not checked in back into the repo).

1. Run the [`4_model_metric_analysis.ipynb`](https://github.com/aws-samples/jumpstart-models-benchmarking-test-harness/blob/main/4_model_metric_analysis.ipynb) to create statistical summaries, plots, tables and a `final report` for the test results. The reports, metrics and plots are written to the `metrics` directory in the results bucket.

1. Run the [`5_cleanup.ipynb`](https://github.com/aws-samples/jumpstart-models-benchmarking-test-harness/blob/main/5_cleanup.ipynb) to delete the deployed endpoints.

## Results

Here is a screenshot of the `report.md` file generated by `FMBT`.
![Report](./img/results.gif)

## Pending enhancements

The following enhancements are identified as next steps for `FMBT`.

1. [**Highest priority**] Containerize `FMBT` and provide instructions for running the container on EC2.

1. Add code to determine the cost of running an entire experiment and include it in the final report. This would only include the cost of running the SageMaker endpoints based on hourly public pricing (the cost of running this code on a notebook or a EC2 is trivial in comparison and can be ignored).

1. Support for a custom token counter. Currently only the LLama tokenizer is supported but we want to allow users to bring their own token counting logic for different models.

1. Support for different payload formats that might be needed for different inference containers. Currently the HF TGI container, and DJL Deep Speed container on SageMaker both use the same format but in future other containers might need a different payload format.

1. Emit live metrics so that they can be monitored through Grafana via live dashboard.

View the [ISSUES](https://github.com/aws-samples/foundation-model-benchmarking-tool/issues) on github and add any you might think be an beneficial iteration to this benchmarking harness.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](./LICENSE) file.
